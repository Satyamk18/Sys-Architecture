# Uber's MCP Gateway

*System design study notes · November 23, 2026*

**Reference:** [Uber Engineering: Designing MCP Gateway](https://www.uber.com/in/en/blog/designing-mcp-gateway/)

## Table of Contents

1. [The Big Idea](#1-the-big-idea)
2. [MCP Basics](#2-mcp-basics)
3. [High-Level Architecture](#3-high-level-architecture)
4. [API Discovery: AutoCrawler](#4-api-discovery-autocrawler)
5. [MCP Registry: What Tools Exist?](#5-mcp-registry-what-tools-exist)
6. [Proxy Gateway: How a Tool Runs](#6-proxy-gateway-how-a-tool-runs)
7. [Reducing LLM Context: Three Techniques](#7-reducing-llm-context-three-techniques)
8. [Third-Party Integrations](#8-third-party-integrations)
9. [MCP Client Lifecycle in Our Own Application](#9-mcp-client-lifecycle-in-our-own-application)
10. [Security and Ownership](#10-security-and-ownership)
11. [Key System Design Lessons](#11-key-system-design-lessons)
12. [Quick Revision](#12-quick-revision)

---

## 1. The Big Idea

Uber has thousands of backend services: Driver, Trip, Payment, Pricing, Fraud, and many more.

Uber wants AI agents to use these services. The naive approach is to ask every service team to build and operate its own MCP server. At Uber's scale, this creates duplicated work, inconsistent security, difficult tool discovery, and a large number of tool definitions for the LLM to process.

**Uber's solution:** Build a centralized MCP Gateway that makes existing backend capabilities available to AI agents through MCP.

The gateway acts like an API gateway designed for AI agents. It provides a common entry point, manages tools, applies security controls, and routes tool calls to existing services.

The article describes a platform with hundreds of MCP servers and thousands of tools.

## 2. MCP Basics

MCP (Model Context Protocol) is a standard way for an AI application to discover and call tools exposed by another system.

| Term | Simple meaning |
|---|---|
| MCP Host | The AI application the user interacts with |
| MCP Client | The component inside the application that communicates with an MCP server |
| MCP Server | A program that exposes tools and other capabilities |
| Tool | An action the AI can request, such as `get_issue` or `get_driver` |
| `tools/list` | Ask the server which tools it provides |
| `tools/call` | Ask the server to execute a particular tool |

Example: an AI application connects to Atlassian's remote MCP server, discovers Jira tools, and calls a tool to retrieve an issue. The MCP server handles the Jira-specific integration.

**Important:** MCP standardizes how the client and server communicate. It does not dictate how the server implements the underlying business operation.

## 3. High-Level Architecture

Uber separates the platform into two major areas:

- **Control Plane:** Knows which tools exist and manages their configuration, ownership, and availability.
- **Data Plane:** Handles live tool requests and forwards them to backend services.

```text
                   AI AGENTS
                       |
                   MCP Client
                       |
                       v
                +--------------+
                | MCP Gateway  |
                +--------------+
                       |
              ---------------------
              |                   |
              v                   v
         MCP Registry        Proxy Gateway
        (Control Plane)      (Data Plane)
              |                   |
          AutoCrawler        Authorization
              |              Rate Limiting
          API Registry       Protocol Mapping
                                  |
                                  v
                               Muttley
                                  |
                     -------------------------
                     |           |           |
                    HTTP        gRPC      TChannel
                     |           |           |
                     v           v           v
                      Backend Microservices
```

The registry manages tool metadata and configuration. The proxy gateway handles actual execution.

## 4. API Discovery: AutoCrawler

### What is an IDL Registry?

IDL means **Interface Definition Language**. In simple terms, an IDL is a formal description of an API: its methods, inputs, outputs, and data types.

For example, a gRPC service can be described using Protocol Buffers:

```protobuf
service DriverService {
    rpc GetDriver(GetDriverRequest)
        returns (GetDriverResponse);
}
```

This definition tells us that the service has a `GetDriver` method, what request it accepts, and what response it returns.

Uber already has a registry of these API definitions.

### What does AutoCrawler do?

AutoCrawler is a Cadence-powered workflow that discovers existing APIs and helps turn them into MCP tools.

1. Scans the API definition registry.
2. Detects new services, methods, or schema changes.
3. Reads method names and request/response definitions.
4. Uses an LLM to generate descriptions that are easier for agents to understand.
5. Converts API schemas into MCP-compatible tool input schemas.
6. Registers the generated tools in the MCP Registry.

Example:

Existing API method:

```text
GetDriver(driverId)
```

Possible MCP tool definition:

```json
{
  "name": "get_driver",
  "description": "Fetch driver information using a driver ID",
  "inputSchema": {
    "type": "object",
    "properties": {
      "driverId": { "type": "string" }
    },
    "required": ["driverId"]
  }
}
```

**Important:** Discovering an API does not automatically make it available to agents. The owning team reviews and explicitly enables the generated tools.

### What about services that already have an MCP server?

AutoCrawler can also discover native MCP servers. It detects their heartbeat and calls `listTools` to retrieve their tool definitions and schemas, then registers them with the platform.

There are therefore two discovery paths:

| Existing system | How Uber discovers it |
|---|---|
| Existing HTTP/gRPC services | Reads API definitions from the IDL Registry |
| Native MCP server | Detects the server and calls `listTools` |

## 5. MCP Registry: What Tools Exist?

Think of the MCP Registry as a central catalogue of available tools and their configuration.

It stores information such as:

- MCP server details.
- Tool names and descriptions.
- Input/output schemas.
- Owning team.
- Enabled/disabled state.
- Security and access configuration.

Example:

```text
Server: Driver Service
Owner: Driver Team

Tools:
  get_driver         ENABLED
  update_driver      DISABLED
  get_driver_rating  ENABLED
```

The registry answers: **What tools are available, who owns them, and are they enabled?**

It does not execute the backend business logic.

## 6. Proxy Gateway: How a Tool Runs

Suppose an AI agent wants to run:

`get_driver(driverId = 123)`

The runtime flow is:

1. The MCP request reaches the gateway.
2. The gateway identifies the requested tool and its target backend.
3. It checks whether the caller is allowed to use that tool.
4. It converts the MCP request arguments into the format expected by the backend.
5. It forwards the request through Muttley.
6. The backend service executes its normal business logic.
7. The gateway converts the result into an MCP-compatible response.
8. Sensitive data can be redacted before the response is returned to the agent.

### What is Muttley?

Muttley is Uber's service-mesh sidecar. It provides existing downstream service communication and routing capabilities.

The gateway reuses this infrastructure rather than rebuilding all service-to-service communication itself.

### Why protocol conversion?

The AI agent speaks MCP, but existing services may speak different protocols.

```text
MCP request (JSON)
       |
       v
Gateway maps arguments to backend format
       |
       +---- HTTP request
       |
       +---- Protobuf request -> gRPC
       |
       +---- TChannel request
       |
       v
Existing backend service
```

**Key benefit:** Existing microservices do not need to implement MCP or change their business logic.

## 7. Reducing LLM Context: Three Techniques

### The problem

If an LLM receives thousands of tool definitions, every name, description, and input schema consumes context. Large tool responses consume even more.

Uber describes three ways to reduce this overhead.

### A. Omni MCP: Discover Tools Only When Needed

Instead of exposing thousands of tools to the LLM at once, Omni MCP provides a small set of discovery and execution tools.

| Omni MCP tool | Responsibility |
|---|---|
| `discover_server` | Find relevant MCP servers |
| `discover_tools` | Find tools within a server |
| `get_tool_schema` | Retrieve the schema for a selected tool |
| `invoke_tool` | Execute the selected tool |

Example: the user asks, "Find the last five trips of driver 123."

```text
1. discover_server("driver trip history")
                 |
                 v
             Trip Service
                 |
                 v
2. discover_tools("Trip Service")
                 |
                 v
          get_driver_trips
                 |
                 v
3. get_tool_schema("get_driver_trips")
                 |
                 v
4. invoke_tool(...)
                 |
                 v
           Trip results
```

The agent progressively discovers only the tools it needs.

**Analogy:** Lazy loading in software engineering. Do not load every tool definition upfront; fetch the relevant ones when required.

### B. Response Projection: Return Only Needed Fields

A backend may return many fields, while the agent needs only a few.

Full response:

```json
{
  "name": "John",
  "email": "john@example.com",
  "phone": "9999999999",
  "rating": 4.8,
  "totalTrips": 500,
  "totalEarnings": 500000
}
```

If the agent only needs the name and rating, it can request those fields:

```json
{
  "responseProjection": ["name", "rating"]
}
```

The gateway filters the response so the agent receives only the requested fields.

**Analogy:** Similar to GraphQL field selection.

**Purpose:** Reduce unnecessary response data and LLM context consumption.

### C. Code Mode: Process Large Results Outside LLM Context

A coding agent may fetch 20,000 lines of logs or a large search result. Sending the entire output into the LLM context is wasteful.

Uber provides `aifx`, a CLI that coding agents can use to discover and invoke MCP tools.

Examples:

```bash
aifx mcp list
aifx mcp search
aifx mcp call
```

The agent can:

1. Discover a tool.
2. Invoke it.
3. Save a large result to a file.
4. Use shell commands or scripts to filter the file.
5. Read only the relevant output into the LLM context.

Example:

```bash
aifx mcp call get_production_logs > logs.json
grep -i "error" logs.json > errors.json
```

**Core idea:** Let the computer process large datasets. Give the LLM only the useful result.

This is especially useful for logs, code repositories, large files, and large API responses.

## 8. Third-Party Integrations

Uber also wants agents to use external platforms such as Jira or Google.

The key challenge is authentication: Uber has its own identity system, while an external platform has its own authentication system. An Uber internal token is not automatically accepted by Jira.

### Simplified architecture

```text
Uber AI Agent
      |
      v
Uber MCP Gateway
      |
      | Internal user identity/token
      v
Third-Party MCP Service
      |
      | Exchanges/obtains external credentials
      v
External MCP Server
      |
      | Calls platform APIs
      v
Jira / Google / Other Platform
```

### Responsibility boundaries

| Component | Responsibility |
|---|---|
| Uber MCP Gateway | Central entry point, internal authorization, rate limiting, and forwarding user identity |
| Third-Party MCP Service | Handles the external authentication/token exchange and integration flow |
| External MCP Server | Exposes external platform capabilities as MCP tools |
| Jira / Google | Owns its data, platform APIs, and business functionality |

An external MCP server is not necessarily hosted by the platform itself. It may be hosted by the platform vendor or another integration provider.

For example, Atlassian offers its own remote MCP server, called Rovo MCP Server.

### Important distinction

- Jira REST API: The actual Jira API used to work with issues, projects, and other Jira data.
- Jira MCP Server: An MCP-compatible interface that exposes Jira operations as tools.
- Third-Party MCP Service: The integration/authentication component in the described Uber pattern.

The central Uber Gateway does not need to implement every external platform's API and authentication details itself.

## 9. MCP Client Lifecycle in Our Own Application

Suppose we build a Spring Boot AI application and want to integrate Jira.

Our application is the MCP Host. It contains an MCP Client that communicates with Atlassian's remote MCP server.

### Application startup

```text
Application starts
       |
       v
Initialize/configure MCP Client
       |
       v
Connect to Jira MCP Server
       |
       v
Call tools/list
       |
       v
Discover tool names, descriptions, and schemas
       |
       v
Make tools available to the AI agent
```

### Runtime

```text
User: "Fetch Jira issue ABC-123"
                 |
                 v
             LLM selects get_issue
                 |
                 v
          Existing MCP Client
                 |
                 v
              tools/call
                 |
                 v
          Jira MCP Server
                 |
                 v
              Jira API
                 |
                 v
              Response
```

### Lifecycle summary

| Component | Typical lifecycle |
|---|---|
| MCP Client | Usually initialized/configured once and reused |
| MCP Connection | Established and reused; reconnect after failures |
| Tool Discovery | At startup and refreshed when needed |
| Tool Invocation | Every time a tool is required |

For multiple MCP servers (Jira, GitHub, Slack, Google Drive), an application can maintain a client connection for each server, often managed through a common client manager.

**Important:** Client reuse is a common design, not a rule that every application must use one permanent connection. Production systems must handle disconnections, reconnection, authentication expiry, timeouts, and tool-list refreshes.

## 10. Security and Ownership

Uber centralizes important controls at the gateway.

| Mechanism | What it does |
|---|---|
| Authorization | Checks whether a caller can execute a tool |
| Rate limiting | Controls request volume |
| Redaction | Removes sensitive information from responses |
| Ownership | Service teams control which tools they expose |
| Approval | Prevents newly discovered APIs from becoming available automatically |

Discovery and authorization are separate: knowing that a tool exists does not mean a particular agent or user is allowed to execute it.

## 11. Key System Design Lessons

| Design decision | Engineering lesson |
|---|---|
| Adapter approach | Expose existing APIs without rewriting backend services |
| Control Plane / Data Plane | Separate configuration and management from live execution |
| AutoCrawler | Automate API discovery and tool generation |
| MCP Registry | Centralize tool metadata, ownership, and availability |
| Protocol conversion | Support different backend protocols behind one interface |
| Muttley integration | Reuse existing service-mesh capabilities |
| Omni MCP | Discover tools progressively instead of loading everything |
| Response Projection | Avoid returning unnecessary fields |
| Code Mode | Process large outputs outside LLM context |
| Third-party integration | Keep external authentication and platform-specific integration out of the central gateway |
| MCP Client reuse | Reuse configured clients/connections and handle their lifecycle properly |

## 12. Quick Revision

**What is Uber building?**

A centralized infrastructure layer that makes thousands of existing backend capabilities accessible to AI agents through MCP.

**What does AutoCrawler do?**

Discovers existing APIs and generates MCP-compatible tool definitions.

**What does the MCP Registry do?**

Stores available tools, schemas, ownership, and configuration.

**What does the Proxy Gateway do?**

Authorizes, translates, routes, and executes tool requests against existing services.

**Why use Omni MCP?**

To discover only relevant tools instead of loading thousands of tool definitions into the LLM context.

**Why use Response Projection?**

To return only the response fields the agent needs.

**Why use Code Mode?**

To process large tool outputs outside the LLM context and send only relevant information to the model.

**Why use a Third-Party MCP Service?**

To handle external authentication and integration with platforms such as Jira without putting every platform-specific detail into the central gateway.

**What is the role of an MCP Client?**

It connects an AI application to an MCP server, discovers its tools, and invokes them.

### One-line takeaway

Uber's MCP Gateway discovers existing APIs, exposes them as AI-accessible tools, centrally manages permissions and execution, connects external platforms, and reduces unnecessary LLM context consumption.

---

## References

- [Uber Engineering: Designing MCP Gateway](https://www.uber.com/in/en/blog/designing-mcp-gateway/)
- [Model Context Protocol Documentation](https://modelcontextprotocol.io/)
- [Atlassian Rovo MCP Server](https://github.com/atlassian/atlassian-mcp-server)

**Status:** Studied and documented  
**Category:** AI Infrastructure / Distributed Systems / System Design
