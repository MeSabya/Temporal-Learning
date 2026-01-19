## WHAT IS A SIGNAL CHANNEL?

```Go
ch := make(chan bool)
```

- In-memory
- Lost on crash
- No persistence

### In Temporal
```go
signalCh := workflow.GetSignalChannel(ctx, "payment-webhook")
```

- This is NOT a Go channel.
- What it actually is
- A view over workflow history
- Reads SignalReceived events
- Backed by durable storage
- Replay-safe

## WHAT IS A SELECTOR?

```Go
select {
case <-ch:
case <-time.After(10 * time.Second):
}
```

- Non-deterministic
- Time-based randomness
- Cannot replay

### In Temporal

```go
selector := workflow.NewSelector(ctx)
selector.Select(ctx)
```

#### What Selector really is

- A deterministic event multiplexer
- Consumes workflow history
- Chooses exactly one ready event
- Deterministic on replay

## SIGNAL vs SIGNAL CHANNEL (TEMPORAL)

They are two sides of the same thing, but used in different places.

### Signal What it is

A message sent to a running workflow.
Where it lives

***Outside the workflow:***
- HTTP handlers
- Activities
- CLIs
- Other workflows

#### Example

```go
client.SignalWorkflow(
	ctx,
	workflowID,
	runID,
	"payment-webhook",
	payload,
)
```

**Meaning: “Append a SignalReceived event to workflow history”**

### SIGNAL CHANNEL (Inside workflow)

***What it is***

- A workflow-side receiver for signals.
- it livesInside workflow code only.

#### Example

```
signalCh := workflow.GetSignalChannel(ctx, "payment-webhook")
```
**Meaning “Give me a handle to read signal events from history”**


#### HOW THEY CONNECT

```
client.SignalWorkflow()
        ↓
SignalReceived event (history)
        ↓
workflow.GetSignalChannel()
        ↓
selector.Receive()
```

👉 They meet via workflow history — not memory.
