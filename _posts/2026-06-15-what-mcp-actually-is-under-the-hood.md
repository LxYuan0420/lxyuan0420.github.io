---
layout: post
title: "What MCP Actually Is Under the Hood"
description: "MCP is not magic. It is JSON-RPC, JSON Schema, transports, and a small set of AI-specific workflows."
---

Ask someone what MCP is and you will usually get the high-level answer:

> MCP stands for Model Context Protocol. It provides a standard way for LLM applications to connect to external tools and data sources.

Sometimes you will hear the shorter version:

> MCP is like USB-C for AI.

Those explanations are useful because they explain why MCP exists. They do not explain how MCP actually works.

If you are an engineer, the more interesting questions are:

- What exactly is MCP?
- Is it an API?
- Is it a server?
- What gets sent over the wire?
- How does Cursor discover GitHub tools?
- How does a model know what arguments a tool accepts?
- What actually happens when the model creates a GitHub issue?

To answer those questions, we need to go below the abstraction layer.

At its core, MCP is a protocol specification built on top of JSON-RPC 2.0. It does not invent a new message format. Instead, it standardizes how AI applications discover capabilities, read context, and invoke tools.

A useful mental model:

```text
MCP
|-- JSON-RPC 2.0
|-- JSON Schema
`-- Transport layer
```

Each layer has a job:

- JSON-RPC defines how messages are structured.
- JSON Schema defines how tools describe their inputs.
- The transport layer moves messages between systems.
- MCP defines the standard methods and workflows built on top.

Before MCP makes sense, JSON-RPC has to make sense.

## Start with JSON-RPC 2.0

RPC stands for remote procedure call.

The idea is simple. Instead of calling a function locally:

```python
subtract(42, 23)
```

you call a function running somewhere else.

That "somewhere else" could be another process, another machine, another service, or another container.

JSON-RPC is a lightweight protocol that standardizes how those requests and responses are represented.

A simple request-response exchange looks like this:

```json
{
  "jsonrpc": "2.0",
  "method": "subtract",
  "params": [42, 23],
  "id": 1
}
```

Response:

```json
{
  "jsonrpc": "2.0",
  "result": 19,
  "id": 1
}
```

The fields are doing specific work:

- `jsonrpc` is the protocol version.
- `method` is the function to invoke.
- `params` are the arguments passed to the function.
- `id` is the request identifier.
- `result` is the returned value.

The `id` field lets a client match responses to requests. That matters when multiple requests are in flight at the same time.

JSON-RPC also defines a standard error structure:

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32601,
    "message": "Method not found"
  },
  "id": 1
}
```

Instead of every implementation inventing its own error format, the protocol gives everyone the same envelope.

JSON-RPC also supports notifications. A notification is a request that does not expect a response. It omits the `id` field:

```json
{
  "jsonrpc": "2.0",
  "method": "logEvent",
  "params": {
    "event": "user_login"
  }
}
```

At this point, JSON-RPC has given us a message format. It has not told us what methods exist or what those methods mean.

That is where MCP comes in.

## Why JSON-RPC instead of REST?

REST is resource-oriented.

A REST API models the world as resources:

```http
GET /users/123
POST /issues
```

JSON-RPC is action-oriented.

Instead of interacting with resource URLs, clients invoke named methods:

```json
{
  "method": "create_issue",
  "params": {
    "title": "Authentication Bug"
  }
}
```

That maps naturally to how AI tools work.

Most AI tool interactions are not:

> Retrieve resource 123.

They are:

- Search this database.
- Create a GitHub issue.
- Execute this query.
- Summarize this document.

Those actions already look like remote function calls. MCP's protocol shape follows from that.

## What transport agnostic means

One phrase that appears often in MCP documentation is:

> MCP is transport agnostic.

The important idea is that the protocol message stays the same even when the delivery mechanism changes.

JSON-RPC messages can move over different transports. In the current MCP spec examples, the standard client-server transports are:

- `stdio`
- Streamable HTTP

With `stdio`, the host application launches the MCP server as a subprocess. The client writes JSON-RPC messages to the server's `stdin`, and the server writes JSON-RPC messages back through `stdout`.

With Streamable HTTP, the server exposes an HTTP endpoint. The client sends JSON-RPC messages over HTTP requests, and the server can optionally use Server-Sent Events for streaming.

A local Cursor installation might talk to a GitHub MCP server over `stdio`.

A remote MCP server might use Streamable HTTP.

The bytes move differently, but the protocol messages are still JSON-RPC.

## JSON Schema is how tools become callable

Most MCP explanations say MCP supports tool calling.

The more interesting question is:

> How does the model know how to call the tool?

The answer is JSON Schema.

When an MCP server exposes a tool, it does not just provide a tool name. It provides a machine-readable contract describing the tool's inputs.

For example:

```json
{
  "name": "create_issue",
  "description": "Create a GitHub issue",
  "inputSchema": {
    "type": "object",
    "properties": {
      "title": {
        "type": "string"
      }
    },
    "required": ["title"]
  }
}
```

This schema tells the client:

- The tool expects an object.
- The object has a field called `title`.
- `title` must be a string.
- `title` is required.

Without JSON Schema, a model might know that a tool exists, but it would not know how to invoke it correctly.

With JSON Schema, tools become self-describing. This is one of the most important parts of MCP, and one of the easiest parts to overlook.

## Hosts, clients, and servers

One common misconception is that the model talks directly to an MCP server.

That is not what happens.

The architecture looks more like this:

```text
User
  |
  v
Host application
  |
  v
MCP client
  |
  v
MCP server
  |
  v
External system
```

Examples of host applications:

- Claude Desktop
- Cursor
- Windsurf
- Custom AI applications

The host is the application the user interacts with.

Inside the host are one or more MCP clients. Each MCP client manages a connection to one MCP server.

The MCP client is responsible for:

- Speaking the MCP protocol.
- Managing the session lifecycle.
- Discovering capabilities.
- Executing tool calls.
- Returning results to the host and model context.

The MCP server exposes capabilities. These generally fall into three server-side categories:

- Tools: executable actions, such as `create_issue`, `send_email`, or `query_database`.
- Resources: readable data, such as files, database rows, or API records.
- Prompts: reusable prompt templates exposed by the server.

The model itself does not speak MCP. The model decides what action should happen. The host and MCP client translate that decision into protocol messages.

## A real example: Cursor and GitHub

Suppose Cursor connects to a GitHub MCP server.

At the beginning, this is just two systems exchanging JSON-RPC messages.

The first step is initialization. The client sends an `initialize` request:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-11-25",
    "capabilities": {},
    "clientInfo": {
      "name": "Cursor",
      "version": "1.0"
    }
  }
}
```

The server responds with its supported protocol version and capabilities.

After successful initialization, the client sends an initialized notification:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

The lifecycle looks like this:

```text
Client connects
  |
  v
initialize
  |
  v
initialize response
  |
  v
notifications/initialized
  |
  v
normal operation
```

Only after that handshake does normal capability discovery begin.

The client asks the server what tools are available:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/list"
}
```

The server returns a list of tools and their schemas. The client now knows what the server can do.

Imagine the server exposes a tool shaped like this:

```text
create_issue(title: string)
```

The tool definition and schema are made available to the model through the host application's tool interface or context.

Now the user says:

> Create a GitHub issue called "Authentication Bug".

The model decides that `create_issue` should be used.

The MCP client converts that decision into a protocol message:

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "create_issue",
    "arguments": {
      "title": "Authentication Bug"
    }
  }
}
```

The GitHub MCP server executes the operation and returns a JSON-RPC response to the client. The raw JSON-RPC response is not usually shown to the model directly. The host extracts the tool result and passes it back into the model context, typically as a tool-result message in the next turn.

From the user's perspective, this feels like a natural conversation.

Under the hood, it is a sequence of protocol messages.

## MCP is bidirectional

Many developers approach MCP with a REST mental model. That model is useful, but incomplete.

In a typical REST system, the client initiates requests and the server responds.

MCP is more flexible. During an active session, both sides can send messages. Servers can send notifications, request model sampling from the client, ask for user input through elicitation, and participate in capability negotiation.

That is why MCP feels more like a protocol than a normal API wrapper. It is not just a collection of endpoints. It defines an ongoing session between two participants.

## Security and trust boundaries

MCP is only part of the security story.

The current spec includes authorization guidance for HTTP transports based on OAuth 2.1 conventions. Local `stdio` servers often use local credentials or environment-based configuration instead.

But the protocol does not automatically make a deployment safe.

Authentication, authorization policies, permission controls, audit logs, and infrastructure security still belong to the host and server operators.

For example, an MCP server exposed over HTTP without proper authentication can still be protocol-correct. It is just accessible to anyone who can reach it.

MCP also introduces an AI-specific risk: tool outputs become model inputs.

Common risks include:

- Prompt injection: a tool returns text that tries to influence the model's future behavior.
- Tool poisoning: a malicious server exposes misleading tool descriptions or responses.
- Excessive permissions: a server exposes broader capabilities than the user actually needs.
- Compromised servers: a compromised MCP server becomes part of the model's decision-making environment.

## The better mental model

Many explanations describe MCP as a protocol that allows AI models to use tools. That is correct, but still high-level. A more precise description is:

> MCP is a standardized RPC layer for AI systems.

The model is not directly connected to GitHub, directly connected to a database, or directly executing code. The host application sits in the middle, and the MCP client turns the model's intent into protocol calls.

Instead:

1. JSON-RPC defines the message structure.
2. JSON Schema defines tool contracts.
3. MCP defines standard capabilities and workflows.
4. A transport moves the messages.
5. MCP clients translate model decisions into protocol calls.
6. MCP servers expose tools, resources, and prompts.

Everything eventually reduces to structured messages moving between systems.

Once you understand that, MCP stops looking like magic and starts looking like what it really is: a protocol specification built on top of a well-established RPC standard.

## Where this post stops

There are many details this post skips on purpose: pagination, resource templates, progress notifications, cancellation, reconnection behavior, OAuth edge cases, schema dialects, task-augmented execution, and production deployment patterns.

Those details matter when you are implementing or operating an MCP server. But they are not the right first layer.

For me, this is the useful stopping point: high-level enough to preserve the mental model, but one step deeper than the usual "USB-C for AI" explanation. Once you understand JSON-RPC messages, JSON Schema tool contracts, transports, and the host-client-server split, the rest of the spec becomes much easier to read.

## Sources

- [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification)
- [Latenode, "Model Context Protocol & JSON-RPC: How MCP Actually Works"](https://latenode.com/blog/model-context-protocol-json-rpc)
- [MCP architecture overview](https://modelcontextprotocol.io/docs/learn/architecture)
- [MCP lifecycle](https://modelcontextprotocol.io/specification/2025-11-25/basic/lifecycle)
- [MCP transports](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)
- [MCP tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)
- [MCP authorization](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization)
