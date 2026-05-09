---
title: "The browser-shaped hole in ACP"
description: "ACP got the primitives right, but the spec stops at stdio. What gets reinvented when teams build to the browser, and what closing the gap would require."
pubDatetime: 2026-05-09
tags: ["agents", "acp", "protocols", "websockets"]
---

The Agent Client Protocol is the right shape for non-trivial agent integrations. It's also missing a transport for clients that don't speak stdio. Both things are true, and this post is about what that costs you in practice.

*The agent runtime is solved; the wire isn't.*

ACP defines a small, sharp set of primitives for the kind of long-running, tool-using, permission-gated agents that are now the actual product surface. The primitives are general. The wire isn't.

The spec ends at stdio. What teams build to bridge it tends to reinvent the primitives more loosely than ACP itself defines. That's the cost worth naming; the rest of this post walks through what it looks like.

## The problem class

The shape of the integration looks like this. A user asks for something non-trivial: refactor a codebase, draft a plan, set up an environment. The agent runs for minutes, calling tools as it goes — opens files, runs scripts, queries APIs.

Sometimes it pauses to ask a question — *should I delete this?* — and blocks on the answer. The user closes the laptop halfway through and reopens it from a different tab on a different machine. The conversation is still there.

Each of those is a different hard problem dressed as a single user-facing experience. Streaming alone is easy. Streaming with mid-stream tool calls and permission blocks is not.

Resumption across devices means the agent's state has to live somewhere durable, not in the client. Cancel has to terminate the tool, not just drop the stream. Each has a known answer; they don't compose when stacked.

This is the shape of any serious agent product that does more than chat — the kind where the agent does real work on the user's behalf, not just generating text. Code assistants are one example. Operations-style agents that touch infrastructure are another. Anything with a permission step, a tool call, and a session that survives the user closing their tab is in the same class.

## The naive path

When you hit this problem class, the instinct is reasonable: REST plus a WebSocket, JSON-RPC over your own custom shape, whatever the stack already speaks.

Streaming responses go on the WebSocket. Tool calls become events on the same channel. Permission requests become a question-event the client UI handles. It works in the demo.

Then it breaks at the edges, in ways that look easy until you're in them.

The first thing that breaks is mid-stream blocking. The agent is streaming tokens; then it needs to ask the user to confirm a delete.

With no protocol-level concept of a request that pauses execution, you're either inventing one — a question event, a response event, correlated by id, with timeouts — or letting the agent's own promise pattern leak into the wire. Both work. Both are something you build, not something you receive.

The second is reconnect. The user closes the tab during a long-running tool call. They reopen it. The agent is somewhere mid-execution, the client has lost its event stream, and you have to design how the new connection picks up state.

Replay since last seen event? Snapshot plus delta? Have the client tell the server which event id it last processed? Every choice is a small protocol you're inventing, with edge cases you don't know about until you ship.

The third is cancel. The user hits cancel; the front-end sends an event; the agent is mid-tool-call. Does the tool run to completion? Stop where it is? Fail the prompt? The protocol has no opinion. You build one.

Most teams get something working. Few get something correct. The answer to all of these is already specified — just not for any transport that reaches a browser.

## What ACP gives you

Four primitives carry most of ACP's surface for a non-trivial agent integration.

`session/prompt` and `session/update`. The agent receives a prompt and emits a stream of structured updates back: text chunks, thinking chunks, tool calls with arguments and results, the stop condition. The structure is what matters.

An ad-hoc stream is just bytes; a structured one tells the client what kind of thing each chunk is and how to render it. Without that, every client builds its own schema and they all disagree.

`session/request_permission`. Mid-execution, the agent can pause and ask the client a question — usually *are you sure I should do this?* — and block until the answer comes back.

The primitive turns out to be more general than its name: any blocking request from agent to user can flow through it. The protocol guarantees the call is request/response, not fire-and-forget on a bus — a stronger contract than it sounds.

`session/load`. When a client reconnects — new tab, restart, fresh device — it doesn't remember anything. It hands the agent a session id; the agent re-emits history through the same `session/update` channel it uses for live events.

Same renderer, same code path, no separate "history fetch + reconcile" track. The agent is the source of truth; the client is a view.

`cancel`. A notification that interrupts whatever the agent is doing. Clean cancellation is hard to retrofit onto a streaming protocol — most ad-hoc designs let cancel arrive but leave the tool call halfway through a write.

ACP's cancel is fire-and-forget at the wire level, but it presumes the agent has organized its execution under something the cancel can actually interrupt.

These are the primitives a non-trivial agent integration needs, regardless of whether the client is an editor, a web app, or a service. The shape is general; the wire is not.

## What gets reinvented at the transport boundary

Even teams who adopt ACP early build a parallel HTTP path for their web clients, because the spec stops at stdio.

[Opencode](https://github.com/anomalyco/opencode) is a worked example. They added ACP support first, then built a separate HTTP API for their webapp. Both share a session core, but the wire contract diverges at the boundary — and that's where the cost shows up.

Take the permission flow as one example. ACP's `session/request_permission` is RPC: one client, message-correlated, blocking round-trip.

Opencode's HTTP equivalent broadcasts a `permission.asked` event on an SSE stream and exposes a separate `POST /permission/:id/reply` for the response.

With two browser tabs open, both see the request. The first POST wins; the second 404s. There's no session affinity in the protocol — only a map keyed on a request id.

![ACP permission flow as a clean RPC: agent calls session/request_permission, client blocks, client returns the response.](/assets/acp-rpc.svg)

![Opencode HTTP permission flow: agent emits permission.asked to the server, the server fans the event out over SSE to both tabs, both tabs POST a reply, the first call resolves the permission, the second receives 404.](/assets/opencode-broadcast.svg)

*Same protocol-level need; two very different shapes.*

This isn't a knock on Opencode's engineering. They obviously know ACP cold; they shipped it before the HTTP API existed. The cost is structural, and it shows up at the transport boundary every time.

*What ACP defines as RPC, the web reinvents as broadcast-and-race.*

## Closing the gap

If you're building agent products that go beyond chat — long-running, tool-using, permission-gated, resumable — ACP gets you most of the way, even outside an editor. The primitives do the work; the transport gap is what's left.

A WebSocket transport for ACP would need to specify, in the abstract: a connect-time handshake binding the connection to a session id, each `session/update` framed as one WS message.

`session/request_permission` preserved as a paired request/response, not flattened to broadcast events. Resumption via `session/load` on reconnect, through the same update channel.

None of it is hard to design. None of it is in the spec.

Until that's published, every team building beyond stdio finds their own version. And the version they find is looser than the one ACP already defines.
