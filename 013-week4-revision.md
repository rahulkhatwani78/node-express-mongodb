# Week 4 Revision Notes: Advanced MongoDB & Node.js 🚀

---

## 1. MongoDB Shell (`mongosh`) & Basics

`mongosh` is the interactive JavaScript REPL for MongoDB.

- **Basic Commands**: `show dbs`, `use <db_name>`, `show collections`.
- **CRUD**:
  - **C**: `insertOne()`, `insertMany()`
  - **R**: `find()`, `findOne()`
  - **U**: `updateOne()`, `updateMany()`
  - **D**: `deleteOne()`, `deleteMany()`

---

## 2. Mongoose & MVC Architecture

**Mongoose** is an Object Data Modeling (ODM) library for Node.js, providing schema validation and powerful model abstractions.

### MVC Architecture in APIs

Separates application concerns for better maintainability:

- **Models**: Defines schemas, data types, validators, and relationships (`userSchema = new mongoose.Schema({...})`).
- **Controllers**: Contains business logic (`User.find()`, `User.create()`).
- **Routes**: Maps HTTP methods/URLs to Controller functions (`router.get('/users', getAllUsers)`).

---

## 3. MongoDB Indexing

Indexes are critical for optimizing read queries, transitioning database scans from **O(N)** (Full Collection Scan) to **O(log N)** (Index Scan).

- **The Goal**: Achieve `IXSCAN` instead of `COLLSCAN`.
- **Types of Indexes**:
  - **Single Field**: `createIndex({ email: 1 })`
  - **Compound**: Multi-field index. Order matters! `createIndex({ name: 1, age: -1 })`
  - **Unique**: Prevents duplicate values ` { unique: true }`.
  - **TTL (Time-To-Live)**: Auto-deletes documents after X seconds (good for sessions).
  - **Multikey**: Indexes arrays automatically.
  - **Text**: For searching inside string content.
- **Covered Queries**: When a query's criteria AND its projection exactly match an index, MongoDB doesn't even need to load the physical document.
- **Cardinality**: Build indexes on high cardinality fields (e.g., `email`, `id`), NOT low cardinality (e.g., `gender`, `boolean`).

---

## 4. Aggregations & Query Planner

The Aggregation Framework processes data through a multi-stage pipeline.

### Core Stages

- `$match`: Filters documents (place this early!).
- `$group`: Groups by specific identifier (`_id`), performs accumulation (`$sum`, `$avg`, `$push`, `$addToSet`).
- `$project`: Includes/excludes fields.
- `$sort`: Orders results.
- `$unwind`: Deconstructs an array, creating a document for each element.
- `$lookup`: Performs a Left Outer Join between collections.

### Array Operators & Analytics

- `$map`, `$filter`, `$reduce` allow manipulation of arrays directly without `$unwind`.
- `$facet`: Run multiple independent pipelines in parallel.
- `$bucket`: Categorize records into boundaries (e.g., age groups).

### Memory Limits

- By default, blocking stages (like `$group`, `$sort`) are restricted to **100MB RAM**. If surpassed, use `{ allowDiskUse: true }`.

### Query Planner (`.explain()`)

MongoDB optimizes queries by testing different plans and finding the fastest (the "Winning Plan"):

- Use `db.collection.find().explain("executionStats")` to verify index hits.
- **Key metrics**: `totalDocsExamined` vs returning document count. If you examine 10,000 docs to return 5, your index is missing or inefficient.

---

## 5. Transactions & ACID Properties

Starting from MongoDB 4.0, multi-document ACID transactions are natively supported on Replica Sets and Sharded Clusters.

### ACID Principles:

1. **Atomicity**: All or nothing.
2. **Consistency**: Data stays valid before and after.
3. **Isolation**: Concurrent operations don't collide.
4. **Durability**: Committed data is saved permanently.

### Usage in Node.js (Mongoose)

Transactions require passing a `session` to every query phase:

```javascript
const session = await mongoose.startSession();
session.startTransaction();
try {
  await Account.updateOne(..., { session });
  await session.commitTransaction();
} catch (error) {
  await session.abortTransaction();
} finally {
  session.endSession();
}
```

---

## 6. Scaling: Replication vs. Sharding

### Replication (High Availability)

Synchronizes data across multiple servers using a **Replica Set**.

- **Nodes**: Primary (handles Writes), Secondaries (handles Replicated data and optional Reads), Arbiter (votes, no data).
- **Read Preferences**: Determine which node to query (`primary`, `secondary`, `secondaryPreferred`, `nearest`).
- **Write Concerns**: Acknowledgment levels for safety (`w: "majority"`, `j: true`).

### Sharding (Horizontal Scaling)

Distributes data physically across entirely different multiple servers to alleviate CPU, RAM, and Disk I/O limits.

- **Components**:
  1. **Shard** (Replica Sets holding subsets of data).
  2. **Config Servers** (Store metadata and routing tables).
  3. **Mongos Router** (The entry point; directs application queries to the right shard).
- **Shard Keys**: Crucial to how data is split. Using a compound key (`tenantId` + `createdAt`) helps avoid **"Jumbo Chunks"**.
- **Balancer**: A background background process that continuously watches and migrates chunks across shards to maintain perfectly balanced data distribution without ever causing downtime.

---

## 7. Practical Task Reference

For a comprehensive, hands-on implementation of the concepts covered in Week 4 (including complex MongoDB Aggregations and Replica Set usage), please refer to the practical task link below:

👉 **[Week 4 Practical Task Repository Link](https://github.com/rahulkhatwani78/node-express-mongodb/tree/master/013-week4-practicalTask)**
