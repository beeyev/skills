# Xquik X Data

Agent guidance for building against Xquik's public REST API, remote MCP
surface, and webhook payloads for X data workflows.

## What It Covers

- Choosing between REST API, remote MCP, and webhook flows.
- Source-checking Xquik docs, OpenAPI, and MCP manifest before implementation.
- Handling credentials through approved runtime secret paths.
- Keeping examples small, opt-in, and validation-friendly.

## Install

```bash
npx skills add beeyev/skills --skill xquik-x-data
```

See the [repository README](../../README.md) for more options.

## Sources

- https://docs.xquik.com/api-reference/overview
- https://xquik.com/openapi.json
- https://xquik.com/.well-known/mcp.json
