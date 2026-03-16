# MongoDB Shell (`mongosh`)

The MongoDB Shell (`mongosh`) is a fully functional JavaScript and Node.js REPL environment for interacting with MongoDB deployments. You can use it to test queries and operations directly with your database.

## Installation

### Windows
1. Download the installer from the [MongoDB Download Center](https://www.mongodb.com/try/download/shell).
2. Look for the "MongoDB Shell" section.
3. Select your preferred package (usually `.msi` or `.zip`).
4. Run the installer and follow the wizard.
5. If using `.zip`, extract it and add the `bin` folder to your system's PATH variable.

### macOS
You can install `mongosh` using Homebrew:
```bash
brew tap mongodb/brew
brew install mongosh
```

### Linux (Ubuntu/Debian)
```bash
sudo apt-get install -y mongodb-mongosh
```
*(Check the official MongoDB documentation for other Linux distributions.)*

## Connecting to a Database

Once installed, you can connect to your MongoDB instance by running the `mongosh` command in your terminal.

```bash
# Connect to a local MongoDB instance running on default port 27017
mongosh

# Connect to a specific local database
mongosh "mongodb://localhost:27017/myDatabase"

# Connect to MongoDB Atlas (Cloud)
mongosh "mongodb+srv://<username>:<password>@cluster0.mongodb.net/myDatabase"
```

## Basics of Mongo Shell

Once connected, you can run various commands to interact with your databases, collections, and documents.

### Database Commands

- **`show dbs`** or **`show databases`**: Lists all databases on the server.
- **`use <db_name>`**: Switches to the specified database (creates it in memory if it doesn't exist; it will be saved physically once you insert data).
- **`db`**: Shows the current database you are using.
- **`db.dropDatabase()`**: Deletes the current database.

### Collection Commands

- **`show collections`**: Lists all collections in the current database.
- **`db.createCollection("<collection_name>")`**: Explicitly creates a new collection.
- **`db.<collection_name>.drop()`**: Deletes the specified collection.

### CRUD Operations (Create, Read, Update, Delete)

Assuming our collection is named `users`.

#### 1. Create (Insert)
```javascript
// Insert a single document
db.users.insertOne({ name: "Alice", age: 25, status: "active" })

// Insert multiple documents
db.users.insertMany([
  { name: "Bob", age: 30, status: "inactive" },
  { name: "Charlie", age: 28, status: "active" }
])
```

#### 2. Read (Find)
```javascript
// Find all documents in the collection
db.users.find()

// Find documents with a filter
db.users.find({ status: "active" })

// Find a single document
db.users.findOne({ name: "Alice" })

// Format the output to be more readable
// (Note: mongosh formats output by default, but .pretty() can be used in older shells)
db.users.find().pretty() 
```

#### 3. Update
```javascript
// Update a single document (updates the first match)
db.users.updateOne(
  { name: "Alice" }, 
  { $set: { age: 26, status: "inactive" } }
)

// Update multiple documents
db.users.updateMany(
  { status: "active" }, 
  { $set: { lastLogin: new Date() } }
)
```

#### 4. Delete
```javascript
// Delete a single document
db.users.deleteOne({ name: "Alice" })

// Delete multiple documents
db.users.deleteMany({ status: "inactive" })
```

## Useful Tips

- **Clear screen**: Press `Ctrl + L` or type `cls`.
- **Exit shell**: Type `exit`, `quit`, or press `Ctrl + C` twice.
- **Help**: Type `help` to list available commands. For specific collection help, use `db.<collection_name>.help()`.
