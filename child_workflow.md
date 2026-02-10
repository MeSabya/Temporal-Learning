## What is a Child Workflow?

A child workflow is a workflow started by another workflow (the parent), with:

- Its own execution history
- Its own retries and failures
- Its own waiting, signals, timers
- A strong lifecycle relationship with the parent
- Think of a child workflow as a durable sub-process, not a function call.

### Key Characteristics

- Started using ExecuteChildWorkflow
- Runs asynchronously by default
- Parent may wait or not wait
- Fully replay-safe
- Can run for minutes, hours, days
- Visible separately in Temporal UI
- Can be cancelled when parent closes (configurable)

```
| Aspect            | Activity           | Child Workflow        |
| ----------------- | ------------------ | --------------------- |
| Purpose           | Do a single action | Orchestrate a process |
| Duration          | Short              | Long                  |
| Waiting           | ❌ No               | ✅ Yes                 |
| Signals           | ❌ No               | ✅ Yes                 |
| Timers / Sleep    | ❌ No               | ✅ Yes                 |
| Internal state    | ❌ No               | ✅ Yes                 |
| Retry with memory | ❌ No               | ✅ Yes                 |
| Observability     | Low                | High                  |

```

### When do you NEED a Child Workflow?

You need a child workflow when a step is not just an action, but a process.

Use a child workflow if any one of these is true:

#### 1️⃣ The step has multiple phases

Example:

- Patch something
- Wait for reconciliation
- Verify outcome
- Retry if needed

➡️ Too complex for an activity.

#### 2️⃣ The step needs to wait (minutes or longer)

Example:

- Waiting for operator / controller
- Polling for external system state
- Human approval

➡️ Waiting belongs in workflows, not activities.

#### 3️⃣ The step has its own lifecycle

If the step can:

- Start
- Progress
- Fail
- Retry
- Be observed independently

➡️ That step deserves a child workflow.

