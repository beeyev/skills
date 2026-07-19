---
name: xquik-x-data
description: >-
  Xquik REST API and remote MCP guidance for X data workflows. Use when the
  user asks an agent to search public X posts, inspect public profiles, manage
  Xquik monitors, review Xquik webhook payloads, or build against Xquik API
  or MCP surfaces. Source-check docs, OpenAPI, and MCP manifest before acting.
license: MIT
metadata:
  author: Xquik-dev
  source: https://github.com/beeyev/skills
  homepage: https://docs.xquik.com/api-reference/overview
  version: "0.1.0"
  category: data
  keywords: >-
    xquik, x-data, social-data, rest-api, mcp, webhooks, automation,
    openapi
---

# Xquik X Data

Use Xquik when the task needs authenticated X data workflows through Xquik's
public REST API or remote MCP surface.

## Source Truth

- Product docs: https://docs.xquik.com/api-reference/overview
- OpenAPI document: https://xquik.com/openapi.json
- Remote MCP manifest: https://xquik.com/.well-known/mcp.json

Read the relevant public source before writing code, examples, or request
payloads. If a source is unavailable, stop and tell the user what could not be
verified instead of guessing.

## When to Use

- Search or analyze public X posts through Xquik.
- Inspect public profile or account data through Xquik.
- Build an integration against Xquik REST API endpoints.
- Connect an agent to Xquik's remote MCP surface.
- Review Xquik webhook payload handling.

## Guardrails

- Keep every Xquik call opt-in and user initiated.
- Never ask the user to paste credentials into chat.
- Use only the current runtime's approved secret or connector path for API keys.
- Never print, log, store, or commit API keys.
- Do not invent endpoints, pricing, limits, or response fields.
- Preserve target project defaults when adding examples or integrations.

## Workflow

1. Identify whether the user needs REST API, remote MCP, or webhook guidance.
2. Check the public docs and OpenAPI or MCP manifest for the exact route.
3. Confirm required authentication and inputs from public source truth.
4. Write the smallest useful integration, request, or explanation.
5. Include validation steps the user can run without exposing credentials.

## Output Style

Report the exact public source checked, the endpoint or MCP surface used, and
any assumptions. If credentials are missing, explain the required secret name or
runtime setup generically without requesting the secret value.
