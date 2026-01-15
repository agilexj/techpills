### **What is a tombstone?**

In database or event-sourcing contexts, a **tombstone** is a special marker row that indicates a record has been **deleted**. Instead of physically removing the row immediately, the system writes a record with a flag like `IS_DELETED = true`.  
Why?

*   For **replication** or **streaming systems** (e.g., Kafka, CDC), consumers need to know that a record was deleted.
*   For **audit/history**, you want to keep track of deletions.
*   For **eventual consistency**, downstream systems can process the delete event.

So, a tombstone is basically:

```json
{
  "id": 123,
  "IS_DELETED": true,
  "other fields": null or minimal
}
```

***

### **Why should a row be expired?**

Tombstones are useful for a while, but **keeping them forever is costly**:

*   They take up storage.
*   They clutter in-memory caches.
*   They slow down queries.

So systems often define a **retention period** (e.g., 7 days, 30 days). After that:

*   The tombstone is considered **expired**.
*   It can be safely **evicted** because all consumers have had enough time to process the delete event.

This is what your code checks with:

```java
isTombstoneInRetention(kbTableRow)
```

If it returns `false`, the tombstone is **expired** and can be dropped.

***

### **Why not just delete immediately?**

Immediate deletion can break:

*   **Replication**: downstream might never see the delete.
*   **Consistency**: caches might still serve stale data.
*   **Audit**: you lose the history of deletion.

Retention gives a buffer for these processes.

***
