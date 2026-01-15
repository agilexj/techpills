This technical documentation describes the architecture and operation of the **Postgres Document Sink**, a custom Flink Sink (V2) designed to synchronize complex financial document updates from an Avro-based stream into a PostgreSQL database.

---

## 1. Overview

The `PostgresDocumentSink` is responsible for translating a single "Master Record" (containing multiple updates for various entities) into structured SQL operations. It uses a **Target Table Descriptor** pattern to maintain a clean separation between the database schema logic and the Flink execution logic.

---

## 2. Core Components

### A. The Target Table Descriptor

Defined in the `PostgresDocumentSinkFactory`, these descriptors act as the mapping layer. Each descriptor specifies:

* **Field Prefix:** The key used to extract specific data lists from the master record (e.g., `borrowerUpserts`).
* **SQL Templates:** Raw SQL for `DELETE` and `UPSERT` (Insert on Conflict), with dynamic schema injection (`{{targetSchema}}`).
* **Statement Setters:** Lambda functions that map Avro `GenericRecord` fields to JDBC `PreparedStatement` parameters.

### B. The Sink Writer (`PostgresSinkWriter`)

The writer manages the lifecycle of the database connection and the execution of statements. It is designed for **high throughput** using JDBC batching and **strict consistency** through manual transaction management.

---

## 3. Data Processing Lifecycle

When a message enters the sink, the following sequence occurs within the `write()` method:

| Phase | Action | Detail |
| --- | --- | --- |
| **Extraction** | **De-multiplexing** | The writer extracts lists of "Upserts" and "Deletes" for every registered table from the master record. |
| **Batching** | **JDBC Preparation** | It iterates through the tables. For each row in the lists, it applies the `StatementSetter` and calls `ps.addBatch()`. |
| **Execution** | **Batch Flush** | It calls `ps.executeBatch()` for each table, pushing the prepared commands to the database buffer. |
| **Commit** | **Atomic Finalization** | Once all tables are processed, `connection.commit()` is called to persist all changes. |

---

## 4. Key Implementation Features

### Bitemporality & Auditing

The sink enforces bitemporal data integrity by capturing:

* **`user_valid_from_ts`:** The effective time in the business domain.
* **`db_valid_from_ts`:** The technical timestamp of the database entry.
* **`audit_record`:** A `jsonb` field containing metadata about the change source.

### Transactionality: "One Message = One Transaction"

The sink operates on an atomic basis per Flink record.

* **Atomicity:** If any table update fails within a message (e.g., a constraint violation on a Guarantee), the entire message is rolled back using `connection.rollback()`.
* **Error Handling:** Depending on configuration (`isDiscardMessagesIfSinkFails`), the sink will either drop the message or throw an exception to trigger a Flink task restart/retry.

### JSONB Document Storage

While primary keys and frequently queried fields are stored in standard columns, the full state of the object is stored in a `document_st` column using the **PostgreSQL JSONB** format. This provides a balance between relational performance and schema flexibility.

---

## 5. Initialization and Configuration

The sink is instantiated using the **Lombok Builder Pattern**. This approach is preferred over standard constructors to ensure:

1. **Readability:** Explicitly named parameters for `context` and `descriptors`.
2. **Maintainability:** Avoiding "Constructor Hell" when new configuration fields are added.
3. **Safety:** Ensuring `@NonNull` fields are validated during the `.build()` phase.

**Example Usage:**

```java
PostgresDocumentSink.builder()
    .context(sinkCtx)
    .targetTableDescriptors(PostgresDocumentSinkFactory.buildTargetTableDescriptors("my_schema"))
    .build();

```

---
