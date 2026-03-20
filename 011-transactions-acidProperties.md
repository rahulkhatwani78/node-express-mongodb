# Transactions and ACID Properties in MongoDB

## What are ACID Properties?

In database systems, **ACID** refers to a set of properties that guarantee that database transactions are processed reliably.

- **Atomicity**: "All or Nothing." A transaction is treated as a single, indivisible logical unit of work. If any operation within the transaction fails, the entire transaction is rolled back, leaving the database unchanged.
- **Consistency**: The database must remain in a consistent state before and after the transaction. It ensures that any transaction will bring the database from one valid state to another, maintaining all predefined rules (like schema validation or constraints).
- **Isolation**: Concurrent transactions occur independently without interfering with each other. The intermediate state of a transaction is invisible to other transactions until it is committed.
- **Durability**: Once a transaction has been committed, it will remain so, even in the event of a power loss, crash, or errors. The modifications are permanently saved to the database.

---

## Transactions in MongoDB

Historically, MongoDB only provided atomic operations on a **single document** level. Because documents can contain nested data (arrays and sub-documents), this was often sufficient for many use cases.

However, complex applications require updates across multiple documents or collections. 
- **MongoDB 4.0** introduced multi-document ACID transactions for Replica Sets.
- **MongoDB 4.2** expanded this support to Sharded Clusters.

### Characteristics of MongoDB Transactions:
1. **Multi-Document & Multi-Collection**: You can update documents across multiple collections and databases within a single transaction.
2. **Standard ACID Guarantees**: They behave just like relational database transactions.
3. **Requirement**: Your MongoDB deployment **must be a Replica Set or Sharded Cluster**. Standalone servers do not support multi-document transactions.

---

## When to Use Transactions

Transactions are ideal for scenarios requiring strong consistency across multiple documents. A classic example is a **Bank Transfer**:

1. Deduct $500 from User A's account.
2. Add $500 to User B's account.

If step 2 fails (e.g., server crash), we must revert step 1 to prevent money from disappearing.

---

## Example 1: MongoDB Shell (`mongosh`)

Here is how you perform a transaction using the native MongoDB shell.

```javascript
// 1. Start a client session
const session = db.getMongo().startSession();

// 2. Start a transaction within the session
session.startTransaction();

try {
  // Get references to the collections, explicitly passing the session
  const accountsCollection = session.getDatabase("bankDB").accounts;
  const transactionsCollection = session.getDatabase("bankDB").transfers;

  // Deduct $100 from Account A
  accountsCollection.updateOne(
    { accountId: "A" }, 
    { $inc: { balance: -100 } }
  );

  // Add $100 to Account B
  accountsCollection.updateOne(
    { accountId: "B" }, 
    { $inc: { balance: 100 } }
  );

  // Record the transfer history
  transactionsCollection.insertOne({
    from: "A",
    to: "B",
    amount: 100,
    date: new Date()
  });

  // 3. Commit the transaction
  session.commitTransaction();
  print("Transaction completed successfully!");

} catch (error) {
  // 4. Abort the transaction if any error occurs
  session.abortTransaction();
  print("Transaction failed. Rolled back operations. Error: " + error);
} finally {
  // 5. Always end the session
  session.endSession();
}
```

---

## Example 2: Node.js with Mongoose

In a typical Node.js application, you'll use an ODM like Mongoose. Working with transactions in Mongoose is straightforward, but remember to pass the `session` to every operation involved.

```javascript
const mongoose = require('mongoose');

// Assuming Account and Transfer models are defined
const Account = mongoose.model('Account');
const Transfer = mongoose.model('Transfer');

async function processBankTransfer(fromAccountId, toAccountId, amount) {
  // 1. Start a session
  const session = await mongoose.startSession();
  
  // 2. Start a transaction
  session.startTransaction();
  
  try {
    // Step A: Deduct amount from sender (check condition: must have sufficient balance)
    // We pass { session } in the options object!
    const sender = await Account.findOneAndUpdate(
      { accountId: fromAccountId, balance: { $gte: amount } },
      { $inc: { balance: -amount } },
      { new: true, session } 
    );

    if (!sender) {
      throw new Error("Insufficient funds or Sender not found");
    }

    // Step B: Add amount to receiver
    await Account.findOneAndUpdate(
      { accountId: toAccountId },
      { $inc: { balance: amount } },
      { session }
    );

    // Step C: Log the transfer history
    await Transfer.create(
      [{
        from: fromAccountId,
        to: toAccountId,
        amount: amount
      }], 
      { session } // Note: create() expects an array of documents when providing options like session
    );

    // 3. Commit the transaction (Saves everything to the database)
    await session.commitTransaction();
    console.log("Transfer successful!");

  } catch (error) {
    // 4. Abort the transaction (Reverts any intermediate changes)
    await session.abortTransaction();
    console.error("Transfer failed, aborted transaction:", error.message);
  } finally {
    // 5. End the session
    session.endSession();
  }
}
```

### Mongoose Convenience Approach (`withTransaction`)
The MongoDB Node.js driver provides a helper called `withTransaction` that automatically handles committing, aborting on error, and retrying transient errors.

```javascript
async function processBankTransferSmart(fromId, toId, amount) {
  const session = await mongoose.startSession();

  try {
    await session.withTransaction(async () => {
      const sender = await Account.findOneAndUpdate(
        { accountId: fromId, balance: { $gte: amount } },
        { $inc: { balance: -amount } },
        { session }
      );
      
      if (!sender) throw new Error("Insufficient funds");

      await Account.updateOne(
        { accountId: toId },
        { $inc: { balance: amount } },
        { session }
      );
    });
    console.log("Transfer successful");
  } catch (error) {
    console.error("Transfer failed:", error.message);
  } finally {
    session.endSession();
  }
}
```

---

## Best Practices & Considerations

1. **Impact on Performance**: Transactions require the database to lock documents. Holding locks for too long can block other operations and degrade performance. **Keep transactions as short as possible**.
2. **Transaction Time Limit**: By default, an active transaction has a maximum execution time of **60 seconds** in MongoDB. If it exceeds this, it will be automatically aborted.
3. **Data Modeling First**: Unlike SQL databases where transactions are heavily relied upon, MongoDB's document model (embedding data) often removes the need for multi-document transactions entirely. Try to structure your schema to rely on single-document atomic operations whenever possible.
4. **Operations Not Allowed**: Some operations cannot be performed inside a transaction, such as creating new collections or dropping collections/indexes. Ensure collections exist before the transaction starts.
