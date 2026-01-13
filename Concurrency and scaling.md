## Temporal Worker Concurrency Controls — Summary

Temporal Workers aggressively poll for tasks. Without limits, a single Worker can overload itself, 
causing CPU spikes, OOMKills, retries, and cascading failures. Temporal provides three complementary mechanisms to control concurrency.

### max_concurrent_activities

(Static safety cap)

- Hard, static upper bound on how many Activities a Worker can execute concurrently.
- Configured at Worker startup.
- Does not consider CPU or memory usage.
- Prevents runaway concurrency due to bugs or misconfiguration.

#### Limitation
- Too high → OOM risk
- Too low → underutilized worker

### ResourceBasedSlotSupplier

(Runtime concurrency enforcer)

- Controls whether a new Activity can start by granting or denying a “slot”.
- Enforces the current allowed concurrency.
- Blocks activity execution when no slots are available.
- Does not decide how many slots exist — only enforces availability.

Role

- Low-level concurrency gate inside a Worker process.
- Prevents immediate overload.

### 3️⃣ ResourceBasedTuner

(Adaptive controller / brain)

- Continuously observes Worker CPU and memory usage.
- Dynamically increases or decreases available slots to keep resource usage near configured targets.
- Adjusts concurrency at runtime based on real conditions.

Role

- Adaptive feedback controller.
- Maximizes throughput while protecting Worker stability.

#### How they work together

Effective concurrency =
  min(
    max_concurrent_activities,
    slots_allowed_by_resource_based_tuner
  )


max_concurrent_activities = hard ceiling

ResourceBasedTuner decides how many slots are safe

ResourceBasedSlotSupplier enforces those slots

### What problem this solves

- Prevents Worker OOMKills
- Prevents CPU thrashing
- Avoids retry storms

## If we are running temporal in k8s cluster .. still we need the tuners and slotsuppliers ?

- Kubernetes limits how much a pod can use
- Temporal resource-based tuning decides how much work the worker should take.
- K8s will kill your pod when limits are exceeded.
- Temporal tuning prevents the pod from reaching that point.

### Concrete failure without tuners (even on K8s)

Setup
```yaml
resources:
  limits:
    cpu: "2"
    memory: 2Gi
```

Temporal Worker:

- Polls aggressively
- Starts 20 activities
- Each uses ~200MB
- Result : 20 × 200MB = 4GB → OOMKill

K8s action:

- Pod killed
- Restarted
- Temporal effect:
    - Activities retried
    - More load
    - More pods die
    - Retry storm
Adapts to mixed workloads (CPU-heavy, memory-heavy activities)

Keeps Workers stable without manual tuning
