# MCP Server Flow with Windsurf

## The Full Picture — All Players

```
┌─────────────┐    ┌───────────────┐    ┌─────────────┐    ┌─────────────┐
│    You      │    │   Windsurf    │    │  MCP Server │    │ Backend API │
│  (Human)    │    │  (Host+LLM)  │    │  (Node.js)  │    │  (REST etc) │
└─────────────┘    └───────────────┘    └─────────────┘    └─────────────┘
```

---

## Phase 1 — Server Startup (happens once at launch)

```
Windsurf launches
       │
       ▼
Spawns your MCP server as child process
  $ node my-server.js
       │
       ▼
StdioServerTransport created
  stdin/stdout pipes connected
       │
       ▼
Handshake happens
  Windsurf ──► "initialize"
  Server   ──► "here are my capabilities"
       │
       ▼
Discovery happens
  Windsurf ──► "tools/list"
  Server   ──► [
               {
                 name: "get-weather",
                 description: "Gets weather for a city",
                 inputSchema: {         ◄── Zod generates this!
                   city: { type: "string" }
                 }
               }
             ]
       │
       ▼
Windsurf now KNOWS your tools exist
Server goes IDLE, waiting on stdin
```

---

## Phase 2 — You Type a Prompt

```
You type: "What is the weather in Singapore?"
       │
       ▼
Windsurf sends your text to the LLM
```

---

## Phase 3 — LLM Decides to Use a Tool

```
LLM receives:
  - Your message: "What is the weather in Singapore?"
  - Tool list:    [get-weather, ...]    ◄── from discovery phase
       │
       ▼
LLM THINKS:
  "The user wants weather.
   I have a tool called get-weather.
   It needs a { city: string }.
   I should call it with city = Singapore"
       │
       ▼
LLM responds with tool call (NOT plain text yet):
  {
    "tool": "get-weather",
    "arguments": { "city": "Singapore" }
  }
```

---

## Phase 4 — Windsurf → MCP Server (via stdio)

```
Windsurf takes the LLM's tool call
       │
       ▼
Writes JSON-RPC message to your server's stdin:

  {
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "get-weather",
      "arguments": { "city": "Singapore" }
    },
    "id": 5
  }
       │
       ▼  (this travels through the stdin PIPE)
       │
StdioServerTransport reads it from stdin
       │
       ▼
transport.onmessage fires   ◄── the wire we talked about!
       │
       ▼
McpServer.handleMessage() receives it
```

---

## Phase 5 — Zod Validates the Input

```
McpServer got: { city: "Singapore" }
       │
       ▼
Zod schema runs:    ◄── THIS is where Zod is involved

  z.object({ city: z.string() })
    .parse({ city: "Singapore" })
       │
       ├── ✅ Valid → proceeds to your handler
       └── ❌ Invalid → returns error immediately
                        (API never gets called)
```

---

## Phase 6 — Your Tool Handler Calls the Backend API

```
Zod passed ✅
       │
       ▼
Your tool handler runs:

  server.tool("get-weather", ..., async ({ city }) => {

    // YOUR CODE — calls the real backend
    const response = await fetch(
      `https://api.weather.com/v1?city=${city}`
    )
    const data = await response.json()

    return {
      content: [{
        type: "text",
        text: `Weather in ${city}: ${data.temp}°C, ${data.condition}`
      }]
    }
  })
       │
       ▼
fetch() goes out to the internet → Backend API responds
```

---

## Phase 7 — Response Travels Back

```
Backend API returns data
       │
       ▼
Your handler returns result to McpServer
       │
       ▼
McpServer wraps it in JSON-RPC response:
  {
    "jsonrpc": "2.0",
    "result": {
      "content": [{
        "type": "text",
        "text": "Weather in Singapore: 31°C, Sunny"
      }]
    },
    "id": 5
  }
       │
       ▼  (this travels through stdout PIPE)
       │
StdioServerTransport writes it to stdout
       │
       ▼
Windsurf reads it from stdout
       │
       ▼
Sends tool result BACK to LLM:
  "The tool returned: Weather in Singapore: 31°C, Sunny"
       │
       ▼
LLM generates final human-friendly response:
  "The weather in Singapore is currently 31°C and sunny! 🌤️"
       │
       ▼
You see the answer in Windsurf chat ✅
```

---

## Complete Flow in One Diagram

```
You type prompt
      │
      ▼
LLM decides tool needed
      │
      ▼
Windsurf
  │  writes to stdin
  ▼
StdioServerTransport  ◄── stdio involved here
  │  onmessage fires
  ▼
McpServer             ◄── MCP involved here
  │  routes to handler
  ▼
Zod validates input   ◄── Zod involved here
  │  passes if valid
  ▼
Your tool handler
  │  fetch() call
  ▼
Backend API           ◄── real data lives here
  │  returns data
  ▼
Your handler returns result
  │
  ▼
McpServer wraps response
  │  writes to stdout
  ▼
StdioServerTransport  ◄── stdio involved here
  │
  ▼
Windsurf reads stdout
  │  sends to LLM
  ▼
LLM writes final answer
  │
  ▼
You see response in Windsurf chat ✅
```

---

## Where Each Technology Fits

| Technology | Role | When |
|------------|------|------|
| **stdio** | The pipe/tunnel between Windsurf and your server | Whole time |
| **MCP** | The protocol/language they speak over that pipe | Every message |
| **Zod** | The bouncer — validates tool arguments before your code runs | Phase 5 |
| **LLM** | The brain — decides which tool to call and what to say | Phase 3 & end |
| **Your handler** | The worker — actually calls the real API | Phase 6 |
| **Backend API** | The data source | Phase 6 |
