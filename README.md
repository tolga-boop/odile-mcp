# Odile Labs — MCP server

Odile is a hosted, paid service run by OdileUSA L.L.C. This repository holds only
the MCP connector listing and instructions, not the service's source code, and
is not open source.

A hosted [Model Context Protocol](https://modelcontextprotocol.io) server at
`https://odilelabs.com/mcp`, so an agent — Claude Code, Codex, or anything else
that speaks MCP — can work with Odile Labs on a person's behalf.

## What it solves

Three things an AI agent cannot do safely on its own:

- **Reach a business that has no API.** Most small businesses only have a phone.
  [Odile Direct](https://odilelabs.com/a2a) finds the business by its phone
  number and leaves it a message or books a time, in seconds
  (`find_business_agent`, `send_to_business`, below). Lookups are free; with a
  free developer key, 50 tasks a month are free, then credits ($10 for 100).
- **Stay inside your API.** An agent holding your API key can do anything that
  key allows: repeat a call in a loop, read another customer's data, change
  something nobody approved. [Odile Gate](https://odilelabs.com/gate) gives each
  app its own key and a list of the calls it may make, stops an agent repeating
  the same call, holds a risky write until a person approves it, and logs every
  call. It is a separate hosted MCP server, `https://gate.odilelabs.com/mcp`
  (registry name `com.odilelabs/gate`, see [gate/](gate/)). Free for 3 apps and
  10,000 calls a month; Pro $29 a month.
- **Answer the phone.** An AI receptionist line for your app that answers
  callers, takes the name, number and need, and texts the owner. A free test
  line, then 25 cents a minute, billed by the second.

Before any sign-in, six tools answer:

- **`find_business_agent`** and **`send_to_business`** — Odile Direct: look a
  business up by its phone number, then leave it a message or a booking request
  (free within per-IP caps).

- **`get_receptionist_benchmark`** — Odile Labs' published benchmark of AI phone
  receptionists: real test calls to public lines, scored against stated
  measures, with the evidence per measure. Run by Odile Labs, which sells two
  of the lines tested, and the output says so first.
- **`get_answering_line_offer`** — the After Hours answering line billed per
  minute: the packs, what is charged, what happens at zero, and what the owner
  has to do.
- **`get_catalog`** and **`estimate_media_job`** — the live Odile Formats price
  list, and the price of a job before anything is uploaded.

Signed in, the agent can use:

- **A free test line** — `try_answering_line` returns a shared test number and a
  4-digit code; calling it reaches an AI receptionist answering as the business
  the agent described. Nothing is booked, sent or charged. Any signed-in account.
- **After Hours answering line** — set up an AI answering line for a US
  business, billed per minute from a prepaid pack, with no monthly fee, and
  read its status, calls and messages.
- **Packs** — a one-tap Stripe Checkout link for Odile Formats credits, meeting
  minutes or phone minutes. The owner pays on Stripe's own page; the
  connection never sees a card.
- **Odile Formats** — turn footage or photos into platform-ready video and
  images for Instagram, YouTube and TikTok.
- **Meeting notes** — send a note-taker to a Zoom, Google Meet or Microsoft
  Teams call and email the summary to the people the account owner names.
  These tools appear only on a deployment where Meeting notes is switched on.

The monthly After Hours plans (United States and Canada) and Dusklin, the same
receptionist sold everywhere else, are still set up on the website
(odilelabs.com/afterhours, dusklin.com). The per-call line is the one set up
through this server, for businesses in the United States, billed per minute.

This repository is the public record of the server: what it exposes, how it
authenticates, and what it costs. The service itself is closed source.

## Connecting

Streamable HTTP, at `https://odilelabs.com/mcp`. Claude Code:

    claude mcp add --transport http odile https://odilelabs.com/mcp

Anything else that takes a remote MCP URL works the same way. Sign-in happens in
a browser on first use.

## Authentication

**Six tools are public; everything that touches an account is OAuth 2.1.**
Without a token, a client can complete the handshake, list the tools, and call
`get_receptionist_benchmark`, `get_answering_line_offer`, `get_catalog`,
`estimate_media_job`, and the two Odile Direct tools `find_business_agent` and
`send_to_business` (free within per-IP caps) — none of them read an account. Every other tool, and any request that carries a token
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

The tools below are always present. Four more appear only on a deployment with
Meeting Notes enabled.

### Public — no account

| Tool | What it does |
|---|---|
| `get_receptionist_benchmark` | The published receptionist benchmark: date, lines tested, points by scenario, the fairness note and the limitations. Filter by `line_id` or `scenario_id`; pass `include_evidence` for the quoted evidence per measure. Vendor-run by Odile Labs, and it says so |
| `get_answering_line_offer` | The answering line billed per minute: the packs and their prices, what is charged, what happens at zero, where it is sold, what the owner must do, and the terms link |
| `get_catalog` | The live Odile Formats price list and limits, read from the same source the website charges from |
| `estimate_media_job` | Prices an Odile Formats job **before** anything is uploaded or charged |

### Odile Direct — reach a business's agent ([odilelabs.com/a2a](https://odilelabs.com/a2a))

| Tool | What it does |
|---|---|
| `find_business_agent` | Resolves a phone number (E.164) to that business's public Odile Direct agent card: what it can do (take a message, check availability, book). 30 lookups per IP per hour without sign-in; 10,000 per key per day signed in |
| `send_to_business` | Sends the business a message, an availability check or a booking through Odile Direct. Without sign-in: free within IP caps. Signed in: the first 50 completed tasks per account each month are free, then a delivered message is 1 credit, a confirmed booking 5, an availability check free (100 credits are $10). Keep `message_id` unchanged on a retry; one keyed task per account runs at a time. Free claimed listings take messages only, up to 20 a day |

### After Hours answering line — per minute

| Tool | What it does |
|---|---|
| `try_answering_line` | Free. A shared test number and a 4-digit code (24 hours, 5 calls; 3 test lines a day per account). Calling it and keying the code reaches an AI receptionist answering as the business described; `hear_url` plays the same receptionist in a browser. Nothing is booked, sent or charged |
| `set_up_answering_line` | Takes the business details, builds the greeting, and returns a checkout link for the first pack. Runs only after the owner has seen the terms and said yes. Nothing is set up or charged until that pack is paid |
| `get_answering_line` | The owner's own lines: status, the number, the forwarding codes, the balance in minutes, and the last calls charged |
| `get_line_calls` | One line's calls, newest first: time, caller, duration, summary, urgent. No audio links |
| `get_line_messages` | One line's messages, newest first: name, number, what they need, urgent |

The line is paid for in prepaid minute packs, with no monthly fee:

| Pack | Price | Per minute |
|---|---|---|
| 40 minutes | $10 | $0.25 |
| 200 minutes | $50 | $0.25 |

Answered phone calls are billed by the second, with a 20-second minimum.
Robocalls, calls from the owner's own numbers, website chats, text messages and
calls whose transcript cannot be read are not charged.

At zero the line keeps answering and the balance goes below zero; the next pack
pays that first. The owner is emailed at 5 minutes left, at zero, and at 10
minutes below zero, and nothing switches the line off on its own. The owner
forwards their business line to the new number with their carrier's forwarding
code; `get_answering_line` returns the codes. The greeting always says the call
is recorded and that the caller is speaking with an AI. Lines set up before
per-minute billing keep their answered-call packs.

The same calls are on a REST API at `https://api.odilelabs.com/v1` with an API
key made at [odilelabs.com/developers](https://odilelabs.com/developers); the
OpenAPI 3.1 spec is [odilelabs.com/openapi.json](https://odilelabs.com/openapi.json).

### Packs

| Tool | What it does |
|---|---|
| `buy_pack` | A Stripe Checkout link for an Odile Formats credit pack, a meeting-minute pack, or a phone-minute pack for one of the owner's own answering lines. Returns the link, the price and what the pack adds. The pack is added once Stripe confirms the payment |

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

Phone minutes for the answering line, credits for media, minutes for meetings,
each bought in prepaid packs and quoted before anything is spent. Phone minutes
are 40 for $10 and 200 for $50 ($0.25 a minute), with no monthly fee, and the
test line is free. `get_answering_line_offer`,
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
