# Getting Started with DNotifier in Node.js

A step-by-step guide to creating a DNotifier project, finding your App ID and App Secret, and connecting to DNotifier from a Node.js backend — so you can build real-time agents, notifications, and AI-powered features.

Follow every step in order and you'll go from "no account" to "connected and receiving messages."

---

## Table of Contents

- [What You'll Do](#what-youll-do)
- [Prerequisites](#prerequisites)
- [Part 1 — Dashboard Setup](#part-1--dashboard-setup)
- [Part 2 — Node.js Project Setup](#part-2--nodejs-project-setup)
- [Part 3 — Connecting from Node.js](#part-3--connecting-from-nodejs)
- [Part 4 — Sending & Receiving Messages](#part-4--sending--receiving-messages)
- [Part 5 — AI and RAG on the Server](#part-5--ai-and-rag-on-the-server)
- [Understanding Incoming Messages](#understanding-incoming-messages)
- [Common Errors & Fixes](#common-errors--fixes)
- [Related Topics](#related-topics)
- [Summary](#summary)

---

## What You'll Do

- Sign up / log in to the DNotifier Dashboard
- Create a new project
- Find your **App ID** and **App Secret**
- Install the DNotifier SDK in a Node.js project
- Write the connection code and confirm it connects successfully
- Send and receive your first message
- Understand the shape of incoming messages

## Prerequisites

- Node.js installed on your machine
- A Node.js backend project (Express or plain Node) — or a fresh empty folder to start one
- A DNotifier account (free to sign up)

---

## Part 1 — Dashboard Setup

### Step 1: Log In to the DNotifier Dashboard

Go to [app.dnotifier.com](https://app.dnotifier.com) and sign up (Google sign-up shown below), or log in if you already have an account.

![DNotifier sign-up / login page](./screenshots/login-page.png)

### Step 2: Create a New Project

From the dashboard, go to **Projects** in the sidebar. Click **+ Create New** (top right, or the button in the empty state) and give your project a name.

![My Projects page showing the + Create New button](./screenshots/create-project.png)

### Step 3: Find Your App ID

Open your project. Your **App ID** is shown on the project's overview page, with a copy icon next to it.

![App ID field](./screenshots/app-id.png)
> *(Blurred here — yours will show a real value like `app_xxxxxxxxxxxxxxxx`)*

### Step 4: Copy Your App Secret

> **Note:** The App Secret is **not** the same as an API Key. It's shown automatically in a popup right after you create the project (Step 2) — it is **not** something you generate separately.

This is the only time you'll see it, so copy it immediately and store it safely before closing the popup.

![App Secret popup shown once after project creation](./screenshots/app-secret.png)
> *(Blurred here — yours will show a real value)*

> ⚠️ **Warning:** Treat this App Secret like a password. Never commit it to GitHub or expose it in frontend/browser code. Store it only in your backend's `.env` file.

---

## Part 2 — Node.js Project Setup

### Step 5: Install the DNotifier SDK

In your Node.js backend project folder, run:

```bash
npm install @dnotifier-realtime/dnotifier
```

Using yarn or pnpm:

```bash
yarn add @dnotifier-realtime/dnotifier
pnpm add @dnotifier-realtime/dnotifier
```

> 💡 **Tip:** If `ws` isn't already in your project's dependencies, install it too:
> ```bash
> npm install ws
> ```
> Node.js doesn't have a browser-native WebSocket, so the SDK needs `ws` to fill that gap. (Not needed if you're connecting from a browser — browsers have `WebSocket` built in.)

> 💡 **Note:** The package ships as ESM (`import { DNotifier } from "@dnotifier-realtime/dnotifier"`) with a CommonJS build available via `require()`, plus TypeScript declarations included — no separate `@types` package needed.

### Step 6: Add Your Credentials to `.env`

Create (or open) your `.env` file in the backend root and add:

```env
DNOTIFIER_APP_ID=your_app_id_here
DNOTIFIER_SECRET=your_secret_key_here
```

> 💡 **Tip:** Make sure `.env` is listed in your `.gitignore` file so these values are never pushed to a public repository.

---

## Part 3 — Connecting from Node.js

### Step 7: Write the Connection Code

Create a file (e.g. `agent.js`) with the following:

```javascript
import { DNotifier } from "@dnotifier-realtime/dnotifier";
import WebSocket from "ws";

const notifier = new DNotifier({
  appId: process.env.DNOTIFIER_APP_ID,
  secret: process.env.DNOTIFIER_SECRET,
  transport: "ws",
  userId: "my_backend_service",   // any identifier you choose
  WebSocketImpl: WebSocket,       // required in Node.js
  onConnected: () => console.log("Connected ✓"),
  onMessage: (data) => {
    console.log("Received:", data.payload.toJSON());
  },
  onDisconnected: () => console.log("Disconnected"),
});

await notifier.connect();

export { notifier };
```

> 💡 **Tip:** `WebSocketImpl: WebSocket` is the key line for Node.js. Without it, you'll get:
> `Error: No WebSocket implementation provided.`
> This line isn't needed in browser code — browsers have `WebSocket` built in.

### Step 8: Run It and Confirm the Connection

Start your backend as usual:

```bash
npm run dev
# or
node index.js
```

You should see `Connected ✓` printed in your terminal, confirming the SDK successfully authenticated and connected.

---

## Part 4 — Sending & Receiving Messages

The core real-time pattern: send a JSON payload to a receiver ID, and handle incoming messages in `onMessage`.

### Sending a Test Message

To confirm two-way communication, send a message to yourself and see it echoed back (or send from a second connected client):

```javascript
await notifier.send({
  senderId: "my_backend_service",
  receiverId: "my_backend_service",
  data: { type: "ping", message: "hello DNotifier" },
});
```

If everything is wired up correctly, your `onMessage` callback will log the payload back to the terminal.

### Sending to Another User

```javascript
await notifier.send({
  senderId: "my_backend_service",
  receiverId: "user-bob",
  data: {
    type: "text",
    text: "Hello from DNotifier!",
  },
  saveHistory: true, // default — set false to skip chat history persistence
});
```

### Echo Test (Two Processes)

**Process 1 — bob listens:**

```javascript
import { DNotifier } from "@dnotifier-realtime/dnotifier";
import WebSocket from "ws";

const bob = new DNotifier({
  appId: process.env.DNOTIFIER_APP_ID,
  secret: process.env.DNOTIFIER_SECRET,
  transport: "ws",
  userId: "user-bob",
  WebSocketImpl: WebSocket,
  onConnected: () => console.log("Bob is online"),
  onMessage: (msg) => console.log("Bob received:", msg.payload.toJSON()),
  onDisconnected: () => {},
});

await bob.connect();
```

**Process 2 — your backend sends:** use the connection code from [Part 3](#part-3--connecting-from-nodejs), then call `notifier.send()` with `receiverId: "user-bob"` as shown above.

Expected on bob's console:
```
Bob received: { type: 'text', text: 'Hello from DNotifier!' }
```
### The `send()` Contract

| Field | Required | Description |
|---|---|---|
| `senderId` | Yes | Must match the connecting user's `userId` |
| `receiverId` | One of | Single recipient user ID |
| `receiverIds` | One of | Multiple recipients (see [Related Topics](#related-topics)) |
| `data` | Yes | JSON-serializable payload — typically `{ type: "text", text: "..." }` |
| `saveHistory` | No | Default `true` — persist for chat history APIs |

> 💡 **Tip:** DNotifier does not enforce a fixed `type` enum — use any string your app understands (`text`, `image`, `audio`, `doc`, `ping`, or your own custom types).

### Requirements for Send & Receive

- `transport: "ws"` — real-time send/receive uses WebSocket
- `connect()` completed — check that `isConnected` is `true` before sending
- Matching `appId` — sender and receiver must use the same application

> 💡 **Note:** The receiver does not need to be online at send time for history-backed chat — the message is still saved. For live delivery, the receiver must be connected.

---

## Part 5 — AI and RAG on the Server

For AI-powered features (like answering questions from a knowledge base), keep prompts and document ingestion on trusted backend infrastructure. This uses `transport: "http"` instead of `"ws"`, since it's a direct request/response with DNotifier's own AI service — not agent-to-agent messaging.

> 💡 **Tip:** This requires a **paid DNotifier plan** with AI enabled. On the free plan, `notifier.sendAI()` and `notifier.addDocument()` calls will not work.

```javascript
const notifier = new DNotifier({
  appId: process.env.DNOTIFIER_APP_ID,
  secret: process.env.DNOTIFIER_SECRET,
  transport: "http",
  userId: "svc-ai",
  onConnected: () => {},
  onMessage: () => {},
  onDisconnected: () => {},
});

await notifier.connect();

await notifier.addDocument({
  senderId: "svc-ai",
  recordId: "policy-v2",
  content: "...",
  type: "text",
});

const answer = await notifier.sendAI({
  senderId: userId,
  message: { useKnowledgeBase: true, text: question },
});
```

- **`addDocument()`** — uploads a document (FAQ, policy, article) into the project's knowledge base, identified by `recordId`
- **`sendAI()`** — sends a user's question to DNotifier's AI, optionally grounding the answer in the uploaded knowledge base (`useKnowledgeBase: true`)

This is also the pattern to use for batch processing, cron-driven reports, and webhook-triggered AI — running workflows on the server rather than in the client.

---

## Understanding Incoming Messages

Every incoming real-time message arrives as a structured object with two parts:
- **`metadata`** — `sender` (prefixed `appId:userId` address), `timestamp` (Unix epoch ms), `id` (dedupe key, when assigned by the server), `type` (transport-level hint)
- **`payload`** — your application data, with conversion methods: `toJSON()`, `toString()`, `raw()`, `toBase64()`

Flow from send to receive:
```
send({ data: { type, text, ... } })
│
▼
DNotifier cloud
│
▼
onMessage({ metadata, payload })
│
├── metadata.sender → who sent it
├── metadata.timestamp → when
└── payload.toJSON() → your { type, text, ... }
```
Full reference: [Understanding messages](https://dnotifier.gitbook.io/product-docs/getting-started/understanding-messages)

---

## Common Errors & Fixes

| Error | Fix |
|---|---|
| `No WebSocket implementation provided` | You forgot `WebSocketImpl: WebSocket` (Node.js only — not needed in browsers). |
| `invalid project credentials` | Double-check you copied the App ID and Secret from the correct **project**, not the Organization-level ID. |
| `senderId required` | Make sure every `notifier.send()` call includes `senderId`, and either `receiverId` or `receiverIds`. |
| Receiver gets nothing | Receiver must use the same `appId` and be connected for live delivery. |
| Auth OK but no delivery | Use `transport: "ws"`, not `"http"`, for real-time messaging. |

---

## Related Topics

| Topic | Link |
|---|---|
| First message walkthrough | [Your first message](https://dnotifier.gitbook.io/product-docs/getting-started/first-message) |
| Structured media | [Structured payloads](https://dnotifier.gitbook.io/product-docs/realtime-communication/structured-payloads) |
| Multiple recipients | [Multiple receivers](https://dnotifier.gitbook.io/product-docs/realtime-communication/multiple-receivers) |
| Building chat | [Building 1:1 chat](https://dnotifier.gitbook.io/product-docs/chat/building-one-to-one-chat) |

---

## Summary

You've now:

- ✅ Created a DNotifier project
- ✅ Located your App ID and App Secret
- ✅ Installed the SDK in a Node.js backend
- ✅ Successfully connected and confirmed two-way messaging
- ✅ Learned the shape of incoming messages (`metadata` + `payload`)

From here, you can build out message handlers, AI agent logic, or real-time features on top of this connection.
