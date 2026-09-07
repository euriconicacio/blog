---
layout: post
title: "pwnloop, fifty-one machines"
description: A month after the first root, the autonomous loop has taken fifty-one lab machines from an IP to root and cleared a Pro Lab the platform itself certified. The count is the proof. The methodology the runs wrote is the point — and it is why the loop breaks business logic, the one thing it is supposed to be unable to break.
date: "2026-09-06 09:00:00 -0300"
---
> ***Disclaimer.*** *This is a personal open-source project, built and run on my own equipment and my own accounts. It has no connection to any current or former employer, and no client, employer or production environment was involved at any point. Every single-host target was a Hack The Box machine, and the campaign target was a Hack The Box Pro Lab, all engaged over the platform's own VPN and within its terms of service. No flag values appear in this post or in the repository. No chain is published for any machine not confirmed retired, and no Pro Lab chain, hostname or credential appears anywhere — flag sharing is a platform violation regardless of a machine's status, and Pro Labs never retire.*

When I wrote [the one-week post]({{ site.baseurl }}/pwnloop-one-week-in/), `pwnloop` had rooted seventeen machines and I said the completion count was the least interesting row in the table. I still think that. Here is the plain number anyway, because it is the thing people argue about:

**In thirty-six days the loop has taken fifty-one machines from an IP address to root, and cleared a Pro Lab the platform certified at 100%.**

The earlier posts cover what the tool is — [the single-host loop]({{ site.baseurl }}/pwnloop-an-autonomous-engagement-loop-for-lab-machines/), [lab mode against a Pro Lab]({{ site.baseurl }}/pwnloop-lab-mode-a-pro-lab-in-four-and-a-half-hours/), and [where it stood after a week]({{ site.baseurl }}/pwnloop-one-week-in/). This one does not repeat them. It is about the part I care about and undersold before: the fifty-one machines are proof, not product. **The product is the methodology the runs wrote, and the methodology is the reason the loop does the thing an agent is supposed to be unable to do — reason about a bespoke system and break its business logic.** That is most of this post. The machine count is the receipt stapled to the front.

## Where it is

| | day 8 | day 36 |
|---|---|---|
| Machines | 17 rooted | 51 rooted |
| Difficulty | up the ladder | Easy → Insane |
| Surfaces | Linux · Windows / AD | Linux · Windows · AD · Kubernetes · OT · 802.11 wireless |
| **Methodology** | **26 files, ~4,000 lines** | **25 files, ~6,200 lines** |
| Pro Lab | 1 · 3/3 hosts | 1 · 3/3 hosts · platform-certified |
| Releases | v1.9.0 | v1.19.0, 29 tags |

The bold row is the one to read. Everything else is an outcome; that row is the machine that produces outcomes. Six thousand lines of methodology now exist that did not a month ago, and not one line of it was written up front by me. It is entirely the residue of engagements — what each run had to work out on target, distilled into something the next run reads before it starts.

And before anyone reads the thirty-six days as *slow*: those are calendar days, not working days. I run this in spare hours — evenings, weekends, and whenever there are leftover tokens sitting around doing nothing. No sprint, no cleared calendar, no team. Fifty-one machines is what a side project did in the gaps of a normal month. The wall-clock is a measure of my free time, not the loop's throughput; point it at a queue and walk away and the limiter is tokens and target availability, not attention.

## The method is the whole thing

Strip away the container and the tooling and pwnloop is one idea: a **loop that is not allowed to close until it has made itself better at the next one.**

The inner loop is an engagement — recon, enumeration, foothold, privilege escalation, cleanup, report, without a human between the phases. That part is table stakes; plenty of things can drive a scanner. The outer loop is the part that matters. When a box falls, the run is *required* to write back into its own shared reference set whatever generalised: a check that paid off, a primitive worth trying early, an environment gotcha, a mistake that a rule would have prevented. It cannot mark the engagement done until it has. So every run leaves the tool different from how it found it, and the difference is always the same shape — something the run had to reason out on target is now on the page, and the next run reads it instead of reasoning it out again.

That is why the reference set is the number I watch. It is not documentation I maintain; it is *compiled experience*, and it compounds. The seventeenth box paid for a lesson the thirtieth read for free. The Active Directory reference grew a whole section on delegation-definition rights because one box turned on a low-privileged group holding the right to *mark* an account trusted for delegation, and the run that untangled it wrote the section so no future run has to. The wireless reference exists at all because one target could only be reached over the air, there was nothing written, and the run that solved it left the section behind. Nobody sat down and authored six thousand lines of tradecraft. Fifty-one engagements did, one hard-won paragraph at a time.

Three rules make this more than a scraper that got lucky, and they are worth stating exactly, because they are also — not coincidentally — the rules that make the loop able to break business logic:

- **Every action must trace to evidence it already collected.** Before the loop runs a command, it has to be able to name the file and the line of output that motivated it. "This kind of box usually has X" is explicitly *not* a reason to act — it is a reason to go collect the observation that confirms or kills X. The loop reasons *forward from what it can see*, never backward from what it expects.
- **It works from the address alone.** The machine's name is the single strongest recall trigger there is; hand a model a well-known box's name and it can recite a published path before a packet is sent. So the name is withheld until *after* root. When a hostname or certificate gives a box away mid-run, the run declares the recognition in its ledger before the next command — because a recognised box stops being evidence that the loop *discovers*.
- **The write-back is mandatory and includes the loop's own mistakes.** Some of the most valuable entries are not about any target — they are about the loop's failure modes. *Memory read at the start is not memory applied at the moment of acting. The vulnerability with the CVE number is rarely the one that decides how bad the outcome is. Enumerate before you exploit, because the intended path is almost always already visible in a banner or a comment.* Each was paid for by a wasted hour, and each is now read before every run. It is a system that debugs itself.

There is a second, quieter reason I care about the methodology over the machines. A completion count is a claim about a month, and a good result is easy to fake with a good month. Numbers about the *method* carry the signal a flag does not: a re-run of the same target on a later version of the reference set is a controlled experiment — same box, heavier methodology — and the delta is the only honest measure of whether the loop is actually getting better or just getting more attempts. The reference set is the thing under test. The flags are just how you score it.

## The hard case: breaking business logic

Here is the objection worth the most, because it comes from people who do this for a living: *sure, an agent can pin a version and fire a public exploit — but that is not the job. The job is understanding a system well enough to abuse the assumptions its designers never wrote down, and a model doesn't "understand," it pattern-matches. It can't break a business rule it was never shown.*

I want to take that seriously rather than wave the machine count at it, because it is pointing at something real. So first, precisely what business logic is, and why it is the exact case that should defeat an agent.

**A business-logic flaw is a gap between the invariants a system's designers *assumed* and the invariants its code actually *enforces*.** No memory corruption, no injection of a foreign language — just a rule the application takes for granted that turns out to be only a convention. "Only the person with the emailed token can reset this password." "A user can only see their own records." "This directory column is confined to `/srv`." "Setup can only be completed once, by us." Every one of those is a sentence in the designer's head. Whether it is *true* depends entirely on what the code does, and the code is bespoke — written once, for this system, nowhere on the internet to have been read before.

That is why it is the double-hard case. It defeats the two objections at once:

- There is **no pattern to match**. A CVE is a known shape; a logic flaw in a one-off application has no signature, no advisory, no PoC to have memorised. If the loop only regurgitated training data, this is precisely where it would go blind.
- It **requires a model built from scratch**. To find the gap you have to first reconstruct what the designer *intended* — the contract — from nothing but the artifacts in front of you: the source you recovered, the shape of the responses, the wording of an error, a database schema, a route table. Then, for each guarantee the design leans on, you have to ask the adversarial question: *is this actually enforced, or is it just assumed?* That is not retrieval. That is modelling a system you were never told about and finding where its self-image is a lie.

The loop does this, and it does it the same way a good operator does — not by recognising an exploit, but by **tracing intent to enforcement and probing the seam.** A few worked reasonings, each the same move under different clothes:

- **The reset token that trusts equality.** The contract is "only the holder of the emailed token proceeds." The loop reads the check and sees it compares the token with PHP's loose `==`. The adversarial question: *does equality here actually mean "same token"?* It does not — `==` coerces, and a hash beginning `0e…` equals the integer zero. So the enforced invariant is weaker than the assumed one, and a token of `0` satisfies the code while violating the intent. The loop did not recall a "PHP type-juggling exploit." It modelled what the check was *for*, then found that the operator chosen to enforce it does not carry that meaning.
- **The record id that trusts sequence.** The contract is "you see only your own data." The data is addressed by a sequential integer. The loop reasons about how such systems are *seeded* — the lowest ids are the oldest, most privileged records, created before any tenant isolation existed — and asks for id `0` before anything else. That is not a signature; it is a hypothesis about the system's history, tested in one request. The isolation was assumed; the incrementing id enforced nothing.
- **The column that trusts a prefix.** The contract is "this path stays under `/srv`." The enforcement is a database `CHECK` that the value *starts with* `/srv`. The loop asks whether "starts with `/srv`" is the same guarantee as "stays under `/srv`" — and it is not, because `/srv/../home/<user>/.ssh` starts with `/srv` and resolves somewhere else entirely. The constraint enforced a *string shape*; the designer assumed a *filesystem boundary*. The gap between those two sentences is the whole exploit.
- **The value the app validated once and trusted twice.** The recurring one, and my favourite, because it is pure logic. An input is validated for its first consumer — sanitised for display, constrained to a charset for a filename — and the designer assumes that validation protects everything downstream. But the same value reaches a *second* consumer that never re-checks it: a shell command, a config renderer, a query. The loop finds it not by matching a sink but by **following where the input actually goes** and noticing that the guarantee established at the door was never re-established at the second room. The application trusted that validating once was enough. It was not.

None of that is memorised, and none of it is a scanner finding. Every one is the loop building a small model of what the system meant to guarantee and locating the sentence where meaning and mechanism diverge. That is what "breaking business logic" *is*, and the same three rules that make the loop honest make it good at this: it reasons forward from evidence it collected (so it reads the real contract, not a remembered one), it works the actual artifacts (so the model is of *this* system), and it treats every assumed guarantee as a question rather than a fact. The methodology was not tuned for logic bugs as a special case. Logic bugs are just what falls out of enumerating exhaustively and refusing to trust anything you have not proven.

## The spread

Breadth is its own answer to "it can only do the easy, mechanical ones." The difficulty picture:

| tier | machines |
|------|---------:|
| Easy | 21 |
| Medium | 22 |
| Hard | 5 |
| Insane | 1 |

More Medium than Easy, with Hard and Insane both taken end to end. Difficulty on these platforms is not a length rating — a Hard box is one where the *path* is non-obvious, the step you have to reason your way to rather than scan your way to. And it is not one trick on repeat:

| surface | machines |
|---------|---------:|
| Linux | 35 |
| Windows / Active Directory | 12 |
| Kubernetes · OT/ICS · 802.11 wireless | 3 |

Active Directory takedowns through delegation and DCSync. A box reachable only over 802.11, cracked and its traffic decrypted. An OT box where the privilege escalation was writing a calibration offset a root safety process trusted. A Kubernetes escape through a service-account token. Different worlds, different primitives — and for each, the loop reads the reference section that world's earlier boxes wrote, and works the problem there.

## The smaller worry, addressed

There is a narrower doubt I should close, though it is secondary to everything above: *maybe it only reproduces retired-box write-ups it absorbed in training.* Fair for a retired box in isolation, and the discipline already forbids it — you cannot satisfy "trace every action to evidence you collected" with a remembered path, and the name that would trigger the memory is withheld until after root. But the cleaner answer is target selection. I deliberately pointed the loop at **freshly-released machines**, rooted at or near the moment they went live, before any walkthrough existed to have memorised — you cannot recite a path nobody has written yet. And at a **Pro Lab**, a multi-host network with no per-box solution to lean on, which it cleared three hosts of three, four flags of four, in about four and a half hours — and which I do not ask you to take on faith, because the platform issues a completion certificate at 100% and nowhere else. The proof of completion is *issued by the target, not asserted by the attacker.* The lab, its host count and its certificate are in the repository's [`labs.md`]({{ site.baseurl }}/) index; the chain never leaves my machine. Memorisation is not an available explanation for a box with no write-up or a network with no per-host answer, and both fell.

## The point

To the friends who doubted this could work: the doubt was correct to have, and it is no longer correct to hold. But do not fixate on fifty-one — that is the scoreboard, and scoreboards are a claim about a month. Fixate on the six thousand lines the runs wrote, because that is a claim about every month after this one.

Handed nothing but IP addresses, forbidden from looking up the box, and required to justify every move from evidence it gathered itself, an autonomous loop climbed the whole difficulty ladder, crossed six kinds of attack surface, and — box after box — reconstructed what a bespoke system assumed and stepped through the place its code did not enforce it. That last part is the one that was supposed to be impossible. It is not. It is just method: model the contract, question every guarantee, and never trust what you have not proven. The loop writes that method down a little more completely every time it runs, which means the honest headline is not that it rooted fifty-one machines.

It is that fifty-two will be cheaper than fifty-one was.

The name still goes in last. The root still comes first.

*Eurico Nicacio —* [*@h3llh0und*](https://github.com/euriconicacio)
