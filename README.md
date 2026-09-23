# Odile Labs — MCP server

A hosted [Model Context Protocol](https://modelcontextprotocol.io) server at
`https://odilelabs.com/mcp`, so an agent — Claude Code, Codex, or anything else
that speaks MCP — can work with Odile Labs on a person's behalf.

Before any sign-in, three tools answer:

- **`get_receptionist_benchmark`** — Odile Labs' published benchmark of AI phone
  receptionists: real test calls to public lines, scored against stated
  measures, with the evidence per measure. Run by Odile Labs, which sells two
  of the lines tested, and the output says so first.
- **`get_catalog`** and **`estimate_media_job`** — the live Make price list, and
  the price of a job before anything is uploaded.

Signed in, the agent can use:

- **Make** — turn footage or photos into platform-ready video and images for
  Instagram, YouTube and TikTok.
- **Meeting notes** — send a note-taker to a Zoom, Google Meet or Microsoft
  Teams call and email the summary to the people the account owner names.
  These tools appear only on a deployment where Meeting notes is switched on.

**What is not here.** Odile Labs' phone receptionist — sold as After Hours in
the United States and Canada and as Dusklin everywhere else — is set up on the
website (odilelabs.com/afterhours, dusklin.com), not through this server. The
benchmark tool is the server's only receptionist surface.

This repository is the public record of the server: what it exposes, how it
authenticates, and what it costs. The service itself is closed source.

## Connecting

Streamable HTTP, at `https://odilelabs.com/mcp`. Claude Code:

    claude mcp add --transport http odile https://odilelabs.com/mcp

Anything else that takes a remote MCP URL works the same way. Sign-in happens in
a browser on first use.

## Authentication

**Three tools are public; everything that touches an account is OAuth 2.1.**
Without a token, a client can complete the handshake, list the tools, and call
`get_receptionist_benchmark`, `get_catalog` and `estimate_media_job` — constants,
arithmetic and a published dataset, none of it about any account. Every other
tool, and any request that carries a token (valid or not), gets the standard
`401` with the discovery header the MCP specification asks for:

    WWW-Authenticate: Bearer realm="OAuth",
      resource_metadata="https://odilelabs.com/.well-known/oauth-protected-resource/mcp"

Both documents a client needs are served:

- `https://odilelabs.com/.well-known/oauth-protected-resource`
- `https://odilelabs.com/.well-known/oauth-authorization-server`

Permissions are ticked on a consent screen. A tool that spends money never runs
without a ceiling the account owner has agreed to — see `unlock_job` below.

## The tools

Nine are always present. Four more appear only on an account with Meeting Notes
enabled, so a client that sees thirteen and a client that sees nine are both
correct.

### Public — no account

| Tool | What it does |
|---|---|
| `get_receptionist_benchmark` | The published receptionist benchmark: date, lines tested, points by scenario, the fairness note and the limitations. Filter by `line_id` or `scenario_id`; pass `include_evidence` for the quoted evidence per measure. Vendor-run by Odile Labs, and it says so |

### Make — video and photos

| Tool | What it does |
|---|---|
| `get_catalog` | The live price list and limits, read from the same source the website charges from |
| `estimate_media_job` | Prices a job **before** anything is uploaded or charged |
| `get_balance` | The signed-in customer's credit balance and when it expires |
| `prepare_upload` | A 30-minute upload token and the command to use it |
| `create_media_job` | Queues a job from uploaded sources. Costs nothing — credits are spent at unlock |
| `get_job` | One job by id, or the ten most recent |
| `get_downloads` | Signed links to every finished file of a job |
| `unlock_job` | Spends credits. Refuses, charging nothing, if the job costs more than `max_credits`, is not ready, or the balance is short |

`unlock_job` is the only tool that moves money, it takes an explicit
`max_credits` the customer agreed to, and unlocking the same job twice charges
once.

### Meeting notes

| Tool | What it does |
|---|---|
| `schedule_meeting_notes` | Sends the note-taker to a Zoom, Meet or Teams link at a stated time, for a roster of addresses the owner gave you |
| `get_meeting` | One meeting, or the recent ones |
| `get_meeting_summary` | The summary once the call is done |
| `cancel_meeting_notes` | Cancels a scheduled note-taker |

Meeting minutes are reserved when a call is scheduled and the unused part is
returned. Every minute on the call is charged, rounded up to the minute.

## What it costs

Credits for media, minutes for meetings, both quoted before they are spent.
`get_catalog` and `estimate_media_job` are the honest way to find out what a job
costs without committing to it. Prices are on
[odilelabs.com/developers](https://odilelabs.com/developers) and are read live by
the server rather than copied here, so this file cannot go stale against them.

## Registry entry

[`server.json`](./server.json) is the entry for the
[official MCP registry](https://registry.modelcontextprotocol.io), published
under the `com.odilelabs` namespace, which is authenticated by DNS on
`odilelabs.com`.

## Who runs it

OdileUSA L.L.C., Florida. [odilelabs.com](https://odilelabs.com) ·
[hello@odilelabs.com](mailto:hello@odilelabs.com)
