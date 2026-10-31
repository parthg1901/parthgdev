---
title: "Stop fixing your agent in the prompt"
description: "A prompt patch after one bad run bets twice: that the failure will repeat, and that the prompt caused it. Why both bets usually lose, and what to do instead."
pubDatetime: 2026-10-31
tags: ["agents", "evals", "prompting", "llm"]
draft: false
---

When an agent misbehaves, the easy fix is a line in the prompt. It's tempting because it's easy, and the next run usually looks fixed.

But present LLMs are quite good at working around things. If the prompt is clean, the wrong behaviour is probably coming from somewhere else, and the patch only hides the symptom.

Worse, the patch doesn't stay where you put it. It adds a step the model has to think about on every request, even the ones that never needed it, and the agent turns over-cautious in places that were fine before.

A prompt patch after one bad run is a bet twice over: that the failure will happen again, and that the prompt is where it came from. Both bets usually lose.

## One failure is a sample

With deterministic code, a failing test is a fact. Run it again and it fails again. You fix the code, the test passes, and that's the end of it.

An agent doesn't work like that. Run the same prompt twice and you get two different programs, not two phrasings of the same one. One run adds a rate limit and the other doesn't. Both look reasonable, and both pass a quick read.

So a test asserts, but an eval estimates. One run is one sample from a distribution, and the question isn't "did it fail?" but "how often, and how badly?"

That changes what a fix means. Before you touch anything, run the failing case again, several times. A surprising number of failures don't come back. You'd have been patching a one-off.

It also means the run after your patch proves nothing. It was going to pass most of the time anyway. Whether the fix worked is a question about a batch of runs, not the next one.

## The cause is rarely in the prompt

When a failure does repeat, it's tempting to assume the model misunderstood the instructions. Usually it understood them fine. It was working with something else that was wrong.

Some of the usual sources:

- **A tool that invites misuse.** A vague parameter, a confusing name, or an error message that suggests the wrong next step. The model does what the tool seems to want.
- **Missing context.** The agent guessed because nothing told it. The guess was reasonable; it was just wrong.
- **A bad default.** A template, a scaffold or a config value it inherited and had no reason to question.
- **The harness.** Stale state, a truncated tool output, or instructions that changed mid-session without the model being told.

None of these is fixed by telling the model to behave. The patch makes this symptom go away while the cause keeps producing others.

The quickest way to find the cause is to look at what the agent saw, not what it did. Read the trace around the failure.

Then ask the agent one follow-up question about why it made that choice. Agents are often honest one question deep; the answer tends to point at the real source.

## Patches leak

A prompt line doesn't apply to the case you wrote it for. It applies to every request, forever.

I once watched a fix go wrong exactly this way.

An agent was supposed to ask the user a few questions through a dedicated question tool. Half the time it asked them in plain text instead, or announced them and stopped.

So a line went in: every question goes through the tool, and never announce one you don't ask. The plain-text questions disappeared. In four of the next ten runs, the agent asked nothing at all, though the task required it.

<figure>
<svg viewBox="0 0 640 210" role="img" aria-label="Before a prompt patch, 6 runs: 3 asked their questions properly, 3 asked the wrong way, which was the bug being fixed. After the patch, 10 runs: 6 asked properly and 4 asked nothing at all. The bug was gone and a new failure appeared." style="width:100%;height:auto;font-family:inherit;font-size:12px">
  <text x="20" y="22" fill="currentColor" fill-opacity="0.8">Same task, before and after one prompt line</text>
  <text x="20" y="74" fill="currentColor">Before the patch</text>
  <text x="20" y="90" fill="currentColor" fill-opacity="0.6">6 runs</text>
  <text x="20" y="144" fill="currentColor">After the patch</text>
  <text x="20" y="160" fill="currentColor" fill-opacity="0.6">10 runs</text>
  <g>
    <circle cx="230" cy="78" r="11" fill="currentColor" fill-opacity="0.75"/>
    <circle cx="268" cy="78" r="11" fill="currentColor" fill-opacity="0.75"/>
    <circle cx="306" cy="78" r="11" fill="currentColor" fill-opacity="0.75"/>
    <circle cx="344" cy="78" r="10" fill="none" stroke="currentColor" stroke-opacity="0.75" stroke-width="2" stroke-dasharray="4 3"/>
    <circle cx="382" cy="78" r="10" fill="none" stroke="currentColor" stroke-opacity="0.75" stroke-width="2" stroke-dasharray="4 3"/>
    <circle cx="420" cy="78" r="10" fill="none" stroke="currentColor" stroke-opacity="0.75" stroke-width="2" stroke-dasharray="4 3"/>
    <circle cx="230" cy="148" r="11" fill="currentColor" fill-opacity="0.75"/>
    <circle cx="268" cy="148" r="11" fill="currentColor" fill-opacity="0.75"/>
    <circle cx="306" cy="148" r="11" fill="currentColor" fill-opacity="0.75"/>
    <circle cx="344" cy="148" r="11" fill="currentColor" fill-opacity="0.75"/>
    <circle cx="382" cy="148" r="11" fill="currentColor" fill-opacity="0.75"/>
    <circle cx="420" cy="148" r="11" fill="currentColor" fill-opacity="0.75"/>
    <text x="458" y="154" text-anchor="middle" font-size="22" font-weight="700" fill="var(--accent, #ff6b01)">×</text>
    <text x="496" y="154" text-anchor="middle" font-size="22" font-weight="700" fill="var(--accent, #ff6b01)">×</text>
    <text x="534" y="154" text-anchor="middle" font-size="22" font-weight="700" fill="var(--accent, #ff6b01)">×</text>
    <text x="572" y="154" text-anchor="middle" font-size="22" font-weight="700" fill="var(--accent, #ff6b01)">×</text>
  </g>
  <g font-size="11.5" fill="currentColor" fill-opacity="0.75">
    <circle cx="26" cy="194" r="6" fill="currentColor" fill-opacity="0.75"/><text x="38" y="198">asked properly</text>
    <circle cx="156" cy="194" r="5.5" fill="none" stroke="currentColor" stroke-width="1.6" stroke-dasharray="3 2.5"/><text x="168" y="198">asked the wrong way (the bug)</text>
    <text x="402" y="200" text-anchor="middle" font-size="17" font-weight="700" fill="var(--accent, #ff6b01)" fill-opacity="1">×</text><text x="414" y="198">didn't ask at all (new)</text>
  </g>
</svg>
<figcaption>The patch removed the symptom. Four runs in ten stopped asking anything.</figcaption>
</figure>

The patch worked on the symptom and taught the wrong lesson. That's the usual pattern. Rules get applied too broadly or quietly ignored; reasons generalize.

Each patch also costs attention. The model has to consider it on every turn, even when it's irrelevant. A prompt that has grown one patch at a time ends up full of special cases the model is always thinking about.

## When the prompt is the right fix

None of this means never touch the prompt. Sometimes the prompt really is the root cause: the model was missing information it couldn't have known.

The test is what you're adding. Context and reasons are fine. If the agent didn't know a constraint existed, tell it, and say why. A model that understands the reason applies it sensibly.

Rules that force behaviour are the ones that leak. And if a behaviour must always hold, a prompt is the wrong place for it anyway. Put it in code, as a guard or a check the agent can't talk its way around.

## What to do instead

When an agent fails, the order matters more than the fix:

<figure>
<svg viewBox="0 0 640 310" role="img" aria-label="When an agent fails: reproduce it by running several times, find what the agent saw, then sort it. Wrong outcome with real cost gets a guard in code. Suboptimal but valid gets left alone. A harness problem gets fixed in the harness. Missing information goes in the prompt as context with a reason. Then check the fix on runs you didn't tune on." style="width:100%;height:auto;font-family:inherit;font-size:13px">
  <defs><marker id="ah" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0 0L10 5L0 10z" fill="currentColor" fill-opacity="0.6"/></marker></defs>
  <rect x="220" y="10" width="200" height="40" rx="6" fill="none" stroke="currentColor" stroke-opacity="0.55" stroke-width="1.5"/><text x="320.0" y="34.0" text-anchor="middle" fill="currentColor">Agent fails</text><line x1="320" y1="50" x2="320" y2="68" stroke="currentColor" stroke-opacity="0.5" stroke-width="1.5" marker-end="url(#ah)"/><rect x="200" y="70" width="240" height="46" rx="6" fill="none" stroke="currentColor" stroke-opacity="0.55" stroke-width="1.5"/><text x="320.0" y="90.0" text-anchor="middle" fill="currentColor">Reproduce</text><text x="320.0" y="106.0" text-anchor="middle" fill="currentColor" fill-opacity="0.65" font-size="11">run it several times</text><line x1="320" y1="116" x2="320" y2="134" stroke="currentColor" stroke-opacity="0.5" stroke-width="1.5" marker-end="url(#ah)"/><rect x="200" y="136" width="240" height="46" rx="6" fill="none" stroke="currentColor" stroke-opacity="0.55" stroke-width="1.5"/><text x="320.0" y="156.0" text-anchor="middle" fill="currentColor">Find what it saw</text><text x="320.0" y="172.0" text-anchor="middle" fill="currentColor" fill-opacity="0.65" font-size="11">trace, tools, context, defaults</text><line x1="85" y1="202" x2="556" y2="202" stroke="currentColor" stroke-opacity="0.5" stroke-width="1.5"/><line x1="320" y1="182" x2="320" y2="202" stroke="currentColor" stroke-opacity="0.5" stroke-width="1.5"/><line x1="84" y1="202" x2="84" y2="220" stroke="currentColor" stroke-opacity="0.5" stroke-width="1.5" marker-end="url(#ah)"/><rect x="10" y="222" width="148" height="46" rx="6" fill="none" stroke="currentColor" stroke-opacity="0.55" stroke-width="1.5"/><text x="84.0" y="242.0" text-anchor="middle" fill="currentColor">Wrong outcome</text><text x="84.0" y="258.0" text-anchor="middle" fill="currentColor" fill-opacity="0.65" font-size="11">→ guard in code</text><line x1="241" y1="202" x2="241" y2="220" stroke="currentColor" stroke-opacity="0.5" stroke-width="1.5" marker-end="url(#ah)"/><rect x="167" y="222" width="148" height="46" rx="6" fill="none" stroke="currentColor" stroke-opacity="0.55" stroke-width="1.5"/><text x="241.0" y="242.0" text-anchor="middle" fill="currentColor">Suboptimal, valid</text><text x="241.0" y="258.0" text-anchor="middle" fill="currentColor" fill-opacity="0.65" font-size="11">→ leave it</text><line x1="398" y1="202" x2="398" y2="220" stroke="currentColor" stroke-opacity="0.5" stroke-width="1.5" marker-end="url(#ah)"/><rect x="324" y="222" width="148" height="46" rx="6" fill="none" stroke="currentColor" stroke-opacity="0.55" stroke-width="1.5"/><text x="398.0" y="242.0" text-anchor="middle" fill="currentColor">Harness problem</text><text x="398.0" y="258.0" text-anchor="middle" fill="currentColor" fill-opacity="0.65" font-size="11">→ fix the harness</text><line x1="555" y1="202" x2="555" y2="220" stroke="currentColor" stroke-opacity="0.5" stroke-width="1.5" marker-end="url(#ah)"/><rect x="481" y="222" width="148" height="46" rx="6" fill="none" stroke="var(--accent, #ff6b01)" stroke-opacity="1" stroke-width="1.5"/><text x="555.0" y="242.0" text-anchor="middle" fill="currentColor">Missing info</text><text x="555.0" y="258.0" text-anchor="middle" fill="currentColor" fill-opacity="0.65" font-size="11">→ context + reason</text><text x="320" y="296" text-anchor="middle" fill="currentColor" fill-opacity="0.75">Then check the fix on runs you didn't tune on.</text>
</svg>
<figcaption>Only one of the four exits touches the prompt.</figcaption>
</figure>

1. **Reproduce it.** Run the case several times. If it doesn't come back, it may not be worth fixing.
2. **Find what the agent saw.** Read the trace and ask it why. Look at tools, context, defaults and harness before the prompt.
3. **Sort it.** A wrong outcome with real cost gets a mechanical guard. A suboptimal but valid choice gets left alone. A harness problem gets fixed in the harness. Missing information goes in the prompt, with the reason.
4. **Check it on runs you didn't tune on.** A fix that only works on the case that prompted it is a patch.

The same rules apply to your evaluator. LLM judges are non-deterministic too: they score the same output differently across runs and miss failures. Check them against cases where you already know the answer before trusting their numbers.

A prompt patch makes the failure you saw go away. The job is to make its cause go away.
