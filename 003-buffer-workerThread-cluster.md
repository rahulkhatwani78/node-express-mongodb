# Buffer, Worker Threads and Cluster

This guide covers three critical concepts for handling high-performance tasks in Node.js: **Buffers** for binary data, **Worker Threads** for CPU-intensive parallel tasks, and **Clusters** for scaling across multiple CPU cores.

---

## 📦 1. Buffer

A **Buffer** is a global object in Node.js used to handle binary data directly. While JavaScript is great at handling strings, many operations (like reading files, network requests, or TCP streams) deal with raw bytes.

### Why do we need Buffers?

- **Binary Data:** Images, videos, and zip files are binary.
- **Streams:** Data in streams is processed in chunks of buffers.
- **Performance:** Buffers allocate memory outside the V8 heap in the "Raw Memory" area, preventing overhead.

### Common Usage

```javascript
// 1. Create a Buffer from a string
const buf1 = Buffer.from("Hello 🚀");
console.log(buf1);
// Output: <Buffer 48 65 6c 6c 6f 20 f0 9f 9a 80> (Hexadecimal)

// 2. Allocate a specific size (e.g., 10 bytes)
const buf2 = Buffer.alloc(10);
buf2.write("NodeJS");
console.log(buf2.toString()); // Output: NodeJS

// 3. Buffer vs String length
console.log("String Length:", "🚀".length); // 2 (UTF-16)
console.log("Buffer Length:", Buffer.from("🚀").length); // 4 (Bytes)
```

> [!TIP]
> Use `Buffer.allocUnsafe()` for even faster allocation if you don't need to zero-fill the memory, but be careful as it may contain old sensitive data.

---

## 🧵 2. Worker Threads

Node.js is **single-threaded** for JavaScript execution. While this is efficient for I/O tasks, **CPU-heavy tasks** (like image processing or complex calculations) will block the Event Loop, making the app unresponsive.

**Worker Threads** allow you to run JavaScript code in parallel on separate threads within the _same process_.

### The Problem: Blocking the Event Loop

```javascript
// This blocks the server for everyone!
app.get("/heavy-task", (req, res) => {
  let count = 0;
  while (count < 1e9) count++; // Blocking...
  res.send("Finished!");
});
```

### The Solution: `worker_threads`

Worker threads share memory using `SharedArrayBuffer` but communicate primarily via messages.

#### `index.js` (Main Thread)

```javascript
const { Worker } = require("worker_threads");

function runWorker(data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker("./worker.js", { workerData: data });
    worker.on("message", resolve);
    worker.on("error", reject);
    worker.on("exit", (code) => {
      if (code !== 0)
        reject(new Error(`Worker stopped with exit code ${code}`));
    });
  });
}

async function main() {
  console.log("1. Starting heavy task...");
  const result = await runWorker(1e9);
  console.log("3. Result from worker:", result);
}

main();
console.log("2. Main thread is NOT blocked!");
```

#### `worker.js` (Worker Thread)

```javascript
const { parentPort, workerData } = require("worker_threads");

let total = 0;
for (let i = 0; i < workerData; i++) {
  total += i;
}

parentPort.postMessage(total);
```

---

## 🚀 3. Cluster Module

While Worker Threads help with CPU tasks _within_ a process, the **Cluster** module allows you to scale your entire application by running multiple instances (processes) of Node.js.

### Why use Clusters?

- **Multi-Core Utilization:** Modern servers have 8, 16, or more CPU cores. A single Node process only uses one.
- **Zero Downtime:** If one worker process crashes, the primary process can fork a new one immediately.
- **Port Sharing:** All worker processes share the same server port (e.g., 8000).

### Implementation with Express

```javascript
const cluster = require("node:cluster");
const os = require("os");
const express = require("express");

const totalCPUs = os.cpus().length;

if (cluster.isPrimary) {
  console.log(`Primary ${process.pid} is running`);

  // Fork workers equal to CPU cores
  for (let i = 0; i < totalCPUs; i++) {
    cluster.fork();
  }

  cluster.on("exit", (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died. Respawning...`);
    cluster.fork();
  });
} else {
  const app = express();
  const PORT = 8000;

  app.get("/", (req, res) => {
    res.json({ message: `Handled by process ${process.pid}` });
  });

  app.listen(PORT, () => {
    console.log(`Worker ${process.pid} started on port ${PORT}`);
  });
}
```

---

## ⚖️ Cluster vs Worker Threads: When to use what?

| Feature           | Cluster Module                        | Worker Threads                    |
| :---------------- | :------------------------------------ | :-------------------------------- |
| **Nature**        | Multi-Process                         | Multi-Thread                      |
| **Memory**        | Isolated (Individual Memory)          | Shared (Shares same memory space) |
| **Best For**      | Scaling Web Servers (High Throughput) | Parallelizing CPU Intensive Tasks |
| **Communication** | IPC (Inter-Process Communication)     | Message Port / SharedArrayBuffer  |

---

## 💡 Interview Pointers: Libuv vs Workers

> [!IMPORTANT]
> **Don't confuse Libuv Thread Pool with Worker Threads!**
>
> - **Libuv Thread Pool:** Handles native asynchronous tasks like `fs` (file system) and `crypto`. It is managed by C++ and you don't write JS for it.
> - Even though Node.js already had the libuv thread pool, it only executes native C/C++ asynchronous operations like file system or crypto tasks. It cannot execute JavaScript code in parallel.
> - **Worker Threads:** Allows **YOU** to run your custom JavaScript logic in parallel.
> - Worker threads were introduced to allow CPU‑intensive JavaScript tasks to run in separate threads, preventing the main event loop from being blocked and improving application performance.

---
