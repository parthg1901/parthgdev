---
title: "Most prompt cache misses are self-inflicted"
description: "A cache hit rate that looks healthy can hide the requests where your agent harness rewrote its own prefix. How to find them, and why the provider is rarely the one to blame."
pubDatetime: 2026-10-21
tags: ["agents", "prompt-caching", "harnesses", "opencode", "codex", "claude-code"]
draft: false
---

In September I set out to make a harness I work on cache-friendly. The big change was freezing the system prompt for the life of a session, so nothing in it could shift mid-conversation.

Then I ran a full agent session against the OpenAI API and logged every request. 91% of input tokens came from cache. By the usual measure, that's healthy.

If you want the basics of prefix caching, [Kamran Tahir's survey](https://saybackend.com/blog/prompt-caching-agent-harnesses/) covers them well. This post is about what breaks it.

A hit rate hides a lot, because a request that appends a big tool result is mostly new tokens anyway. So I counted something narrower: tokens the API had already seen, billed again as new.

<figure>
<svg viewBox="0 0 640 250" role="img" aria-label="Tokens already sent once and billed again uncached, per request, across 26 requests in one session. 22 requests re-billed about zero. Four spikes: request 7, about 21k, old tool output rewritten; request 8, about 42k, mode switch with the tool list going from 47 to 42; request 20, about 6k, mode exit; request 25, about 62k, old tool output rewritten." style="width:100%;height:auto;font-family:inherit;font-size:11px">
  <g stroke="currentColor" stroke-opacity="0.15">
    <line x1="40" y1="40" x2="620" y2="40"/>
    <line x1="40" y1="82.9" x2="620" y2="82.9"/>
    <line x1="40" y1="125.7" x2="620" y2="125.7"/>
    <line x1="40" y1="168.6" x2="620" y2="168.6"/>
  </g>
  <line x1="40" y1="190" x2="620" y2="190" stroke="currentColor" stroke-opacity="0.5"/>
  <g fill="currentColor" fill-opacity="0.6" text-anchor="end">
    <text x="34" y="44">70k</text>
    <text x="34" y="86.9">50k</text>
    <text x="34" y="129.7">30k</text>
    <text x="34" y="172.6">10k</text>
    <text x="34" y="194">0</text>
  </g>
  <g fill="currentColor" fill-opacity="0.35">
    <rect x="44" y="188" width="14" height="2"/><rect x="66.3" y="188" width="14" height="2"/><rect x="88.6" y="188" width="14" height="2"/><rect x="110.9" y="188" width="14" height="2"/><rect x="133.2" y="188" width="14" height="2"/><rect x="155.5" y="188" width="14" height="2"/>
    <rect x="222.4" y="188" width="14" height="2"/><rect x="244.7" y="188" width="14" height="2"/><rect x="267" y="188" width="14" height="2"/><rect x="289.3" y="188" width="14" height="2"/><rect x="311.6" y="188" width="14" height="2"/><rect x="333.9" y="188" width="14" height="2"/><rect x="356.2" y="188" width="14" height="2"/><rect x="378.5" y="188" width="14" height="2"/><rect x="400.8" y="188" width="14" height="2"/><rect x="423.1" y="188" width="14" height="2"/><rect x="445.4" y="188" width="14" height="2"/>
    <rect x="490" y="188" width="14" height="2"/><rect x="512.3" y="188" width="14" height="2"/><rect x="534.6" y="188" width="14" height="2"/><rect x="556.9" y="188" width="14" height="2"/>
    <rect x="601.5" y="188" width="14" height="2"/>
  </g>
  <g fill="var(--accent, #ff6b01)">
    <rect x="177.8" y="146" width="14" height="44"/>
    <rect x="200.1" y="99.6" width="14" height="90.4"/>
    <rect x="467.7" y="176.1" width="14" height="13.9"/>
    <rect x="579.2" y="57.1" width="14" height="132.9"/>
  </g>
  <g fill="currentColor" font-size="10.5">
    <text x="214" y="88" text-anchor="middle">mode switch</text>
    <text x="214" y="76" text-anchor="middle" fill-opacity="0.65">tools 47→42</text>
    <text x="150" y="136" text-anchor="middle" fill-opacity="0.85">old tool output</text>
    <text x="150" y="124" text-anchor="middle" fill-opacity="0.65">rewritten</text>
    <text x="474" y="166" text-anchor="middle">mode exit</text>
    <text x="560" y="46" text-anchor="end">old tool output</text>
    <text x="560" y="34" text-anchor="end" fill-opacity="0.65">rewritten</text>
  </g>
  <g fill="currentColor" fill-opacity="0.6" text-anchor="middle">
    <text x="51" y="206">1</text>
    <text x="184.8" y="206">7</text>
    <text x="207" y="206">8</text>
    <text x="474.7" y="206">20</text>
    <text x="586" y="206">25</text>
    <text x="330" y="230">request (one session, 26 requests)</text>
  </g>
  <text x="40" y="16" fill="currentColor" fill-opacity="0.8">Tokens already sent once, billed again uncached</text>
</svg>
<figcaption>One agent's session, 82% cached overall. These four requests carry 52% of its uncached input.</figcaption>
</figure>

Most requests re-billed nothing. Four re-billed tens of thousands of tokens each, more than half of everything the session paid full price for.

The first big one was a mode switch. I'd made sure the instructions stayed the same when the agent switched modes. They did. The cache still missed completely.

## The tools are part of the prefix

The [common advice](https://x.com/ericzakariasson/status/2102853511637774551) is to keep the prefix identical across turns. I thought I had. But I'd been treating "the prefix" as the system prompt. To the provider, it's everything before the conversation, and tool definitions come first.

<blockquote class="twitter-tweet" data-dnt="true"><p lang="en" dir="ltr">here&#39;s a prompt to improve your agent harness based on what we&#39;ve learned at cursor. enjoy</p>&mdash; Eric Zakariasson (@ericzakariasson) <a href="https://twitter.com/ericzakariasson/status/2102853511637774551">September 23, 2026</a></blockquote>

The switch changed two things: it trimmed the tool list from 47 to 42, and it added a one-line note announcing the new mode.

On the next run I dropped the note. Everything else was byte-identical across the switch. Still a full miss. The tool list alone was enough.

Hiding tools per mode felt like the obvious way to scope an agent. It cost the whole cache on every switch.

The fix was to stop hiding them. Every mode now sends the same tool list, and a tool the mode doesn't allow just refuses when called. The switch went from 0% cached to 91%.

[opencode](https://github.com/anomalyco/opencode) landed on the same rule. Its [plan mode](https://github.com/anomalyco/opencode/blob/681156c82fcdd02a64dee0e81967ea35e5c8b886/packages/core/src/plugin/plan.ts#L39-L53) keeps the edit tool listed and rejects the edits it doesn't allow. Its [docs](https://github.com/anomalyco/opencode/blob/681156c82fcdd02a64dee0e81967ea35e5c8b886/packages/core/src/tool/AGENTS.md?plain=1#L54) put it well: "Tool filtering is catalog visibility, not execution authorization."

## Don't edit what you've already sent

The small spike was the agent switching back. The other two had nothing to do with modes. The tools and prompt were untouched. The history wasn't.

In one, a tool output the model had seen at 59,651 characters came back on the next turn as 828. The cache served everything before it and nothing after.

The other was the same bug in reverse: a 1,608-character output came back as 51,814, because the stored copy held more than the model was sent. Together they re-billed twice what the mode switch did.

Neither was a caching decision. Both came from code that tidied up history, or filled it in, after the fact. Each version was reasonable. They just weren't the same version.

This is the whole problem in one picture. The provider's cache is an append-only log of what you sent. Append, and you pay for the new part. Edit anything already in it, however small or far back, and you pay again for everything after.

The fix is to render each tool output once and replay exactly that.

[Codex](https://github.com/openai/codex) truncates output once, when it's recorded; [its source](https://github.com/openai/codex/blob/c419fd01a70dd1d05356da443ac44328bda2e6f4/codex-rs/core/src/context_manager/history.rs#L518) notes the truncation "applies only to live history." opencode [dropped its option](https://github.com/anomalyco/opencode/blob/681156c82fcdd02a64dee0e81967ea35e5c8b886/packages/core/src/config/normalize.ts#L353) to prune old outputs and now [caps them once](https://github.com/anomalyco/opencode/blob/681156c82fcdd02a64dee0e81967ea35e5c8b886/packages/core/src/tool-output.ts#L13-L14), when produced.

## Other ways to edit the past

Once I knew the shape, I found more. None showed up in that session. All of them showed up in production.

**Tool order that depends on timing.** MCP servers connect in parallel, and the harness listed their tools in whatever order the connections finished. In production the order changed 14 times in 3 days, sometimes mid-session.

Same tools, different order, full miss. The fix is to sort. Codex connects in parallel too, then [sorts](https://github.com/openai/codex/blob/c419fd01a70dd1d05356da443ac44328bda2e6f4/codex-rs/codex-mcp/src/tools.rs#L197). opencode [sorts](https://github.com/anomalyco/opencode/blob/681156c82fcdd02a64dee0e81967ea35e5c8b886/packages/core/src/mcp/index.ts#L632-L667) and [waits](https://github.com/anomalyco/opencode/blob/681156c82fcdd02a64dee0e81967ea35e5c8b886/packages/core/src/session/context.ts#L132) for the first connections before sending anything.

**Replay that tells the story differently.** The agent called a tool, read the output, then called another. On replay, the harness merged those into one step: two calls, then two outputs.

Same content, different shape, and every new turn or resume diverged there. The fix was to record which step each part belonged to and replay the steps as they happened.

**Instructions that refresh themselves.** The harness re-read AGENTS.md and git status every turn, so a commit or a doc edit rewrote the system prompt mid-session.

Freezing them fixed the cache and created a new bug: a guide written after the freeze never reached the model.

Codex and opencode both keep the original and [append the change](https://github.com/anomalyco/opencode/blob/681156c82fcdd02a64dee0e81967ea35e5c8b886/packages/core/src/session/instruction-state.ts#L38-L61) as a new message. [Codex's](https://github.com/openai/codex/blob/c419fd01a70dd1d05356da443ac44328bda2e6f4/codex-rs/core/src/context/world_state/agents_md.rs#L11-L12) begins: "These AGENTS.md instructions replace all previously provided AGENTS.md instructions."

## Finding your own

Usage numbers tell you that a request missed, not why. For that you need the request bodies, so the first thing I do in any harness I work on is log them in full.

For every request that cached less than it should, I compare it with the one before. I flatten each body into prefix order (tools, instructions, then each conversation item), hash each part, and find the first that differs.

```ts
import { createHash } from "node:crypto";

const h = (v: unknown) =>
  createHash("sha1").update(JSON.stringify(v)).digest("hex").slice(0, 8);

// A Responses API request, in prefix order.
function parts(body: any): string[] {
  return [
    `tools ${h(body.tools ?? [])}`,
    `instructions ${h(body.instructions ?? "")}`,
    ...(body.input ?? []).map(
      (item: any, i: number) =>
        `input[${i}] ${item.role ?? item.type} ${h(item)}`
    ),
  ];
}

function firstDivergence(prev: any, next: any): string {
  const a = parts(prev);
  const b = parts(next);
  const i = a.findIndex((p, k) => p !== b[k]);
  return i === -1
    ? "prefix intact, only appended"
    : `${a[i]} -> ${b[i] ?? "missing"}`;
}
```

The answer lands in one of three buckets. If `tools` or `instructions` differ, the harness changed its prefix. If an `input[i]` item differs, usually an old tool output, it rewrote history.

If the prefix is intact and the request still missed, it's the provider, or the cache expired.

Most of the misses in this post were found with a version of that function. It takes minutes to write. Guessing took me much longer.

Then make it a test. Codex has [one](https://github.com/openai/codex/blob/c419fd01a70dd1d05356da443ac44328bda2e6f4/codex-rs/core/tests/suite/prompt_caching.rs#L473) that changes approval and sandbox settings mid-session and asserts the next request is the previous one plus appended items. If your harness can't pass that, your invoice will tell you.

## It's usually you

To check this wasn't just my harness, I ran the same analysis on several weeks of my own Claude Code and Codex logs, about 59,000 requests. One person's usage, mostly agent-driven runs, but a consistent shape.

Claude Code cached 97% of input tokens; Codex cached 93%. In both, 3 to 4% of requests carried more than half of what was avoidable. The averages looked fine. The cost sat in a handful of transitions.

The provider does miss. In Codex, about 0.6% of steps within a turn missed with nothing changed and no time passed, and the next request usually hit again. Real, but about one step in 170.

Most of the rest came from the harness's own choices. In Codex, misses clustered around compaction. In Claude Code, it was time: requests after the cache expired, like a subagent waiting on a slow tool or a session idle over lunch.

OpenAI has since made 12-hour caching available on its API. That helps with time. It does nothing for a harness that rewrites its own prefix: twelve hours of cache can't match a prefix you no longer send.

<blockquote class="twitter-tweet" data-dnt="true"><p lang="en" dir="ltr">You can now choose 12-hour prompt caching in the API.</p>&mdash; OpenAI Developers (@OpenAIDevs) <a href="https://twitter.com/OpenAIDevs/status/2112603916076781778">October 20, 2026</a></blockquote>

<script>
  document.querySelectorAll("blockquote.twitter-tweet").forEach((el) =>
    el.setAttribute("data-theme", document.documentElement.dataset.theme === "dark" ? "dark" : "light"),
  );
</script>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

The cache doesn't miss at random. It misses where your harness changed its mind about the past.
