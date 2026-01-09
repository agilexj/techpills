*Start by reading [YARN](yarn.md).*

## 1. The Container-to-TaskManager Mapping

Your assessment is **correct**.

In a Flink-on-YARN deployment, the Resource Manager (RM) allocates containers based on the job's requirements.

* **The "11th" Container:** As seen in your YARN metrics (`Running Containers: 11`), one container is dedicated to the **ApplicationMaster (AM)**. In Flink, this AM functions as the **JobManager**, which coordinates the job, handles checkpoints, and manages the TaskManagers.
* **The 10 TaskManagers:** The remaining 10 containers are your **TaskManagers** (the worker nodes).

---

## 2. Slots vs. Parallelism

With 10 TaskManagers and 8 slots each, your cluster has a total capacity of **80 Task Slots**.

However, it is vital to distinguish between **Capacity** and **Parallelism**:

* **Total Capacity:** 80 slots. This is the "real estate" available to run work.
* **Job Parallelism:** This is the number of parallel instances of a specific task.
* **The Critical Correction:** While your cluster *capacity* is 80, your *total parallelism* is not a single number. Each operator (Source, Map, Sink) has its own parallelism. In your case, you have multiple operators running at a parallelism of 80 simultaneously.

---

## 3. Total Source Parallelism & Threading

![](/pics/flink-clc-sources.png)

This is where we need to be critical of the "Total Parallelism" concept.

### Is the total parallelism 80?

**Technically, No.** Parallelism is defined per operator.

* `Source: mutations-1`: Parallelism 80.
* `Source: mutations-2`: Parallelism 80.
* `Source: transactions`: Parallelism 1.

The "Job Parallelism" is usually considered the maximum parallelism of any single operator (which is 80). If you tried to set any operator to 81, the job would fail because you only have 80 slots available.

### The Thread Count Calculation

Your calculation of **161 source threads ()** is **correct**.
Each subtask in Flink runs as a separate thread within the TaskManager's JVM. Therefore:

1. 80 threads are reading from `mutations-1`.
2. 80 threads are reading from `mutations-2`.
3. 1 thread is reading from `transactions`.

---

## 4. Understanding Slot Sharing (The "Co-location" Secret)

You asked: *"Is it likely that a single task slot can run one or more source subtask?"*

**Yes, and this is a core Flink optimization called "Slot Sharing."**

By default, Flink allows subtasks from **different** operators of the same job to share a single slot. This prevents slots from sitting idle while waiting for data from a previous stage.

In your specific job:

* **Slot 1** likely contains: 1 subtask of `mutations-1`, 1 subtask of `mutations-2`, and 1 subtask of the `transactions` source.
* **Slots 2 through 80** likely contain: 1 subtask of `mutations-1` and 1 subtask of `mutations-2`.

**Why this is efficient:**
If Flink *didn't* allow slot sharing, you would need  slots to run this job. Because of slot sharing, you can squeeze all 161 source threads (plus all downstream transformation threads) into just **80 slots**.

---

## 5. Knowledge Base Summary Table

| Concept | YARN Perspective | Flink Perspective |
| --- | --- | --- |
| **Unit of Resource** | **Container**: A physical slice of RAM/CPU on a node. | **Task Slot**: A logical slice of a TaskManager's memory. |
| **Management** | **ResourceManager**: Allocates containers to apps. | **JobManager**: Allocates subtasks to slots. |
| **Scaling** | Add more **Nodes** to the YARN cluster. | Increase **Parallelism** (if slots are available). |
| **Multi-threading** | Usually 1 process per container. | Multiple **Threads** (subtasks) sharing one slot. |

## 6. Thread Count Calculation (Logical View)

You asked about the total number of threads. To find this, we must look at every operator in your DAG (Directed Acyclic Graph). In Flink, each subtask of every operator runs as a **separate thread**.

![](/pics/flink-clc-topology.png)

### Summing the Operator Subtasks

According to your topology screenshot:

* **Source: transactions**: Parallelism 1 = **1 thread**
* **Source: mutations-1**: Parallelism 80 = **80 threads**
* **Source: mutations-2**: Parallelism 80 = **80 threads**
* **Source: Custom File Source**: Parallelism 1 = **1 thread**
* **DeduplicateKeyedTransactions**: Parallelism 80 = **80 threads**
* **mutations-funnel**: Parallelism 80 = **80 threads**
* **KnowledgeBaseUpdatesFromFile**: Parallelism 80 = **80 threads**
* **DeltaUpdates**: Parallelism 80 = **80 threads**
* **KnowledgeBaseBuilder -> Sink**: Parallelism 80 = **80 threads**

**Total Logical Threads: 562 threads**

### The "Operator Chaining" Optimization

Flink will attempt to "chain" operators that have the same parallelism and a `FORWARD` connection to reduce thread-to-thread communication overhead.

* **Likely Chain:** `mutations-1`  `mutations-funnel` (both P=80, FORWARD connection). These will likely run in the **same thread**.
* **Total "Task" Threads:** If the funnel is chained, your total thread count drops by 80, resulting in roughly **482 active task threads**.

---

## 7. Resource Allocation: Threads per JVM/Node

Now let's bridge this to your YARN dashboard.

### The Math:

* **TaskManagers (Containers):** 10.
* **Total Slots:** 80 (8 slots per TaskManager).
* **Max Parallelism:** 80.

### Threads per TaskManager (JVM):

Because of **Slot Sharing**, Flink co-locates subtasks from different stages into the same slot to maximize resource use.

* **Total Threads per TM:** .
* **Per Slot:** Each of your 8 task slots is hosting roughly **6 to 7 concurrent threads** (one from each "stage" of your pipeline).

---

## 8. Critical Reasoning: The YARN/Flink Mismatch

Here is where you should be critical of your current setup. According to your YARN "About the Cluster" and "Node Metrics" screenshots:

1. **CPU Starvation:** Your YARN dashboard shows **11 Allocated vCores** for **11 Containers**. This means each TaskManager (running ~48 threads) is only allocated **1 vCore**.
* **The Problem:** You are asking 1 physical CPU core to context-switch between 48 active Flink threads. While your "Physical VCores Used %" is currently low (0-1%), if your data volume spikes, this single vCore will become a massive bottleneck, and your "BackPressure" will skyrocket.


2. **Memory Oversubscription:** Each node has **20GB** allocated to YARN. For 8 slots, that is **2.5GB per slot**. This is quite generous for most streaming tasks unless you have very large window states.
3. **Slot Distribution:**
* **Slot 0 (in TaskManager 1):** This is the "heavy" slot. It likely contains the `transactions` source (P1), the `Custom File Source` (P1), **plus** one subtask from every other P80 operator.
* **Slots 1-79:** These only contain the P80 operator subtasks.



---

## 9. Slot Sharing Visualized

The following diagram illustrates how your specific DAG "squeezes" into your 80 slots. Even though you have many operators, they overlap vertically in the same slots.

### Slot Sharing Logic for your Case:

```text
[ TaskManager 1 ] (1 vCore Allocated by YARN)
|-- Slot 1: [Trans. Source (P1)] + [Mut.-1 subtask] + [Deduplicate subtask] + [Sink subtask] ...
|-- Slot 2: [Mut.-1 subtask] + [Deduplicate subtask] + [Sink subtask] ...
|-- ...
|-- Slot 8: [Mut.-1 subtask] + [Deduplicate subtask] + [Sink subtask] ...

```

---

## 10. Summary for Knowledge Base

| Entity | Count | Learning Note |
| --- | --- | --- |
| **YARN Containers** | 11 | 1 JobManager (AM) + 10 TaskManagers. |
| **YARN vCores** | 1 per TM | **Critical Risk**: Low CPU allocation for high thread count. |
| **Flink Task Slots** | 80 | Total capacity. Limits the maximum parallelism of any single stage. |
| **Total Task Threads** | ~482 | Sum of subtasks after operator chaining optimizations. |
| **Threads per JVM** | ~48 | Total task threads divided by the 10 TaskManager processes. |

---

## 11. What the "9" Actually Means: Vertices

In Flink's terminology, a **Vertex** is a box in your topology diagram. If you look at your 8th screenshot (the full DAG), you can count exactly 9 blue boxes:

1. **Source: transactions**  `TransactionToKeyedTransactions`
2. **Source: mutations-1**
3. **Source: mutations-2**
4. **DeduplicateKeyedTransactions**
5. **mutations-funnel**
6. **Source: Custom File Source**
7. **DeltaUpdates**
8. **KnowledgeBaseUpdatesFromFile**
9. **KnowledgeBaseBuilder**  `DocumentCreation`  `DocumentSink: Writer`

Because the number is **9**, it confirms that every single stage of your pipeline has successfully deployed and is currently active. If one of your sources failed, that number would drop to 8.
