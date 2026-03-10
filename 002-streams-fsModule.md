# Node.js Streams and File System (FS) Module

## 1. Streams in Node.js

A **Stream** is an abstract interface for working with streaming data in Node.js. It allows you to handle data gradually—piece by piece—instead of loading the entire dataset into memory at once. This makes streams highly efficient for handling large files or real-time data.

### Chunks

When a stream reads or writes data, it does so in small pieces called **chunks**. This process prevents memory overflow by ensuring only a small portion of the data is in flight at any given time.

---

## 2. Types of Streams

### I. Readable Streams

Used to read data from a source (e.g., a file, an HTTP request).

**Example:**

```javascript
const fs = require("fs");

const readStream = fs.createReadStream("sample2.txt");

readStream.on("open", () => {
  console.log("Stream Opened!");
});

readStream.on("data", (chunk) => {
  console.log("--- New Chunk Received ---");
  console.log(chunk.toString());
});

readStream.on("end", () => {
  console.log("File Reading Finished!");
});

readStream.on("error", (err) => {
  console.log("Error occurred while reading the file:", err.message);
});

readStream.on("close", () => {
  console.log("Stream Closed!");
});
```

**Expected Output:**

```text
Stream Opened!
--- New Chunk Received ---
[Content of sample2.txt here...]
File Reading Finished!
Stream Closed!
```

### II. Writable Streams

Used to write data to a destination (e.g., a file, an HTTP response).

**Example:**

```javascript
const fs = require("fs");

const writeStream = fs.createWriteStream("file.txt");

writeStream.write("Hello ");
writeStream.write("World");
writeStream.end();

writeStream.on("finish", () => {
  console.log("All data has been written to file.txt");
});
```

**Expected Output:**

```text
All data has been written to file.txt
(A file named 'file.txt' will be created with the content "Hello World")
```

### III. Duplex Streams

A **Duplex** stream can be both read from and written to. The reading and writing parts operate independently.

- **Examples:** TCP sockets, WebSockets.

### IV. Transform Streams

A type of Duplex stream where the output is computed based on the input. They are typically used for modifying data while it is being streamed (e.g., compression or encryption).

**Example (Gzip Compression):**

```javascript
const zlib = require("zlib");
const fs = require("fs");

fs.createReadStream("file.txt")
  .pipe(zlib.createGzip())
  .pipe(fs.createWriteStream("file.txt.gz"));

console.log("File successfully compressed!");
```

**Expected Output:**

```text
File successfully compressed!
(A new file 'file.txt.gz' will appear in the directory)
```

---

## 3. The Pipe Method

The `pipe()` is used to connect two streams together, allowing data to flow from the source to the destination automatically.

**Example:**

```javascript
const fs = require("fs");

fs.createReadStream("input.txt").pipe(fs.createWriteStream("output.txt"));

console.log("Streaming data from input.txt to output.txt...");
```

---

## 4. Stream Pipeline API (Modern Node.js)

The `pipeline()` utility is a cleaner way to handle multiple streams. It automatically handles error propagation and cleans up streams if one of them fails.

**Example:**

```javascript
const { pipeline } = require("stream");
const fs = require("fs");
const zlib = require("zlib");

pipeline(
  fs.createReadStream("sample2.txt"),
  zlib.createGzip(),
  fs.createWriteStream("output.gz"),
  (err) => {
    if (err) {
      console.error("Pipeline failed:", err);
    } else {
      console.log("Pipeline succeeded!");
    }
  },
);
```

---

## 5. File System (FS) Module

The `fs` module provides a lot of very useful functionality to access and interact with the file system. It offers both **Synchronous** (blocking) and **Asynchronous** (non-blocking) methods.

### I. Reading Files

```javascript
const fs = require("fs");

// Synchronous (Blocks execution)
const fileContent = fs.readFileSync("sample2.txt", "utf-8");
console.log("Sync Read:", fileContent);

// Asynchronous (Non-blocking)
fs.readFile("sample2.txt", "utf-8", (err, data) => {
  if (err) {
    console.log("Failed to read file!");
  } else {
    console.log("Async Read:", data);
  }
});
```

### II. Writing Files

Will overwrite the file if it already exists.

```javascript
// Synchronous
fs.writeFileSync("sample2.txt", "Hello there!");

// Asynchronous
fs.writeFile("sample2.txt", "Hello there!", (err) => {
  if (err) {
    console.log("Failed to write file!");
  } else {
    console.log("File written successfully!");
  }
});
```

### III. Appending to Files

Adds data to the end of the file instead of overwriting.

```javascript
// Synchronous
fs.appendFileSync("sample2.txt", " Once again!");

// Asynchronous
fs.appendFile("sample2.txt", " Once again!", (err) => {
  if (err) {
    console.log("Failed to append in file!");
  } else {
    console.log("File appended successfully!");
  }
});
```

### IV. Copying Files

```javascript
// Synchronous
fs.cpSync("sample2.txt", "sample3.txt");

// Asynchronous
fs.cp("sample2.txt", "sample3.txt", (err) => {
  if (err) {
    console.log("Failed to copy file!");
  } else {
    console.log("File copied successfully!");
  }
});
```

### V. Creating Directories (Folders)

```javascript
// Basic directory creation
fs.mkdirSync("hello1");

// Recursive directory creation (creates nested folders if they don't exist)
fs.mkdirSync("hello/devLife/allGood", { recursive: true });

// Asynchronous version
fs.mkdir("hello2/devLife/allGood", { recursive: true }, (err) => {
  if (err) {
    console.log("Failed to create folder!");
  } else {
    console.log("Folder created successfully!");
  }
});
```

---

**Note:** In modern JavaScript, it is often preferred to use the `fs/promises` API for cleaner asynchronous code using `async/await`.
