- Temporal server does NOT run your activity code.
- Activity code runs inside your worker process

Temporal server can:

  - schedule activities
  - mark them as cancellation requested
  - time them out

- But the server cannot interrupt your Go function like a thread interrupt
- So the server has no way to “stop” activity code directly.


Heartbeat is the only communication channel from:

```
Activity → Temporal server → back to Activity
```
When your activity calls:

- activity.RecordHeartbeat(ctx)
Temporal server replies with:

- “You’re still good”

OR

- “You’ve been canceled”

At that moment:

- The activity’s ctx.Done() is closed
- Your code can exit early


## What happens with NO heartbeat

Let’s replay your exact scenario carefully.

Configuration

```
StartToCloseTimeout = 2m
HeartbeatTimeout    = not set
WaitForCancellation = true
```

Timeline

```
t=0s   Activity starts
t=1s   Workflow calls cancelHandler()
t=1s   Temporal server marks activity as "cancel requested"
t=1s–120s Activity keeps running (no heartbeat → no feedback)
t=120s StartToCloseTimeout fires
```

Consequences

- Activity wasted 119 seconds of CPU / I/O
- Worker threads blocked

## Why StartToCloseTimeout cannot help here

Because:

- StartToCloseTimeout is checked only after time passes
- It does not notify the activity while it is running
- It does not poll the worker
- It does not cancel execution

It’s like a judge saying:

- “After 2 minutes, I’ll declare this illegal.”
- But the runner keeps running anyway.

### Same scenario WITH heartbeat
Timeline

```
t=0s   Activity starts
t=1s   Workflow cancels
t=8s   Activity sends heartbeat
t=8s   Server replies: "You're canceled"
t=8s   ctx.Done() is closed
t=8s   Activity exits
```

## Suppose we have to batch process 10 activities or a long running activity ...Then we need to use heartbeat 

### When Do You Need Heartbeats?

You need heartbeat when:

- ✅ Activity runs long
- ✅ Activity processes batch items
- ✅ Activity calls external systems
- ✅ Partial progress matters
- ✅ Idempotency is expensive

### When Heartbeat IS Useful (Even for "Single" Activity)

Even if it's just one activity, heartbeat helps when:

#### Long-running task

Example:

```
GenerateLargeReport()  // takes 20 minutes
```

- If worker crashes at minute 18:
- Without heartbeat → restart from 0
- With heartbeat → resume from last checkpoint

#### Crash detection (VERY important)

Heartbeat isn't only for resuming.
It also detects:
- Worker crash
- Deadlock
- Infinite loop
- Network freeze

Because you set:

- HeartbeatTimeout: 10 * time.Second
- If heartbeat not received → Temporal retries immediately.
- Without heartbeat:
- Temporal waits full StartToCloseTimeout.

So not only batch activities even a single long activity benefits with heart-beats.


👉 internal/activity.go
```go
package internal

import (
	"context"
	"fmt"
	"time"

	"go.temporal.io/sdk/activity"
	"go.temporal.io/sdk/temporal"
)

func LongRunningActivity(ctx context.Context, start, total int) error {
	logger := activity.GetLogger(ctx)

	i := start

	// Resume from heartbeat
	if activity.HasHeartbeatDetails(ctx) {
		var lastCompleted int
		if err := activity.GetHeartbeatDetails(ctx, &lastCompleted); err == nil {
			i = lastCompleted + 1
			logger.Info("Resuming from heartbeat", "lastCompleted", lastCompleted)
		}
	}

	processedThisAttempt := 0

	for ; i < total; i++ {
		fmt.Println("Processing item:", i)

		time.Sleep(1 * time.Second)

		// Save progress
		activity.RecordHeartbeat(ctx, i)

		processedThisAttempt++

		// Fail after 3 items (simulate crash)
		if processedThisAttempt == 3 && i < total-1 {
			fmt.Println("Simulated failure...")
			return temporal.NewApplicationError("temporary failure", "Retryable")
		}
	}

	fmt.Println("Activity completed successfully")
	return nil
}
```

👉 internal/workflow.go 

```go
package internal

import (
	"time"

	"go.temporal.io/sdk/temporal"
	"go.temporal.io/sdk/workflow"
)

func HeartbeatWorkflow(ctx workflow.Context) error {
	ao := workflow.ActivityOptions{
		StartToCloseTimeout: 2 * time.Minute,
		HeartbeatTimeout:    5 * time.Second,
		RetryPolicy: &temporal.RetryPolicy{
			InitialInterval:    time.Second,
			BackoffCoefficient: 2.0,
			MaximumAttempts:    4,
		},
	}

	ctx = workflow.WithActivityOptions(ctx, ao)

	return workflow.ExecuteActivity(
		ctx,
		LongRunningActivity,
		0,   // start index
		10,  // total items
	).Get(ctx, nil)
}

```
