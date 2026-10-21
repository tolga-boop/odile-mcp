# Odile Gate — MCP server

Registry name `com.odilelabs/gate`. A hosted MCP server at
`https://gate.odilelabs.com/mcp`, run by OdileUSA L.L.C. This folder holds only
the listing, not the service's source code.

## What it solves

An AI agent, or an app an AI wrote, that holds your API key can do anything
that key allows: repeat the same call in a loop, read another customer's data,
or change something nobody approved. Odile Gate sits in front of your API:

- each app or agent gets its own key and a list of the calls it may make
  (everything else is refused with a reason the agent can read);
- a loop breaker stops an agent that keeps sending the same call;
- Gate Actions give one agent session a few actions for one customer, with
  limits and an expiry, hold a risky write until a person approves that exact
  call, and make a retried write run only once (Idempotency-Key);
- every call, allowed or refused, lands in an audit log you can export.

Request and response bodies are never stored, except, encrypted, a Gate Actions call
waiting for approval. Free: 3 apps, 10,000 calls a month.
Pro: $29 a month, 25 apps, 1,000,000 calls a month, then $10 per extra million.
Details: https://odilelabs.com/gate

## Connecting

The coding agent that writes your app sets Gate up in the same session: it
creates the project, registers each app with only the calls it makes, and reads
the audit log. No install.

    claude mcp add --transport http odile-gate https://gate.odilelabs.com/mcp

The tools answer without a sign-in; `gate_create_project` returns the
project's admin key, to be kept in the project's `.env`, never in code.
