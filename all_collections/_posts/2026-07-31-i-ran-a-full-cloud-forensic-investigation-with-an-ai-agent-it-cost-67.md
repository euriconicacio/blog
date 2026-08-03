---
layout: post
title: I ran a full cloud forensic investigation with an AI agent. It cost $67.
description: And 99% of that bill was the model re-reading context it had already been given. If you budget for this work by counting what the model writes, you will be wrong by roughly 7×.
date: "2026-07-31 09:00:00 -0300"
image: /assets/images/posts/i-ran-a-full-cloud-forensic-investigation-with-an-ai-agent-it-cost-67/01.png
---
![I ran a full cloud forensic investigation with an AI agent. It cost $67.](/assets/images/posts/i-ran-a-full-cloud-forensic-investigation-with-an-ai-agent-it-cost-67/01.png)

> ⚠️ Disclosure — please read this first
>
> **This work was performed under an independent consulting engagement, in a client environment that has no connection to any current or former employer of mine.** Nothing here describes, references, or is drawn from any employer’s infrastructure, incidents, or data.
>
> The token counts, timings and costs below are mine — they measure my own tooling, not the client’s. Every client-identifying detail has been removed or replaced, as in the companion post.

In [the previous post](https://euriconicacio.medium.com/cloud-forensics-in-the-age-of-ai-agents-a-field-report-461851da7708) I described the method: an agent-assisted incident response on a compromised AWS credential — 3,122 API calls reconstructed twice from independent sources, 298,196 raw log events parsed from scratch as a cross-check, 17,776 access logs downloaded on a hunch that turned out to reverse a published conclusion, a controlled authorization experiment, and the writing of the response that closed the case.

That post was about how the work was done, and about the three times the machine was confidently wrong.

This one is the invoice.

I have not seen many honest accountings of what agent-assisted work actually costs — where the money goes, what it displaces, and what it quietly adds back. So here is mine, with the measurement method included so you can run it against your own history.

## What I measured, and how

Everything below comes from one session transcript — a JSONL file where every model response records a `usage` object. No estimation, no sampling. Four fields matter:

![I ran a full cloud forensic investigation with an AI agent. It cost $67.](/assets/images/posts/i-ran-a-full-cloud-forensic-investigation-with-an-ai-agent-it-cost-67/02.png)

Sum them across the session and apply your provider’s rates. The script is at the end of this post; it is about fifteen lines.

Two caveats stated up front, because they bound everything that follows.

**First, prices change.** The rates I use are the list prices in effect when I measured: **$5 per million input tokens, $25 per million output**, with cached reads at **0.1×** the input rate and cache writes at **1.25×** (five-minute TTL) or **2×** (one hour). Anything you compute from this post should be recomputed against today’s numbers.

**Second, this covers one session.** There was earlier work on this incident, in a separate session I did not instrument. The figures below are the main investigation only, and they are therefore a floor, not a total. I would rather publish a number I can defend than a bigger one I can’t.

## The session

![I ran a full cloud forensic investigation with an AI agent. It cost $67.](/assets/images/posts/i-ran-a-full-cloud-forensic-investigation-with-an-ai-agent-it-cost-67/03.png)

Fourteen messages. That number matters later.

## The bill

![I ran a full cloud forensic investigation with an AI agent. It cost $67.](/assets/images/posts/i-ran-a-full-cloud-forensic-investigation-with-an-ai-agent-it-cost-67/04.png)

Sixty-seven dollars for the forensic reconstruction of a twenty-day intrusion across seventeen regions, twice, from independent sources.

That’s the headline, and it is the least interesting number in the post.

## Ninety-nine percent of it was re-reading

Look at the input column again.

- Uncached input: **928 tokens**
- Cache reads: **97,002,227 tokens**

**99.1% of all input was cache reads.** The model was fed roughly 98 million tokens of context over 464 turns, and 928 of them were new.

That is not a quirk of this session. It’s the arithmetic of how these systems work. Every turn resends the conversation: the system prompt, the tool definitions, every prior tool call, every prior result. A 464-turn investigation with large tool outputs — log dumps, CSV excerpts, JSON blobs — accumulates a context that gets re-transmitted on every single turn. The volume of *re-reading* dwarfs the volume of *writing* by two orders of magnitude.

Which produces the number that should change how you budget:

> **Without prompt caching, the same session costs $502 instead of $67.**

97.9 M input tokens at full price is $489. Add the $13 of output and you are at roughly **$502** — a **7.4×** multiplier, paid for nothing but reading the same bytes again.

Two consequences follow, and both are load-bearing.

**Budget by input, not by output.** The instinct is to estimate agent cost from how much it produces — the reports, the code, the analysis. On this workload, output was **$13 of a $67 bill: 20%**. If you scope a project by imagining how much the model will write, you will be wrong by roughly 5×. Estimate the context that gets re-read instead.

**Caching is not an optimization here; it’s the business case.** At $502 a run, “let the agent parse 300,000 raw events as a cross-check” is a decision you’d think about. At $67 it isn’t a decision at all. The entire argument of the previous post — that marginal hypotheses become worth testing — depends on that 7.4×. If your provider or configuration isn’t caching effectively, you are not running the workload I described. You’re running a much more expensive one that will push you back toward the human triage-by-budget habits.

Forensics has an unusually bad profile for this. Log lines are long, poorly compressible, and mostly semantically identical to each other. A single day of raw trail parsing pushes a lot of near-duplicate text through context. If you’re going to run this kind of work, verify your cache hit rate before you scale it, not after the invoice.

## What it displaced

Now the other side. What would this have cost in human time?

This part is an **estimate**, not a measurement, and I’m labelling it as such. It’s my honest read on how long these phases take a competent cloud security engineer with access already in place and no interruptions.

**To be explicit about the baseline, because this is where this kind of comparison usually cheats: the human writes scripts too.** Nobody reads 298,196 log events, and nobody greps 17,776 files by hand. They write a paginator loop, a gzip parser, a field-offset parser for a log format they haven’t parsed before, a billing query. Those are exactly the artifacts the agent produced — same language, same libraries, often the same approach. The hours below are authoring, debugging and running time, not reading time.

![I ran a full cloud forensic investigation with an AI agent. It cost $67.](/assets/images/posts/i-ran-a-full-cloud-forensic-investigation-with-an-ai-agent-it-cost-67/05.png)

I’m not converting that into money. I don’t know what an hour of your team’s time costs, and inventing a rate would weaken the comparison rather than strengthen it. Price it yourself.

So if both sides write scripts, what actually differs? Three things, and only three:

**Authoring latency per throwaway script.** Not whether the script gets written, but how long the gap is between wanting it and having output. A field-offset parser for an unfamiliar log format is twenty minutes of reading a spec and fixing off-by-ones. That gap is where the decision “is this worth it?” lives, and shortening it is what changes the decision.

**Parallelism without context-switching cost.** Three collections ran in the background while I worked on the billing analysis. A person can technically do that too — and then pays for it in the re-orientation tax every time they come back to a half-finished parser.

**Willingness to write the sixth one.** By the fifth throwaway script of a long day, the marginal hunch stops feeling worth a new file. That’s the mechanism that killed phases 2 and 5 in the conventional version — not incapacity, fatigue and budget.

And the counterweight, which belongs in the same breath: **the agent’s scripts needed debugging exactly like a person’s.** Its first collection pass silently truncated at fifty results and reported success. Its background monitors deadlocked on a `pgrep` that matched their own command line. Neither is a mistake a careful engineer wouldn't also make at 3 a.m. — the difference is that fixing them cost a sentence instead of a context switch.

## Why “2.4 hours versus 6 working days” is the wrong comparison

It’s the number everyone reaches for, and it’s misleading in a specific way.

**A human working this incident under a deadline does not do phases 2 and 5.** Downloading 35,000 log files to independently re-derive a number you already have, and downloading 17,776 access logs on a hunch about one bucket, are exactly the tasks that get cut when time is the constraint. They’re not sloppy to cut. They’re arithmetic.

And those two phases are where the value was:

- Phase 2 produced the independent cross-validation. Without it, every number in the report is one tool’s unaudited opinion.
- Phase 5 surfaced the data-plane activity that the control-plane logs structurally could not see — and **forced a correction to a conclusion I had already written down**.

So the realistic human version isn’t six days of the same work. It’s one or two days producing a subset — and stating, with no way to know it, that the attacker never reached the audit-log bucket. That claim would have happened to be close enough: they listed it and read nothing. But it would have been asserted without evidence, on a bucket where the stolen credential could have deleted everything.

The honest framing:

> The gain was not doing the same work faster. It was that work previously cut for budget became cheap enough to do — and one of those cuts was load-bearing.

That’s a smaller claim than the usual pitch, and it’s the one the evidence supports.

## The costs that don’t show up on the invoice

If the post stopped here it would be an advertisement. It shouldn’t, because this run generated real costs that a token counter never sees.

**Verification time — the big one.** Every number the agent produced needed confirmation from a second source before it could go in a report. That verification is human time, it doesn’t shrink as generation gets faster, and it is not optional: an unverified number in a forensic report has *negative* value, because it will be believed. The $67 buys you generation. It does not buy you the right to trust the output.

**The fourteen human messages were load-bearing.** Not oversight, not approvals — actual contributions:

- One question — *“and if no applications existed, would it still return empty?”* — exposed that my authorization test had covered one quadrant of four.
- One question — *“are those background tasks still running?”* — surfaced five processes stuck in an infinite loop.
- One directive on disclosure scope removed eight items from an external communication that nobody had asked for.

Fourteen messages across two and a half hours is roughly one every ten minutes. That’s the actual human cost of “autonomous” work on something consequential, and any planning that assumes fire-and-forget is planning for a different task than this one.

**Rework from concluding early.** Generation is cheap, so conclusions land early — and when one is corrected, the correction has to be chased through every document already written from it. The data-plane correction propagated through two finished reports. Fast generation means fast propagation of error; budget for it, or write the conclusions last.

**Compute spent on nothing.** Five background monitors ran in an infinite loop for most of the session. The cause was a one-line bug the agent wrote and I didn’t catch: each monitor waited on `pgrep -f "fetch-ct.sh"` returning nothing, and `pgrep` matched *the monitor's own command line*, because the string was right there in it. Five processes, each waiting forever for itself to exit. Zero output, real consumption, and the only reason it was caught is that a human asked what was still running.

**An hour of work the documentation had already answered.** The authorization experiment was well-designed, controlled, and conclusive — and the vendor’s own API reference stated the behavior in plain language the whole time. When building the test rig gets cheap enough, running it becomes easier than looking it up. That’s a new failure mode, and it has a line item.

## When this doesn’t pay

Three cases where I would not run this pattern:

**Small, well-specified tasks.** The overhead — setup, verification, the human attention tax — doesn’t amortize. If you can do it in twenty minutes, do it in twenty minutes.

**Environments with no independent telemetry.** The entire method rests on cross-validation. In an environment where every number comes from one source, an agent produces confident output you have no way to check, faster than before. That is worse than not having it.

**Teams without someone who knows which number to check.** This is the real constraint. The agent gave me a count of 3,122 events. Knowing that this particular API silently truncates pagination, that the raw log files are a genuinely independent path, and that agreement between those two is what makes the number defensible — none of that came from the tool. Someone who doesn’t already know that gets all of the speed and none of the safety.

## Measure your own

Here’s the script. Point it at a session transcript and it prints the same table:

```python
import json, sys

RATES = {           # $ per million tokens — update to current list prices
    "in": 5.00, "out": 25.00,
    "cache_read": 0.50,   # 0.1x input
    "cache_write": 6.25,  # 1.25x input (5-minute TTL)
}
t = {"in": 0, "out": 0, "cache_read": 0, "cache_write": 0}

for line in open(sys.argv[1]):
    u = (json.loads(line).get("message") or {}).get("usage") or {}
    t["in"]          += u.get("input_tokens", 0)
    t["out"]         += u.get("output_tokens", 0)
    t["cache_read"]  += u.get("cache_read_input_tokens", 0)
    t["cache_write"] += u.get("cache_creation_input_tokens", 0)

total = sum(t[k] / 1e6 * RATES[k] for k in t)
for k in t:
    print(f"{k:12} {t[k]:>12,}  ${t[k]/1e6*RATES[k]:>8.2f}")
print(f"{'TOTAL':12} {'':>12}  ${total:>8.2f}")
print(f"cache read = {t['cache_read']/max(1,t['cache_read']+t['in']+t['cache_write']):.1%} of input")
```

Run it against a session you already consider expensive. If that last line prints something in the high nineties, your bill is a re-reading bill, and the lever is caching — not shorter outputs, not a cheaper model.

## What actually changed

Two and a half hours, sixty-seven dollars, and roughly four to six days of specialist work that didn’t have to happen — set against fourteen human interventions that did, verification that doesn’t compress, and an hour spent rediscovering something the documentation already said.

The bottleneck moved. It used to be execution: could you get the data, parse it, cross-check it in the time available. Now it’s verification: can you tell whether what came back is true.

Verification doesn’t scale with tokens. It scales with whether someone on the team knows which number to check — and that is the part you still have to hire for.

*Details of the environment and the organization have been changed or removed. The token counts, timings, and costs are unmodified. Prices are list prices as of the measurement and will have moved.*
