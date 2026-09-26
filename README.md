# Odile Labs — MCP server

Odile is a hosted, paid service run by OdileUSA L.L.C. This repository holds only
the MCP connector listing and instructions, not the service's source code, and
is not open source.

A hosted [Model Context Protocol](https://modelcontextprotocol.io) server at
`https://odilelabs.com/mcp`, so an agent — Claude Code, Codex, or anything else
that speaks MCP — can work with Odile Labs on a person's behalf.

Before any sign-in, four tools answer:

- **`get_receptionist_benchmark`** — Odile Labs' published benchmark of AI phone
  receptionists: real test calls to public lines, scored against stated
  measures, with the evidence per measure. Run by Odile Labs, which sells two
  of the lines tested, and the output says so first.
- **`get_answering_line_offer`** — the After Hours answering line sold per
  answered call: the packs, what counts as an answered call, what happens when
  the calls run out, and what the owner has to do.
- **`get_catalog`** and **`estimate_media_job`** — the live Odile Formats price
  list, and the price of a job before anything is uploaded.

Signed in, the agent can use:

- **After Hours answering line** — set up an AI answering line for a US
  business, billed per answered call from a prepaid pack, with no monthly fee,
  and read its status.
- **Packs** — a one-tap Stripe Checkout link for Odile Formats credits, meeting
  minutes or answering-line calls. The owner pays on Stripe's own page; the
  connection never sees a card.
- **Odile Formats** — turn footage or photos into platform-ready video and
  images for Instagram, YouTube and TikTok.
- **Meeting notes** — send a note-taker to a Zoom, Google Meet or Microsoft
  Teams call and email the summary to the people the account owner names.
  These tools appear only on a deployment where Meeting notes is switched on.

The monthly After Hours plans (United States and Canada) and Dusklin, the same
receptionist sold everywhere else, are still set up on the website
(odilelabs.com/afterhours, dusklin.com). The per-call line is the one set up
through this server, for businesses in the United States.

This repository is the public record of the server: what it exposes, how it
authenticates, and what it costs. The service itself is closed source.

## Connecting

Streamable HTTP, at `https://odilelabs.com/mcp`. Claude Code:

    claude mcp add --transport http odile https://odilelabs.com/mcp

Anything else that takes a remote MCP URL works the same way. Sign-in happens in
a browser on first use.

## Authentication

**Four tools are public; everything that touches an account is OAuth 2.1.**
Without a token, a client can complete the handshake, list the tools, and call
`get_receptionist_benchmark`, `get_answering_line_offer`, `get_catalog` and
`estimate_media_job` — constants, arithmetic and a published dataset, none of it
about any account. Every other tool, and any request that carries a token
(valid or not), gets the standard `401` with the discovery header the MCP
specification asks for:

    WWW-Authenticate: Bearer realm="OAuth",
      resource_metadata="https://odilelabs.com/.well-known/oauth-protected-resource/mcp"

Both documents a client needs are served:

- `https://odilelabs.com/.well-known/oauth-protected-resource`
- `https://odilelabs.com/.well-known/oauth-authorization-server`

Permissions are ticked on a consent screen. The two that lead to a payment,
`billing:checkout` (checkout links) and `lines:manage` (the answering line), start
unticked; the owner ticks them. A connection made before they existed does not
gain them: reconnect and tick them.

Nothing on this server takes a card. `buy_pack` and `set_up_answering_line`
return a Stripe Checkout link, and nothing is charged until the owner pays on
Stripe's page. `unlock_job` is the only tool that spends a balance a person
already holds, and it never runs without a ceiling the owner agreed to.

## The tools

Thirteen are always present. Four more appear only on an account with Meeting
Notes enabled, so a client that sees seventeen and a client that sees thirteen
are both correct.

### Public — no account

| Tool | What it does |
|---|---|
| `get_receptionist_benchmark` | The published receptionist benchmark: date, lines tested, points by scenario, the fairness note and the limitations. Filter by `line_id` or `scenario_id`; pass `include_evidence` for the quoted evidence per measure. Vendor-run by Odile Labs, and it says so |
| `get_answering_line_offer` | The per-call answering line: the packs and their prices, what counts as an answered call, what happens at zero, where it is sold, what the owner must do, and the terms link |
| `get_catalog` | The live Odile Formats price list and limits, read from the same source the website charges from |
| `estimate_media_job` | Prices an Odile Formats job **before** anything is uploaded or charged |

### After Hours answering line — per answered call

| Tool | What it does |
|---|---|
| `set_up_answering_line` | Takes the business details, builds the greeting, and returns a checkout link for the first pack. Runs only after the owner has seen the terms and said yes. Nothing is set up or charged until that pack is paid |
| `get_answering_line` | The owner's own per-call lines: status, the number, the forwarding codes, calls left, and the last calls charged (caller shown as the last four digits only) |

The line is paid for in prepaid packs, with no monthly fee:

| Pack | Price | Per call |
|---|---|---|
| 10 answered calls | $15 | $1.50 |
| 50 answered calls | $75 | $1.50 |

**An answered call** is a phone call the receptionist answered that lasted at
least 20 seconds. Recognised robocalls, shorter calls and the owner's own test
calls are free. Calls are counted after they end, once each.

When the calls run out the line keeps answering, and up to 10 calls carry over
to the next pack. The owner forwards their business line to the new number with
their carrier's forwarding code; `get_answering_line` returns the codes. The
greeting always says the call is recorded and that the caller is speaking with
an AI.

### Packs

| Tool | What it does |
|---|---|
| `buy_pack` | A Stripe Checkout link for an Odile Formats credit pack, a meeting-minute pack, or a call pack for one of the owner's own answering lines. Returns the link, the price and what the pack adds. The pack is added once Stripe confirms the payment |

### Odile Formats — video and photos

| Tool | What it does |
|---|---|
| `get_balance` | The signed-in customer's credit balance and when it expires |
| `prepare_upload` | A 30-minute upload token and the command to use it |
| `create_media_job` | Queues a job from uploaded sources. Costs nothing — credits are spent at unlock |
| `get_job` | One job by id, or the ten most recent |
| `get_downloads` | Signed links to every finished file of a job |
| `unlock_job` | Spends credits. Refuses, charging nothing, if the job costs more than `max_credits`, is not ready, or the balance is short |

`unlock_job` takes an explicit `max_credits` the customer agreed to, and
unlocking the same job twice charges once.

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

Answered calls for the answering line, credits for media, minutes for meetings,
each bought in prepaid packs and quoted before anything is spent. The call packs
are 10 for $15 and 50 for $75, with no monthly fee. `get_answering_line_offer`,
`get_catalog` and `estimate_media_job` are the honest way to find out what
something costs without committing to it. Every price is on
[odilelabs.com/developers](https://odilelabs.com/developers), is the same for
everyone, and is read live by the server; if this file and the server ever
disagree, the server is right.

## Registry entry

[`server.json`](./server.json) is the entry for the
[official MCP registry](https://registry.modelcontextprotocol.io), published
under the `com.odilelabs` namespace, which is authenticated by DNS on
`odilelabs.com`.

## Who runs it

OdileUSA L.L.C., Florida. [odilelabs.com](https://odilelabs.com) ·
[hello@odilelabs.com](mailto:hello@odilelabs.com)
