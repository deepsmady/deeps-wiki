# SQL Server Internals Overview

## 1. SQL Database Cache

### What is a Database Cache?
A **database cache** stores frequently accessed data in memory (RAM) to reduce disk I/O and speed up queries.

### How It Works
1. When a query requests data:
   - The database engine checks its **buffer pool** (RAM).
   - If the needed data pages are there → serve from memory (fast).
   - If not → load those pages from disk into the buffer pool, then serve.
2. Future queries on the same data can be answered directly from memory.

### Buffer Pool
- A **buffer pool** is a memory region managed by the database engine.
- It holds **data pages** (fixed-size chunks, usually 8 KB) copied from disk.
- Data modifications are done **in memory** first:
  - The page is marked as **dirty**.
  - The change is logged in the **transaction log** (LDF).
  - Later, a **checkpoint process** writes dirty pages back to disk.

### Cache Update Behavior
- Updates happen in **two phases**:
  1. **Immediate log write** → Transaction log (LDF) records the change.
  2. **Deferred data write** → Modified pages stay in buffer pool until:
     - A checkpoint flushes them.
     - Memory pressure forces eviction.

---

## 2. SQL Server Database File Structure

### File Types
| File Type | Description                                      | Count per DB |
|-----------|--------------------------------------------------|--------------|
| **MDF**   | Master Data File: main file with schema & data   | 1            |
| **NDF**   | Secondary Data Files: optional extra data files  | 0..N         |
| **LDF**   | Log Data File: stores transaction logs           | 1..N         |

#### MDF (Primary Data File)
- Stores tables, indexes, and database schema.
- Each database has exactly **one MDF**.

#### NDF (Secondary Data Files)
- Optional, used to spread data across multiple files/disks.
- Helps with large database scaling.

#### LDF (Transaction Log File)
- Records **all changes** (insert, update, delete).
- Enables crash recovery and point-in-time restores.
- Updates must be written to LDF **before** the user’s transaction is considered committed.

---

## 3. Data Flow: Buffer Pool → Disk

1. Query updates data:
   - Data page modified **in buffer pool**.
   - Log record written to LDF.
2. Data page remains in RAM:
   - Marked as **dirty**.
3. At a checkpoint:
   - Dirty pages written to MDF/NDF (persistent storage).
   - Transaction log ensures durability until the page is written.

---

## 4. SQL Server Backups

### What Does a Backup Copy?
- A **full database backup**:
  - Copies all **data pages** from MDF/NDF files.
  - Includes enough **log records (from LDF)** to make the backup transactionally consistent.
- It **does not copy the buffer pool** (RAM content).
  - Any dirty pages in RAM are first flushed to disk (checkpoint) before backup reads them.
- Backup result is a **.bak file**, which is a logical snapshot, not a raw file copy.

### Backup Types
| Type           | What It Contains                            |
|----------------|--------------------------------------------|
| Full Backup    | All data pages + required log info          |
| Differential   | Only pages changed since last full backup   |
| Log Backup     | Only transaction log records                |

---

## 5. Key Takeaways

- **Cache = Buffer Pool** → Holds data pages in RAM.
- **Dirty pages** are not immediately written to MDF.
- **LDF guarantees durability**: commits are logged before MDF writes.
- **Backups are logical**:
  - They capture data from MDF/NDF + log records.
  - They do not dump raw files or buffer pool content.
