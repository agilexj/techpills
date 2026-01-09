# Fault Tolerant State Using Checkpoints

## Redistribute State Across Parallel Instances

### The state is partitioned exactly like the stream

State is split and distributed across the cluster in the same way that the data stream is partitioned by key.

That means:

* If an event with key `"user42"` always goes to Task 3
* Then the state belonging to `"user42"` also lives on Task 3

![](./pics/state_partitioning.svg)

---

### You can only use keyed state after `keyBy()`

Keyed state only works on keyed streams, meaning you must first call `keyBy(...)`.

This ensures that:

* Every event has an associated key
* You only access the state for the current event’s key

For example:
If the current event has key `"A"`, then you can only read/write state belonging to key `"A"`.

---

### Local state access = fast & consistent

Because the event and its state are always colocated:

* No distributed transactions needed
* State updates are local
* Consistency is maintained efficiently

---

# State Persistence in Flink

Flink ensures fault tolerance through **checkpoints** that capture both:

* The positions of input streams
* The state of all operators

If a failure occurs, Flink restores the most recent checkpoint, then replays records from that point to maintain **exactly-once** consistency.

Checkpoints are periodic, and the interval balances:

* **Overhead during execution**, and
* **Recovery time** (less replay if frequent)

Snapshots are lightweight when state is small and are typically stored in a distributed filesystem.

On failure, Flink:

1. Stops the job
2. Restores the latest checkpoint
3. Resets input streams
4. Guarantees that no records are applied twice

---

# Barriers in Flink Checkpointing

![](./pics/stream_barriers.svg)

Flink uses **stream barriers** to create consistent distributed snapshots.

* Barriers flow *in order* with the stream
* They separate data belonging to different snapshots
* They are lightweight and do not pause the stream
* Multiple snapshots can be in progress at once

---

## Behavior at Sources

![](./pics/checkpointing.svg)

Each barrier marks the exact position in the source stream (e.g., Kafka offset) for that snapshot.

As barriers move downstream:

* Operators forward a barrier only after receiving it from **all inputs**
* Sinks acknowledge the snapshot once they receive all barriers
* A snapshot is complete when **all sinks** acknowledge it
* Flink will never request data before the barrier position again

---

## Barrier Alignment (Multi-input Operators)

![](./pics/stream_aligning.svg)

For operators with multiple inputs, Flink aligns barriers:

* Upon receiving a barrier on one input, the operator **pauses that input**
* It waits for the barrier on other inputs
* Once all inputs have the barrier:

  * It forwards the barrier
  * Takes a snapshot of its state
  * Resumes normal processing

Alignment is needed for:

* Joins
* Operators after shuffles
* Any multi-input operator

---

# Recovery

On failure:

1. Flink selects the latest completed checkpoint **k**
2. Re-deploys the entire dataflow
3. Restores each operator’s state from checkpoint **k**
4. Sets sources to position **Sk** (e.g., Kafka offset Sk)

---

# Unaligned Checkpointing

![](./pics/stream_unaligning.svg)

Unaligned checkpointing allows checkpoints to **overtake** in-flight data by treating in-flight data as part of operator state.

Key ideas:

* Barriers are still inserted at sources
* Operators process data continuously without alignment delays

Operator behavior on unaligned barriers:

* Reacts to the *first* barrier in input buffers
* Immediately forwards the barrier to outputs
* Marks overtaken records for asynchronous storage
* Creates a snapshot of its own state

This minimizes pause time and ensures barriers reach sinks quickly.

---

# Savepoints

Savepoints are similar to checkpoints but:

* **Triggered manually by the user**
* **Do not expire** when new checkpoints occur

They are typically used for upgrades, migrations, or controlled restarts.

---

# Exactly Once vs. At Least Once in Flink

Checkpoint alignment ensures **exactly-once** guarantees but may add small latency.

For ultra-low latency applications, alignment can be **disabled**.

### When alignment is disabled:

* Operators continue processing all inputs even after receiving some barriers
* They may process data from snapshot *n+1* before snapshot *n* is taken
* After recovery, these records appear **twice**:

  * Included in the snapshot of *n*
  * Replayed from the stream
* → This results in **at-least-once** processing

### Important Note

Alignment is only required for operators with **multiple inputs** (joins, shuffles).
Purely parallel pipelines (`map`, `flatMap`, `filter`, etc.) still achieve **exactly-once** behavior even in at-least-once mode.

