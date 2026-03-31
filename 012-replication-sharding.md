# Replication and Sharding in MongoDB

As your application grows, a single database server becomes insufficient to handle the load or provide the necessary redundancy. MongoDB provides two primary mechanisms to scale and ensure high availability: **Replication** and **Sharding**.

---

## 1. Replication (High Availability)

Replication is the process of synchronizing data across multiple servers. In MongoDB, this is achieved through a **Replica Set**.

### What is a Replica Set?

A replica set is a group of `mongod` instances that maintain the same data set. It provides redundancy and high availability.

- **Primary Node**: The only node that accepts **write** operations. There is strictly one primary active at a time. It records all changes to its data sets in its oplog (operation log).
- **Secondary Nodes**: These nodes replicate the primary's oplog and apply the operations to their data sets such that they mirror the primary. They can optionally handle **read** operations.
- **Arbiter** (Optional): A lightweight node that does not hold data but participates in elections to break ties and elect a new primary if the current primary goes down.

### Why Use Replication?

- **High Availability**: If the primary node fails, an automatic election determines a new primary within seconds.
- **Data Redundancy**: Keeps data safe from hardware failure.
- **Read Scaling**: You can distribute read-heavy workloads to secondary nodes.

---

## 2. Sharding (Horizontal Scaling)

While replication ensures availability, it does not increase the overall storage capacity or write throughput of the cluster. **Sharding** is the method for distributing data across multiple machines.

### Key Components of a Sharded Cluster:

1. **Shard**: A replica set that contains a subset of the cluster's data.
2. **Mongos (Query Router)**: Acts as a routing service for client applications. It receives queries and routes them to the appropriate shard.
3. **Config Servers**: A replica set that stores the metadata and configuration settings for the cluster (which data lives on which shard).

### The Shard Key

To distribute documents, MongoDB divides them using a **Shard Key**—a field or fields that exist in every document in the collection. Choosing a good shard key (e.g., hashed over monotonically increasing) is critical for evenly distributing data and avoiding "hotspots."

### Why Use Sharding?

- **Horizontal Scaling**: Adding more servers to handle massive datasets and high write loads that a single machine cannot handle.
- **Storage Constraints**: Distributing TBs or PBs of data across multiple hard drives.

---

## Replication vs. Sharding

| Feature               | Replication                              | Sharding                                               |
| :-------------------- | :--------------------------------------- | :----------------------------------------------------- |
| **Primary Goal**      | Data Redundancy and High Availability    | Horizontal Scaling (Storage and Write throughput)      |
| **Data Distribution** | Every node holds a full copy of the data | Data is partitioned; each shard holds part of the data |
| **Writes**            | Handled by a single Primary node         | Distributed across multiple shards                     |
| **Complexity**        | Easy to set up and manage                | Complex infrastructure (Mongos, Config Servers)        |

_Note: In production environments, every Shard in a sharded cluster is deployed as a Replica Set to ensure high availability._

---

## Practical Implementation in Node.js (Mongoose)

### 1. Connecting to a Replica Set

When connecting to a Replica Set, you typically provide the connection string with all the nodes, and append `replicaSet=[replicaSetName]`.

```javascript
const mongoose = require("mongoose");

// The connection string includes multiple hosts and the replica set name
const uri =
  "mongodb://node1:27017,node2:27017,node3:27017/myDatabase?replicaSet=rs0";

mongoose
  .connect(uri, {
    useNewUrlParser: true,
    useUnifiedTopology: true,
  })
  .then(() => {
    console.log("Connected to Replica Set successfully");
  })
  .catch((err) => {
    console.error("Connection error:", err);
  });
```

### 2. Read Preferences

By default, all read operations go to the Primary node (`primary`). You can change this to offload read operations to Secondary nodes.

- `primary`: Default. Reads from the primary.
- `secondary`: Reads only from secondary members.
- `primaryPreferred`: Reads from the primary, but switches to secondary if primary is unavailable.
- `secondaryPreferred`: Reads from secondary, but switches to primary if no secondaries are available.
- `nearest`: Reads from the member with the lowest network latency.

**Using Read Preferences in Mongoose:**

```javascript
// At the schema level
const userSchema = new mongoose.Schema(
  { name: String },
  {
    read: "secondaryPreferred",
  },
);

// For a specific query
const users = await User.find({ status: "active" }).read("secondary");
```

### 3. Write Concerns (Data Safety)

Write Concern dictates the level of acknowledgment requested from MongoDB for write operations.

- `w: 1` (Default): Acknowledges the write on the Primary node only. Fast, but less safe if the primary crashes before replicating.
- `w: "majority"`: Acknowledges the write only after it has been replicated to a majority of the replica set members. Slower, but highly durable.
- `j: true`: Acknowledges the write only after it has been written to the on-disk journal.

**Using Write Concerns in Mongoose:**

```javascript
// Connecting with Write Concern
const uri =
  "mongodb://localhost:27017/myDb?replicaSet=rs0&w=majority&wtimeoutMS=5000";
mongoose.connect(uri);

// On a specific operation
await User.create([{ name: "Alice" }], {
  writeConcern: { w: "majority", j: true },
});
```

### 4. Connecting to a Sharded Cluster

Connecting to a Sharded Cluster from Node.js is identical to connecting to a single standalone instance. You connect to multiple `mongos` query routers for high availability, not the individual shards.

```javascript
const mongoose = require("mongoose");

// Connect to multiple 'mongos' proxy instances
const uri = "mongodb://mongos1:27017,mongos2:27017/myDatabase";

mongoose.connect(uri).then(() => {
  console.log("Connected to Sharded Cluster via mongos successfully");
});
```

---

## Practical Example: Setting Up a Replica Set Locally

Here are the step-by-step commands to set up a local replica set with 3 nodes for testing and development.

### 1. Preparing the Environment (Linux/macOS)

First, you need to create dedicated data directories for each MongoDB instance.

```bash
# Create directories for the three nodes
mkdir db1 db2 db3

# If you need to remove old data from a previous run:
rm -r db1 db2 db3
```

### 2. Starting MongoDB Instances

We need to start three separate `mongod` instances on different ports. They must all be assigned to the same replica set (in this example, `rs1`). Open three separate terminal windows to run these.

```bash
# Node 1 (Port 27018)
mongod --port 27018 --dbpath db1 --replSet rs1

# Node 2 (Port 27019)
mongod --port 27019 --dbpath db2 --replSet rs1

# Node 3 (Port 27020)
mongod --port 27020 --dbpath db3 --replSet rs1
```

### 3. Configuring the Replica Set

Connect to one of the instances using the MongoDB Shell (`mongosh` or `mongo`) to initialize the replica set.

```javascript
// Connect to the MongoDB shell on the first port
$ mongosh --port 27018

// Initiate the replica set with all 3 members configuration
rs.initiate({
  _id: "rs1",
  members: [
    { _id: 0, host: "localhost:27018" },
    { _id: 1, host: "localhost:27019" },
    { _id: 2, host: "localhost:27020" },
  ],
});

// Check the status to see which node became PRIMARY and which became SECONDARY
rs.status();
```

### 4. Reading Data from Secondary Nodes

By default, MongoDB protects against stale data by preventing read operations directly from secondary nodes. If you try to query a secondary node, you will get an error. You must explicitly allow reads.

```javascript
// Connect to a secondary node shell (e.g., mongo --port 27019)

// Option 1: Legacy command to enable reads
rs.slaveOk();

// Option 2: Modern approach using read preferences
db.getMongo().setReadPref("secondary");
```

---

## Sharding Commands in MongoDB Shell (`mongosh`)

As an application developer, you will usually only connect to the `mongos` router. However, setting up sharding requires administrative commands:

```javascript
// Connect to the mongos router
// 1. Enable sharding for a specific database
sh.enableSharding("myLargeAppDB");

// 2. Creating an Index for the Shard Key
// Before you can shard a collection, MongoDB requires an index on the fields that will serve as the Shard Key.
// We use a compound index (tenantId and createdAt) because:
// - Targeted Queries: Queries filtering by tenantId route to a specific shard.
// - Avoiding Jumbo Chunks: Adding 'createdAt' prevents a single massive tenant from creating an unbreakable chunk of data.
db.getSiblingDB("myLargeAppDB").users.createIndex({
  tenantId: 1,
  createdAt: 1,
});

// 3. Sharding the Collection
// Once the index exists, you execute the shardCollection command.
// What happens underneath? The collection is logically partitioned into 'chunks'.
// A background 'Balancer' process monitors the chunks across shards.
// If Shard A has 50 chunks and Shard B has 10, the Balancer automatically migrates
// chunks from A to B (without downtime) to ensure even data distribution.
sh.shardCollection("myLargeAppDB.users", { tenantId: 1, createdAt: 1 });

// 4. View the sharding status and how chunks are distributed
sh.status();
```
