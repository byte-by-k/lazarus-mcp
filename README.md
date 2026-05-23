# lazarus-mcp

> **MCP Server for the Lazarus DB — let AI agents query, triage, and resolve your failures.**

[![Java](https://img.shields.io/badge/Java-17+-orange)](https://openjdk.org/)
[![MCP](https://img.shields.io/badge/MCP-2025--03--26-blue)](https://modelcontextprotocol.io/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.x-green)](https://spring.io/projects/spring-boot)

---

## Overview

`lazarus-mcp` is an **MCP (Model Context Protocol) server** that exposes the failure database populated by [`lazarus-lib`](https://github.com/byte-by-k/lazarus-lib) as structured, AI-queryable tools.

Once connected, an AI agent (Claude, a LangChain agent, or any MCP-compatible client) can:

- **Inspect** pending failures — what failed, when, and with what payload
- **Categorize** failures by exception type, method, or time window
- **Trigger reprocessing** of specific events or batches
- **Summarize** failure trends in natural language
- **Mark events** as resolved or dead-letter

This closes the loop: `lazarus-lib` captures failures → `lazarus-mcp` makes them intelligible to AI → agents act on them autonomously or assist human operators.

---

## Exposed MCP Tools

### Query Tools

| Tool | Description | Key Parameters |
|---|---|---|
| `list_events` | Paginate lazarus events by status, method, or time range | `status`, `method_name`, `since`, `limit` |
| `get_event` | Fetch a single event by ID with full payload | `event_id` |
| `count_by_status` | Count events grouped by status | `since` (optional) |
| `count_by_exception` | Top N exception types in a time window | `limit`, `since` |
| `count_by_method` | Top N failing methods | `limit`, `since` |
| `search_payload` | Full-text / JSONPath search in `payload_json` | `jsonpath`, `since` |

### Action Tools

| Tool | Description | Key Parameters |
|---|---|---|
| `retry_event` | Trigger reprocessing of a single event via its configured strategy | `event_id` |
| `retry_batch` | Bulk retry all PENDING events matching a filter | `method_name`, `exception_type`, `since` |
| `mark_resolved` | Mark an event as RESOLVED (manual resolution) | `event_id`, `resolution_note` |
| `mark_dead` | Mark an event as DEAD_LETTER (give up) | `event_id`, `reason` |

### Analytical Tools

| Tool | Description |
|---|---|
| `failure_summary` | Returns a structured JSON summary: total counts, top methods, top exceptions, oldest unresolved |
| `suggest_remediation` | (AI-assisted) Given event details, suggest likely root causes |

---

## MCP Resources

In addition to callable tools, the server exposes **MCP Resources** for direct inspection:

| Resource URI | Description |
|---|---|
| `lazarus://events/{event_id}` | Full event record as JSON |
| `lazarus://events/pending` | Live list of all PENDING events |
| `lazarus://summary/today` | Today's failure summary |

---

## Architecture

```
  ┌─────────────────────────────────────────────────────┐
  │                 lazarus-mcp                         │
  │                                                     │
  │  MCP Transport Layer                                │
  │  ┌─────────────────┐   ┌──────────────────────────┐ │
  │  │ HTTP + SSE       │   │ stdio (local dev)        │ │
  │  │ /sse endpoint    │   │ ProcessBuilder transport │ │
  │  └────────┬────────┘   └───────────┬──────────────┘ │
  │           └──────────┬─────────────┘                │
  │                      ▼                              │
  │  MCP Protocol Handler (list_tools / call_tool)      │
  │                      │                              │
  │           ┌──────────▼──────────┐                  │
  │           │   Tool Registry      │                  │
  │           │  - QueryTools        │                  │
  │           │  - ActionTools       │                  │
  │           │  - AnalyticsTools    │                  │
  │           └──────────┬──────────┘                  │
  │                      │                              │
  │           ┌──────────▼──────────┐                  │
  │           │  LazarusEventService │                  │
  │           │  (business logic)    │                  │
  │           └──────────┬──────────┘                  │
  │                      │                              │
  │           ┌──────────▼──────────┐                  │
  │           │  LazarusEventRepo    │ (Spring Data JPA)│
  │           └──────────┬──────────┘                  │
  └──────────────────────┼──────────────────────────────┘
                         │ JDBC
                         ▼
             ┌───────────────────────┐
             │   Lazarus DB          │
             │   (lazarus_events)    │
             └───────────────────────┘
```

---

## Implementation Approach

### Java (Spring Boot MCP Server)

The Spring AI MCP Server starter makes it straightforward to expose Spring beans as MCP tools:

```java
@Configuration
public class LazarusMcpConfig {

    @Bean
    public McpServerFeatures.AsyncToolsRegistrar lazarusTools(LazarusEventService service) {
        return registry -> {

            // list_events tool
            registry.addTool(
                Tool.builder()
                    .name("list_events")
                    .description("List lazarus events filtered by status, method, or time range")
                    .inputSchema(ListEventsInput.JSON_SCHEMA)
                    .build(),
                (exchange, args) -> {
                    ListEventsInput input = objectMapper.convertValue(args, ListEventsInput.class);
                    List<LazarusEvent> events = service.findEvents(input);
                    return Mono.just(new CallToolResult(objectMapper.writeValueAsString(events)));
                }
            );

            // retry_event tool
            registry.addTool(
                Tool.builder()
                    .name("retry_event")
                    .description("Trigger reprocessing of a single lazarus event by its ID")
                    .inputSchema(RetryEventInput.JSON_SCHEMA)
                    .build(),
                (exchange, args) -> {
                    String eventId = args.get("event_id").asText();
                    service.retrigger(eventId);
                    return Mono.just(new CallToolResult("Event " + eventId + " queued for reprocessing"));
                }
            );

            // ... additional tools
        };
    }
}
```

```yaml
# application.yml
spring:
  ai:
    mcp:
      server:
        name: lazarus-mcp
        version: 1.0.0
        transport: HTTP_SSE    # or STDIO
```

### Python Alternative (FastMCP)

If you prefer a Python MCP server that connects to the same database:

```python
from fastmcp import FastMCP
from sqlalchemy import create_engine, text

mcp = FastMCP("lazarus-mcp")
engine = create_engine(DATABASE_URL)

@mcp.tool()
def list_events(status: str = "PENDING", limit: int = 20) -> list[dict]:
    """List lazarus events from the Lazarus DB filtered by status."""
    with engine.connect() as conn:
        rows = conn.execute(
            text("""
                SELECT event_id, method_name, exception_type, status,
                       created_at, payload_json
                FROM lazarus_events
                WHERE status = :status
                ORDER BY created_at DESC
                LIMIT :limit
            """),
            {"status": status, "limit": limit}
        ).mappings().all()
    return [dict(row) for row in rows]

@mcp.tool()
def retry_event(event_id: str) -> str:
    """Trigger reprocessing of a single lazarus event."""
    with engine.connect() as conn:
        conn.execute(
            text("UPDATE lazarus_events SET status = 'REPROCESSING' WHERE event_id = :id"),
            {"id": event_id}
        )
        conn.commit()
    # Publish to Kafka / call Lambda / POST to endpoint
    return f"Event {event_id} queued for reprocessing"

@mcp.tool()
def failure_summary(since_hours: int = 24) -> dict:
    """Return a structured summary of recent failures."""
    # ...aggregate query...

if __name__ == "__main__":
    mcp.run(transport="sse", host="0.0.0.0", port=8081)
```

---

## Integration: Full System Picture

```
  Your Application
  ┌────────────────────────────────────────────────────────────────┐
  │  @Rise(retryOn = {TransientException.class})                   │
  │  public void processOrder(@RisePayload OrderRequest order) {   │
  │      // fails after 3 retries                                  │
  │  }                    │                                        │
  └───────────────────────┼────────────────────────────────────────┘
                          │ persist + publish
                          ▼
              ┌───────────────────────┐
              │   Lazarus DB          │◄────────────────────┐
              │   lazarus_events      │                     │
              └───────────┬───────────┘                    │
                          │ JDBC                            │ JDBC
                          ▼                                 │
              ┌───────────────────────┐          ┌──────────────────────┐
              │  lazarus-mcp          │          │  Background Poller   │
              │  (MCP Server)         │          │  (Kafka / scheduler) │
              └───────────┬───────────┘          └──────────────────────┘
                          │ MCP Protocol
              ┌───────────┴───────────┐
              │                       │
  ┌───────────▼────────┐   ┌──────────▼──────────────────────┐
  │  mcp-agent-java    │   │  mcp-agent-python                │
  │  (Spring AI /      │   │  (Anthropic SDK / LangChain)     │
  │   LangChain4j)     │   │                                  │
  │                    │   │  "Summarize today's failures"    │
  │  REST API for      │   │  "Retry all payment timeouts"    │
  │  ops dashboard     │   │  "Why did order A1 fail?"        │
  └────────────────────┘   └──────────────────────────────────┘
```

---

## Example Agent Conversations

Once connected, AI agents can answer natural-language questions like:

> *"How many events failed in the last hour and which methods are most affected?"*

The agent calls `count_by_method(since="1h")` and `count_by_exception(limit=5, since="1h")` and synthesizes a readable summary.

> *"Retry all pending ORDER events that failed with TransientDataException"*

The agent calls `retry_batch(method_name="processOrder", exception_type="TransientDataException")` and reports how many were queued.

> *"Show me the payload of event abc-123 — it looks like a data issue"*

The agent calls `get_event(event_id="abc-123")`, extracts the JSON payload, and reasons about what field might have caused the failure.

---

## Roadmap

- [ ] `v1.0` — Core query + action tools, HTTP+SSE transport, Java Spring Boot
- [ ] `v1.1` — Python FastMCP alternative implementation
- [ ] `v1.2` — MCP Resources (lazarus://events/pending, lazarus://summary/today)
- [ ] `v1.3` — `suggest_remediation` tool — AI-assisted root cause hints
- [ ] `v2.0` — Multi-application support (tag events by application name)
- [ ] `v2.1` — Grafana-compatible metrics endpoint alongside MCP

---

## Related Projects

- **[lazarus-lib](https://github.com/byte-by-k/lazarus-lib)** — The library that writes to the Lazarus DB
- **[mcp-agent-java](https://github.com/byte-by-k/mcp-agent-java)** — Java agent that consumes this MCP server
- **[mcp-agent-python](https://github.com/byte-by-k/mcp-agent-python)** — Python agent that consumes this MCP server

---

## License

MIT © Kamlesh
