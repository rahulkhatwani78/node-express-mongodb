# EJS, Multer, WebSockets, REST APIs, HTTP APIs, GraphQL, RPC and gRPC

This guide provides an overview, detailed description, and practical examples of various essential web technologies commonly used in Node.js and Express applications.

---

## 1. EJS (Embedded JavaScript Templating)

### Description

EJS is a simple templating language that lets you generate HTML markup with plain JavaScript. No religiousness about how to organize things. No strict syntax or tags. Just plain JavaScript in simple templates. It helps in rendering dynamic data from the server directly into the HTML pages.

### Key Features

- Rapid development using plain JavaScript.
- Simple syntax (`<% %>` for execution, `<%= %>` for output).
- Supports partials (includes) for reusable components.

### Example (Node.js/Express)

**Setup:**

```bash
npm install ejs express
```

**`server.js`:**

```javascript
const express = require("express");
const app = express();

// Set EJS as the view engine
app.set("view engine", "ejs");

app.get("/", (req, res) => {
  const user = { name: "John Doe", age: 30 };
  const hobbies = ["Reading", "Coding", "Gaming"];

  // Render the 'index.ejs' file in the 'views' folder
  res.render("index", { user: user, hobbies: hobbies });
});

app.listen(3000, () => console.log("Server running on port 3000"));
```

**`views/index.ejs`:**

```html
<!DOCTYPE html>
<html>
  <head>
    <title>EJS Example</title>
  </head>
  <body>
    <h1>Welcome, <%= user.name %>!</h1>
    <p>You are <%= user.age %> years old.</p>

    <h3>Your Hobbies:</h3>
    <ul>
      <% hobbies.forEach(function(hobby) { %>
      <li><%= hobby %></li>
      <% }); %>
    </ul>
  </body>
</html>
```

---

## 2. Multer

### Description

Multer is a Node.js middleware for handling `multipart/form-data`, which is primarily used for uploading files. It is written on top of busboy for maximum efficiency.

### Key Features

- Easily handle file uploads in Express.
- Support for storage configuration (disk or memory).
- File filtering and size limits.

### Example (Node.js/Express)

**Setup:**

```bash
npm install multer express
```

**`server.js`:**

```javascript
const express = require("express");
const multer = require("multer");
const path = require("path");

const app = express();

// Configure storage
const storage = multer.diskStorage({
  destination: function (req, file, cb) {
    cb(null, "uploads/"); // Create an 'uploads' directory manually
  },
  filename: function (req, file, cb) {
    const uniqueSuffix = Date.now() + "-" + Math.round(Math.random() * 1e9);
    cb(
      null,
      file.fieldname + "-" + uniqueSuffix + path.extname(file.originalname),
    );
  },
});

const upload = multer({ storage: storage });

// Single file upload route
app.post("/upload", upload.single("profilePic"), (req, res) => {
  if (!req.file) {
    return res.status(400).send("No file uploaded.");
  }
  res.send(`File uploaded successfully: ${req.file.filename}`);
});

app.listen(3000, () => console.log("Server started on port 3000"));
```

**HTML Form (`index.html`):**

```html
<form action="/upload" method="post" enctype="multipart/form-data">
  <input type="file" name="profilePic" />
  <button type="submit">Upload</button>
</form>
```

---

## 3. Web Sockets

### Description

WebSocket is a computer communications protocol providing full-duplex communication channels over a single TCP connection. Unlike HTTP, which is request-response based, WebSockets allow for real-time, two-way, persistent communication between a client and a server.

### Key Features

- Real-time communication.
- Low latency.
- Event-driven architecture.
- Ideal for chat applications, live dashboards, and multiplayer games.

### Example (Node.js using `socket.io`)

**Setup:**

```bash
npm install express socket.io
```

**`server.js`:**

```javascript
const express = require("express");
const http = require("http");
const { Server } = require("socket.io");

const app = express();
const server = http.createServer(app);
const io = new Server(server);

app.get("/", (req, res) => {
  res.sendFile(__dirname + "/index.html");
});

io.on("connection", (socket) => {
  console.log("A user connected");

  // Listen for a message from the client
  socket.on("chat message", (msg) => {
    console.log("Message received: " + msg);
    // Broadcast the message to everyone
    io.emit("chat message", msg);
  });

  socket.on("disconnect", () => {
    console.log("User disconnected");
  });
});

server.listen(3000, () => {
  console.log("listening on *:3000");
});
```

**`index.html`:**

```html
<!DOCTYPE html>
<html>
  <body>
    <ul id="messages"></ul>
    <form id="form" action="">
      <input id="input" autocomplete="off" /><button>Send</button>
    </form>

    <!-- Load Socket.io client script -->
    <script src="/socket.io/socket.io.js"></script>
    <script>
      var socket = io();
      var form = document.getElementById("form");
      var input = document.getElementById("input");

      form.addEventListener("submit", function (e) {
        e.preventDefault();
        if (input.value) {
          socket.emit("chat message", input.value);
          input.value = "";
        }
      });

      socket.on("chat message", function (msg) {
        var item = document.createElement("li");
        item.textContent = msg;
        document.getElementById("messages").appendChild(item);
      });
    </script>
  </body>
</html>
```

---

## 4. Basics of REST APIs

### Description

REST (Representational State Transfer) is an architectural style for designing networked applications. A REST API (also known as RESTful API) is an application programming interface that conforms to the constraints of REST architectural style. It uses HTTP requests to access and use data.

### Key Principles

1.  **Client-Server Architecture:** Separation of UI (client) from data storage (server).
2.  **Statelessness:** Each request from client to server must contain all information needed to understand the request. The server does not store client context.
3.  **Cacheability:** Responses must define themselves as cacheable or not.
4.  **Uniform Interface:** Consistent fundamental design (using standard HTTP methods, URIs as resources).

### HTTP Methods in REST

- **GET:** Retrieve a resource.
- **POST:** Create a new resource.
- **PUT:** Update an existing resource completely.
- **PATCH:** Partially update an existing resource.
- **DELETE:** Remove a resource.

### Example Rules

- **Resource Nouns:** Use nouns, not verbs in endpoints (`/users` instead of `/getUsers`).
- **Plurals:** Use plural nouns (`/users/123` instead of `/user/123`).

---

## 5. HTTP APIs

### Description

"HTTP API" is a broader term that encompasses any API that uses HTTP as its communication protocol. All REST APIs are HTTP APIs, but not all HTTP APIs are RESTful. If an API uses HTTP methods without strictly adhering to REST principles (like statelessness or uniform interface), it is just an HTTP API.

### REST API vs HTTP API

- **HTTP API:** Simply uses HTTP to communicate. Might use endpoints like `/create-user` (POST) or `/delete-user` (POST), treating URLs as command actions rather than resources.
- **REST API:** Follows strict architectural constraints (methods align with CRUD, noun-based URLs).

### Example (Non-RESTful HTTP API)

```javascript
// RPC-style over HTTP - an HTTP API that is NOT RESTful
app.post("/api/userActions/createUser", (req, res) => {
  /* logic */
});
app.post("/api/userActions/deleteUser", (req, res) => {
  /* logic */
});
```

_(In a REST API, these would be `POST /users` and `DELETE /users/:id`)_

---

## 6. GraphQL

### Description

GraphQL is a query language for your API, and a server-side runtime for executing queries using a type system you define for your data. Developed by Facebook, it provides a more efficient, powerful, and flexible alternative to REST.

### Key Features

- **Ask for exactly what you want:** The client dictates the shape and amount of data returned, preventing over-fetching (getting too much data) and under-fetching (needing multiple requests).
- **Single Endpoint:** Usually operates over a single HTTP endpoint (e.g., `/graphql`), unlike REST which employs multiple URLs for multiple resources.
- **Strongly Typed:** Schemas define exactly what data is available and what formats are expected.

### Example

**GraphQL Query (sent by client):**

```graphql
query {
  user(id: "1") {
    name
    email
    posts {
      title
    }
  }
}
```

**JSON Response (returned by server):**

```json
{
  "data": {
    "user": {
      "name": "John Doe",
      "email": "john@example.com",
      "posts": [{ "title": "My first GraphQL post" }]
    }
  }
}
```

---

## 7. RPC and gRPC

### RPC (Remote Procedure Call)

**Description:** RPC is an inter-process communication protocol that allows a computer program to cause a subroutine or procedure to execute in another address space (commonly on another computer on a shared network), without the programmer explicitly coding the details for this remote interaction.

- You call a function that looks local, but it executes on a remote server.
- Focuses on _actions_ or _commands_ rather than _resources_ (like REST).
- E.g., JSON-RPC or XML-RPC.

### gRPC (Google Remote Procedure Call)

**Description:** gRPC is a modern, open-source, high-performance RPC framework developed by Google. It uses HTTP/2 for transport, Protocol Buffers (Protobufs) as the interface description language, and provides features such as authentication, bidirectional streaming and flow control.

### Key Features of gRPC

- **Protocol Buffers (Protobuf):** A language-neutral, platform-neutral, extensible mechanism for serializing structured data. It's binary and much faster/smaller than JSON.
- **HTTP/2:** Supports multiplexing, server push, and header compression.
- **Streaming:** Supports client-streaming, server-streaming, and bidirectional streaming.
- **Multi-language:** Generate client and server stubs automatically in multiple languages (Go, Java, Python, Node.js, C++, etc.).

### Example (Conceptual Protocol Buffer Definition - `.proto` file)

```protobuf
syntax = "proto3";

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```
