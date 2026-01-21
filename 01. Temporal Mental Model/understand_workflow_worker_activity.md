# Temporal Core Concepts (Quick Notes)

## Workflow

- Durable orchestration logic
- Runs inside Temporal, not on workers
- State is persisted as event history
- Can run for seconds to years
- Must be deterministic
- Cannot do blocking IO, sleep, network calls

Think:
Workflow = Business state machine

## Activity

- Side-effecting work
- Runs on workers

Can do:
- HTTP calls
- DB access
- File IO
- CPU heavy work
- Can fail, retry, timeout independently
- Result is recorded in workflow history

Think:
- Activity = Imperative task / function 

## Worker

- Stateless process that executes tasks
- Polls task queues

Executes:

- Workflow tasks (decision making)
- Activity tasks (actual work)
- Can be killed anytime safely
- Horizontally scalable

Think:
- Worker = Executor / hands

## Task Queues

- Logical routing mechanism
- Workflows and activities are placed into queues

Workers poll queues they are configured for

Think:
Task Queue = Inbox

## Workflow Task vs Activity Task

### Workflow Task

- Executes workflow code
- Replays history
- Produces decisions (schedule activity, wait, complete)
- Lightweight, CPU cheap

### Activity Task

- Executes real work
- Can block, retry, timeout
- Heavy, slow, external

## Can you give me a code which will show workflow only workers, activity only worker?

What YOU have today (Mixed Worker)

This is essentially what your code is doing 👇

```go
w := worker.New(c, "MAIN_QUEUE", worker.Options{})

// workflows
w.RegisterWorkflow(OrderWorkflow)

// activities
w.RegisterActivity(CreateOrder)
w.RegisterActivity(ChargePayment)

w.Run(worker.InterruptCh())
```

Meaning

- Same worker process
- Same task queue

Executes:

- workflow tasks
- activity tasks
- ✔ Works
- ❌ Poor scaling & isolation in production

## 2️⃣ Workflow-ONLY Worker (Production Pattern) : 

- Polls workflow tasks only
- If an activity task lands here → ignored
  
worker/workflow/main.go

```go
package main

import (
	"log"

	"go.temporal.io/sdk/client"
	"go.temporal.io/sdk/worker"
)

func main() {
	c, err := client.Dial(client.Options{})
	if err != nil {
		log.Fatal(err)
	}
	defer c.Close()

	w := worker.New(c, "ORDER_WORKFLOW_Q", worker.Options{
		DisableActivityWorker: true, // 🔑 KEY LINE
	})

	w.RegisterWorkflow(OrderWorkflow)

	log.Println("Workflow worker started")
	w.Run(worker.InterruptCh())
}
```

## Activity-ONLY Worker
worker/activity/main.go

```golang
package main

import (
	"log"

	"go.temporal.io/sdk/client"
	"go.temporal.io/sdk/worker"
)

func main() {
	c, err := client.Dial(client.Options{})
	if err != nil {
		log.Fatal(err)
	}
	defer c.Close()

	w := worker.New(c, "ORDER_ACTIVITY_Q", worker.Options{
		DisableWorkflowWorker: true, // 🔑 KEY LINE
	})

	w.RegisterActivity(CreateOrder)
	w.RegisterActivity(ChargePayment)

	log.Println("Activity worker started")
	w.Run(worker.InterruptCh())
}
```
## Workflow That Uses Activity Queue

```go
func OrderWorkflow(ctx workflow.Context, orderID string) error {
	ctx = workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
		TaskQueue: "ORDER_ACTIVITY_Q", // 🔑 IMPORTANT
		StartToCloseTimeout: time.Minute,
	})

	err := workflow.ExecuteActivity(ctx, CreateOrder, orderID).Get(ctx, nil)
	if err != nil {
		return err
	}

	return workflow.ExecuteActivity(ctx, ChargePayment, orderID).Get(ctx, nil)
}
```
## Kubernetes Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workflow-worker
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: worker
        image: order-workflow-worker

```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: activity-worker
spec:
  replicas: 20
  template:
    spec:
      containers:
      - name: worker
        image: order-activity-worker
```


# How workflow and workers are deployed in prod 
## Think in three independent planes:

```
┌──────────────────────┐
│   Temporal Server    │  (Control Plane)
└──────────────────────┘
          ▲
          │
┌─────────┴─────────┐
│   Task Queues     │  (Routing Plane)
└─────────┬─────────┘
          │
┌─────────┴─────────┐
│   Workers         │  (Execution Plane)
└───────────────────┘
```

### Temporal Server (Managed or Self-Hosted)

#### Deployed as:

- Temporal Cloud (most companies)
- OR Self-hosted Temporal on Kubernetes

#### Responsibilities:

- Stores workflow state
- Stores event history
- Schedules timers
- Manages retries
- Owns task queues
- 🚫 Never runs your business logic

### Workers (Your Code)

#### Deployed as:

- Kubernetes Deployments
- ECS Services
- VMs
- Bare metal

#### Workers are:

- Stateless Go services
- Horizontally scalable
- Safe to kill anytime

### Workflows & Activities (Inside Workers)

- Not deployed separately ❌

They are:

- Compiled into the worker binary

Loaded at startup via:

```go
worker.RegisterWorkflow(...)
worker.RegisterActivity(...)
```

