---
layout: post
title: "pwnloop lab mode: a Pro Lab in four and a half hours"
description: "Campaign mode points the loop at a network instead of a box — a frontier, a credential matrix, tunnels that prove themselves. Its first run took HTB's free Puppet mini Pro Lab end to end: 3/3 hosts, 4/4 flags, seven sessions, ~4h30."
date: "2026-08-05 12:00:00 -0300"
image: /assets/images/posts/pwnloop-lab-mode-a-pro-lab-in-four-and-a-half-hours/01.png
---
> ***Disclaimer.*** *This is a personal open-source project, built and run on my own equipment and my own accounts. It has no connection to any current or former employer, and no client, employer or production environment was involved at any point. The target was a* ***Hack The Box Pro Lab***, *engaged over that platform's own VPN and within its terms of service. Pro Labs never retire, so this post contains* ***no chain, no hostnames, no credentials and no flags*** *— and neither does the repository. What a lab is worth publishing is covered near the end, and it is deliberately not a walkthrough.*

![pwnloop lab mode: a Pro Lab in four and a half hours]({{ site.baseurl }}/assets/images/posts/pwnloop-lab-mode-a-pro-lab-in-four-and-a-half-hours/01.png)

Two days ago I wrote up [`pwnloop`](https://github.com/euriconicacio/pwnloop), an autonomous engagement loop that takes a single lab machine from an IP address to a root flag without checking in. The last line of that post's limitations section was that the sample had nothing multi-host in it.

That gap is now closed. **Campaign mode** points the same loop at a *network*, and its first run took HTB's free **Puppet** mini Pro Lab from one entry address to full compromise:

**3 of 3 hosts owned. 4 of 4 flags. Seven sessions. About four and a half hours of wall-clock.** Verified by the platform's own completion certificate, `HTBCERT-0398EDB2AC` — not by a screenshot I produced.

![Puppet campaign result]({{ site.baseurl }}/assets/images/posts/pwnloop-lab-mode-a-pro-lab-in-four-and-a-half-hours/02.png)

Everything below is about the *harness*. The lab's path is not here and will not be, for a reason I will get to.

## A Pro Lab is not a long machine

A machine is one target and one context window. You can hold the whole thing in your head, and if you cannot, the run was probably too long anyway.

A Pro Lab is a **graph**. Most of its hosts are unreachable until you have owned another one. The work outlives your context window many times over. And the mistakes that cost you a day are not technical — they are organisational:

- spraying a credential you already sprayed, three hosts and two hours ago;
- debugging a scan through a tunnel that died forty minutes ago, and reading the results as *"this host is hardened"*;
- forgetting which of nineteen hosts still has an unexplored port;
- locking out the one account that mattered, which is the only truly irreversible mistake available in a lab.

Every one of those is a *state* problem. So campaign mode is almost entirely about state.

## The two-level loop

The single-host methodology is not replaced. It becomes the **inner** loop, run per host, unchanged. What lab mode adds is the outer loop that decides *which* host, keeps the network model on disk, and survives being resumed by a session that remembers nothing:

```text
campaign loop  ── pick the highest-value lead from the frontier
     │             (unowned reachable host · untried credential · new subnet)
     │
     ├─→ host loop   ── the pwnloop skill, scoped to one IP
     │                   recon → enum → foothold → privesc → flag
     │
     └─→ write back  ── every host, credential, route, flag and lead goes into
                        campaign.json through the CLI, never by hand
```

```bash
cd ~/pwnloop && claude
> /pwnloop-lab 10.10.110.0/24     # an entry range
> /pwnloop-lab 10.10.110.5        # or a single entry host
```

**You pass the entry point, never the lab's name** — the same trade the single-host loop makes with machine names, for the same reason. A lab's name is the strongest recall trigger there is, and the best-documented Pro Labs are the ones whose names carry the most. The campaign directory is derived from the address, `campaigns/.current` is the handle, and you are asked what the lab is called only at the end, when the search order is already on record. Asking then costs nothing. Asking at the start costs the entire measurement.

## State is the design

The rule that makes the whole thing work is one line: **nothing important lives in your context.** If a fact matters after the current host, it goes through the state CLI the moment it is learned. Assume you will be resumed by a session that has read nothing.

There are four layers, and knowing which is which is the difference between a campaign that can be handed over and one that only exists in a transcript:

![The four state layers]({{ site.baseurl }}/assets/images/posts/pwnloop-lab-mode-a-pro-lab-in-four-and-a-half-hours/03.png)

The CLI is the **only writer** of the state file. On a twenty-host lab, free-form state edits drift within hours. Everything the agent learns goes in as a command:

```bash
pwnloop host add 10.10.110.100 os=linux subnet=10.10.110.0/24 ports="22 80 445"
pwnloop cred add user=james secret='S3cret!' source=/var/www/.env host=10.10.110.100
pwnloop try c3 10.10.110.101 smb ok        # every spray result, win or lose
pwnloop route add subnet=172.16.1.0/24 via=10.10.110.100 type=ligolo canary=172.16.1.5:445
pwnloop lead add kind=subnet target=172.16.2.0/24 note="route print on .100" prio=1
```

Three mechanisms carry most of the value.

**The credential matrix.** Every spray result is recorded, win *or* lose, so `pwnloop try next` only ever suggests untried (credential, host, service) triples. A `locked` result removes that credential from every future suggestion. Recording a failure is worth as much as recording a success: it is what stops the next session re-running it.

**Route canaries.** Every tunnel is registered with an IP:port behind it that is known to answer, so `route check` *tests* whether a route still carries traffic instead of trusting the state file. An unverifiable route is worse than no route — after a lab reset, a dead tunnel and a hardened target are indistinguishable, and one of those two readings wastes a whole session.

**Checkpoints.** The state file records what is *true*. The checkpoint records what you were in the *middle of* and what you would have done next, which is the one thing nothing can infer for you. It is written before running out of room, not after, and `resume` prints it before anything else.

Then the resume protocol, in a fixed order: VPN first, because a dead VPN makes every route look dead. Then the **entry host**, before any route — it is the first hop of every chain, so if it was reset, everything below it is dead for one reason and testing them individually tells you nothing. Then re-establish dead routes. Then re-verify one owned host per subnet, because labs get reset. *Then* pick a lead.

## When the entry point is one host

The common Pro Lab shape is a single address fronting a network you cannot see yet. There, phase 0 is an ordinary single-host engagement — the machine loop, exactly as written, because there is no frontier to rank and depth is the only move.

The campaign proper begins **at the first shell**, and there the priority inverts: mapping outward outranks escalating locally. Interfaces, routes, ARP, DNS, domain trusts — each one becomes a host or a subnet lead *before* going back to privesc. A second NIC on the entry host is the actual door. Root on the entry host without it is a dead end with a flag attached.

## Seven sessions

The design assumes an operator with a few hours, not a weekend, and Puppet was run exactly that way: seven sessions, each ending with a deliberate checkpoint rather than just stopping.

Two things happened that I could not have arranged on purpose, and both are worth more than the completion.

**The lab went dark mid-run.** Every implant on one host dropped at the same instant, and the secondary access path — a key-based login that had worked for two sessions — started being refused too. The correct read there is not "my technique broke", it is "the environment moved": simultaneous, uniform failure across independent mechanisms is an *environment* event. The loop tested once with verbose output, established that a clean key rejection is not a lockout, waited six minutes without hammering anything, retested once, and then wrote a checkpoint and stopped. That restraint is the whole game — the wrong move there is a retry storm that turns a temporary outage into a real lockout.

**The next session got it all back.** It resumed from disk, found the environment recovered, re-established access and finished. That is precisely the property the state model exists to produce, and it is the first time I have seen it pay off under conditions I did not create.

## What the run changed

Finishing a campaign requires writing back into the methodology — it is not optional and it is not the report's job. This one exposed two capability gaps that had been invisible while the loop only ever ran against single machines, and produced four more entries:

![What the campaign wrote back]({{ site.baseurl }}/assets/images/posts/pwnloop-lab-mode-a-pro-lab-in-four-and-a-half-hours/04.png)

The two new files are the ones I would point at. The reference set had **nothing on deployment infrastructure** — configuration management is an authenticated API whose entire purpose is running code as root on every node it manages, which makes it the highest-value target on an internal network, and the loop had no page telling it to look. And it had **nothing on operating through a C2 framework**, which is how a modern internal engagement actually moves, as opposed to the single shell a machine gives you.

The constraint on writing any of that down is the interesting part. `skills/` and `references/` are public, and a Pro Lab is never retired — so the write-back rule that is normally about *methodology quality* is here also a platform-rules boundary. The test before an entry is committed: **could a reader use it to identify the lab, or skip a step on it?** If yes, it goes in a gitignored memory file instead. Write the class, never the chain; and if a technique genuinely needs a concrete example, take it from a retired machine or vendor documentation, never from the campaign that prompted it.

## What the run broke

The more useful half. A live campaign found six defects, and the pattern in them is not subtle — five of the six are cases where a mechanism **silently did nothing** while everything looked fine:

![What the first live campaign broke]({{ site.baseurl }}/assets/images/posts/pwnloop-lab-mode-a-pro-lab-in-four-and-a-half-hours/05.png)

The one I would frame is the last one. `campaigns/` was gitignored on the branch that shipped the feature, but the working copy was checked out on a branch that predated it — so for a window, live engagement data sat in a repository that would happily have committed it. The content rules (private-key material, flag-shaped strings) would have caught it at commit time, which is the wrong layer to depend on. A path rule that only exists on one branch is not a control.

## Why there is no write-up

The single-host loop produces a teaching write-up, and ten of them are in the repository. Campaign mode deliberately produces **none**.

Machines retire, and a retired machine's solution becomes publishable. Pro Labs never retire — there is no future date at which sharing one stops being a violation, so the honest position is that the chain, the hostnames, the credentials, the per-host ledgers and the report stay in `campaigns/`, gitignored, on the machine that ran them, indefinitely.

What a campaign *may* publish is the numbers, and I think they are actually the more interesting artifact. A write-up proves a machine fell to *someone*; it reads the same whether the path took forty minutes or was known in advance. Hosts owned per session, how much frontier was open when a session ended, how long a resumed session took to get back to productive work, how many credentials the matrix replayed into a foothold — those describe the **loop**, not the lab. They are what should improve between one campaign and the next, and none of them says what any host was running.

The lab's own page is a different matter: its name, entry point, machine count and scenario are the vendor's marketing copy, and repeating them discloses nothing. The line is between what the platform tells everyone and what the lab told *me*.

## Honest limitations

**Three hosts is a small network.** Campaign mode is designed for nineteen. Two of its mechanisms barely fired: the credential matrix ended with ten credentials and a single recorded attempt, and per-host **delegation to subagents never triggered at all**, because it only kicks in at three or more live hosts on one subnet. The machinery that is supposed to matter most at scale is, at this point, untested at scale. (The empty matrix is also how one of the six defects was found: a campaign can accumulate credentials while recording no attempts, silently disabling the exact mechanism that prevents re-spraying and lockouts. `campaign status` now says so loudly.)

**Wall-clock is not effort.** Four and a half hours is elapsed time across seven sessions on a three-host lab. It is a baseline for *this lab, this methodology* — the number a second run gets measured against — not a claim about Pro Labs in general.

**One campaign is not a trend.** Same caveat as the ten machines. The honest test of "the loop got better" is the same lab twice, with a methodology that changed in between, recording what memory short-circuited and what stayed slow anyway.

**A campaign is its own only record.** Because nothing publishable comes out of one, the campaign directory has no replication path — which is why this release also added `pwnloop backup`, an encrypted archive of exactly the two things git does not track and cannot recover: the local memory file, and `campaigns/`.

---

[**github.com/euriconicacio/pwnloop**](https://github.com/euriconicacio/pwnloop) — MIT. The results index is [`labs.md`](https://github.com/euriconicacio/pwnloop/blob/main/labs.md); it explains exactly what may and may not go in a row.

The next thing I want is a bigger network — enough hosts that delegation and the credential matrix have to carry real weight, and enough sessions that the resume protocol is doing something other than working. If you run this against something that breaks it, that is the most useful thing you could send me.

*Eurico Nicacio —* [*@h3llh0und*](https://github.com/euriconicacio)
