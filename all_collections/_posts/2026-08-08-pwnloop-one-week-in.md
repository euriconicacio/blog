---
layout: post
title: "pwnloop, one week in"
description: Eight days ago it was a container and a 200-line skill file. It has since rooted seventeen machines and taken a Pro Lab end to end — and every path it had to work out on the way is now written into the skill, so it never works it out twice.
date: "2026-08-08 09:00:00 -0300"
image: /assets/images/posts/pwnloop-one-week-in/01.png
---
> ***Disclaimer.*** *This is a personal open-source project, built and run on my own equipment and my own accounts. It has no connection to any current or former employer, and no client, employer or production environment was involved at any point. Every single-host target was a Hack The Box machine and the campaign target was a Hack The Box Pro Lab, both engaged over that platform's own VPN and within its terms of service. No flag values appear in this post or in the repository. No chain is published for any machine not confirmed retired, and no Pro Lab chain, hostname or credential appears anywhere — flag sharing is a platform violation regardless of a machine's status, and Pro Labs never retire.*

![pwnloop, one week in]({{ site.baseurl }}/assets/images/posts/pwnloop-one-week-in/01.png)

Eight days ago `pwnloop` was a disposable Kali container and a 200-line skill file that I started on a Friday to see whether an agent could take a lab machine from an IP address to a root flag without me in the loop.

It can. That stopped being the interesting question on day two.

The interesting question is what happens the *second* time. Because the loop is not allowed to close a run until it has written back into its own methodology, every engagement leaves the tool different from how it found it — and the difference is always the same shape: something the run had to work out on target is now on the page, so the next run reads it instead of working it out again.

That is the whole mechanism, and a week is the first interval long enough to watch it compound. This post is the state of the tool after eight days, the full surface of what it does, and what each run put into it.

The two earlier posts cover the origin — [the single-host loop]({{ site.baseurl }}/pwnloop-an-autonomous-engagement-loop-for-lab-machines/) and [lab mode against a Pro Lab]({{ site.baseurl }}/pwnloop-lab-mode-a-pro-lab-in-four-and-a-half-hours/). This one does not repeat them.

## Where it is

| | day 1 | day 8 |
|---|---|---|
| Skill file | ~200 lines, single host | loop specification + campaign layer |
| Reference set | 14 files | 26 files, ~4,000 lines |
| Machines | 0 | 17 rooted, 17 root flags |
| Campaigns | 0 | 1 Pro Lab, 3/3 hosts, 4/4 flags |
| Modes | one machine | one machine · one network |
| Releases | unversioned | v1.9.0 — 18 tagged releases |
| State model | a markdown ledger | ledger + campaign state CLI + checkpoints |

Seventeen machines, seventeen root flags, one Pro Lab. It has not walked away from a target.

That is the least interesting row in the table, and I want to get it out of the way early. A completion count is a claim about a week; the reference set is a claim about the next one.

## What it does

### One command, one address

```bash
cd ~/pwnloop && claude
> /pwnloop 10.129.x.x
```

The agent verifies the VPN and reachability, creates an engagement directory named after the address, and runs recon → enumeration → foothold → privilege escalation → cleanup → report without stopping between phases. It announces each flag in chat the moment it reads one, so you can submit while it keeps working.

You pass the address, never the machine's name. The name is the strongest recall trigger there is: hand a model a well-known box's name and it can return the published chain before a packet has been sent, and you will never know how much of the run was discovery and how much was retrieval. Working from an address means recognition can only happen *after* enumeration has earned the fingerprint. When it happens anyway, the run declares it in the ledger before the next command.

Four conditions — and only four — return control to the operator: VPN down, target unreachable for more than five minutes, a genuine scope question, or three full enumeration passes with no new leads. Everything else it decides. Generous escalation criteria are how an autonomous agent quietly degrades into a chat session.

### One command, one network

```bash
> /pwnloop-lab 10.10.110.0/24
```

Campaign mode points the same loop at a graph instead of a box. The single-host methodology becomes the inner loop, unchanged, run per host. The outer loop picks the highest-value lead off a frontier — unowned reachable host, untried credential, new subnet — and writes everything it learns to disk through a CLI that is the state file's only writer.

Its first run took HTB's free Puppet mini Pro Lab from one entry address to full compromise: 3 of 3 hosts, 4 of 4 flags, seven sessions, about four and a half hours of wall-clock, verified by the platform's own completion certificate rather than by a screenshot I produced.

### What comes out

```text
engagements/<address>/
  FINDINGS.md    append-only ledger, updated live — findings, evidence, status
  REPORT.md      defender-facing: chain, impact, remediation, earliest break point
  WRITEUP.md     teaching-facing: the narrative, including the leads that failed
  scans/         raw tool output, one file per run
  loot/          credentials, hashes, keys, downloaded artifacts
  www/           payloads staged for delivery to the target
```

`REPORT.md` and `WRITEUP.md` are separate documents on purpose. The report argues to a defender: what is broken, what it costs, what to fix, and which single control would have broken the chain earliest. The write-up teaches a reader: how the box fell, and — the part most write-ups omit — which leads were dead ends and why. A defender does not want your narrative and a learner does not want your CVSS table.

### The machinery underneath

**A disposable container.** All offensive tooling and the lab VPN live inside it. The container gets `NET_ADMIN` and `/dev/net/tun`, so OpenVPN runs there and your host's routing table is never touched. Tear it down and every trace of the engagement's tooling goes with it. Native arm64 on Apple Silicon, no emulation.

**An append-only ledger.** Every finding carries an evidence file and a status: `LEAD` → `CONFIRMED` / `PARKED` / `DEAD`, where `DEAD` carries the reason it was ruled out. That last part is what stops the loop re-testing it on the next pass. If you are demonstrating this to a room, project a `tail -f` of this file rather than the transcript.

**A grounding invariant.** No finding without an evidence file. Before running a command the loop should be able to name the file and the line of output that motivated it. This is the single highest-value constraint in the whole document: it forces the loop to operate on *observed* state rather than *plausible* state.

**A divergence guard.** Any lead that has produced no concrete artifact in about fifteen minutes is marked `PARKED` and the loop moves on. Without it, an agent spends an hour on the most *interesting* lead rather than the most *productive* one.

**A credential matrix.** Every spray result is recorded, win or lose, so the loop only ever suggests untried (credential, host, service) triples. A locked result removes that credential from every future suggestion. Recording a negative is worth exactly as much as recording a positive — it is what stops the next session re-running it, and lockout is the only irreversible mistake available in a lab.

**Route canaries.** Every tunnel is registered with an `IP:port` behind it known to answer, so a route can be tested rather than trusted. After a lab reset, a dead tunnel and a hardened target are indistinguishable, and one of those two readings wastes a whole session.

**Checkpoints.** The state file records what is true; the checkpoint records what you were in the middle of and what you would have done next — the one thing nothing can infer for you. Written before running out of room, not after.

**A memory split.** `memory/patterns.md` is upstream and curated; `memory/local.md` is yours and created at install. Both are read before every engagement, and neither ever conflicts on a pull. Same split for the container package list.

**Leak controls.** `engagements/`, `campaigns/`, `flags.local.md` and `vpn/` are gitignored, and a pre-commit hook refuses any commit containing a flag-shaped string, a path under those directories, private key material or an attacker VPN address. The realistic failure is the maintainer pasting engagement output into a reference file, not an outsider — so the hook exists for me.

**`pwnloop backup`.** An encrypted archive of the two things git does not track and cannot recover: the local memory file and `campaigns/`. A direct consequence of campaign mode — a campaign produces nothing publishable, so it is its own only record.

### What it deliberately is not

The exclusions are decisions, not a roadmap:

- **No C2, EDR evasion or malware development.** A lab machine needs none of it, and a repository that ships it is a different kind of artifact with a different set of obligations.
- **No phishing or social-engineering infrastructure.** There is no human on the other side of a lab box.
- **No log or audit tampering.** Cleanup removes the operator's artifacts and nothing else. The engagement's footprint in the logs is the defender's evidence and part of what makes the exercise worth anything.
- **No write-up lookup, and no acting on recall.** This is the one I get asked about most. Fetching the target's own walkthrough would raise the completion rate immediately and destroy the only thing being measured. Researching a *technology* — a CVE, a protocol, an exploit's source — is the opposite of that and is constant.

## The week

Eighteen releases in eight days. What makes them worth listing is that not one of them is a feature I sat down and designed — every single one is a gap a run walked into first.

**Fri 31 Jul – Sun 2 Aug — the loop, and ten machines.** Container, skill file, first targets, six releases. The gaps were mundane and expensive: packages that were absent exactly when credential reuse, pcap analysis or PDF extraction needed them; a preset git identity, without which `git commit-tree` refuses and plumbing-based exploitation is impossible; the fact that `sed -i` cannot edit a bind-mounted `/etc/hosts`. Every one of those cost a run several minutes and costs nothing ever again.

**Mon 3 Aug — three more machines, four releases.** The first post went out in the morning; Support, Snapped and Zero went down that afternoon and evening. Zero — the only Insane box in the set — produced two new primitive classes on its own: an attacker-controlled `.htaccess` as a file-read primitive where a served directory is writable, and a root command-injection class where a privileged config-check rebuilds its command from a process's own command line.

**Tue 4 Aug — Lame, and a blind spot worth more than the box.** A deliberately antique host, easy by any measure, which exposed something the methodology had no answer for: a correct, well-evidenced lead that **stock tooling can no longer deliver**. The injectable field was one a modern client negotiates *around*, so `smbclient` and impacket packed it into an NTLMSSP blob where the metacharacters never reached a shell. The symptom is indistinguishable from "not vulnerable" — a clean `NT_STATUS_LOGON_FAILURE` and no side effect — and the loop very nearly filed it as a dead end. It instead hand-built the raw SMB1 exchange and got unauthenticated RCE as root in a single packet. That diagnosis order is now a reference section: confirm the precondition on the target rather than arguing with the version, know which legacy client knobs are *accepted and ignored*, emit the raw exchange yourself, and confirm with a side-effect oracle, since these bugs run the command and *then* reject the login.

**Wed 5 Aug — campaign mode, a Pro Lab, and a two-forest AD box.** The last line of the first post's limitations was that nothing in the sample was multi-host. Campaign mode closed it in five releases, and the Puppet run immediately found two things the reference set had no page for: **deployment infrastructure** — configuration management is an authenticated API whose entire purpose is running code as root on every node it manages, which makes it the highest-value target on an internal network — and **operating through a C2 framework**, which is how a modern internal engagement actually moves, as opposed to the single shell a machine gives you. Both are pages now. That evening it took a two-forest Active Directory machine across a trust boundary, which produced the release the same day: a *blocked precondition* — an approval gate, a missing role, a patched sink — is the cue to re-enumerate the version's other CVEs for a sibling with a different trigger, not a dead end; and multi-RPC operations drop through a SOCKS proxy as `INVALID_HANDLE`, so prefer a stable local forward and keep the receiving host in-segment.

**Fri 7 – Sat 8 Aug — two machines, and one release with a theme.** A camera-management box and a monitoring-stack box, and between them the run stalled at three separate seams that turned out to be the same seam: **a tool's silence read as a negative result.** A live path traversal answered `404`, because the sink was a static route rather than a parameter and `curl` had collapsed the `..` client-side before the request was ever sent — the fix is `--path-as-is`. A hash that looked like an uncrackable password was a format john would not load: a PBKDF2 digest longer than the 32 bytes the format takes, reported as the indistinguishable `No password hashes loaded`, and truncatable because the derivation is block-concatenated. And a `sudo` rule that read as restrictive ended in a trailing `*`, which grants the binary's *entire flag surface* — including every option that changes what privilege the work runs with — so the escalation is a documented feature of the allowed binary and nothing about it looks anomalous.

The rule that came out of that is worth more than the three techniques: **validate the pipeline with a control input you constructed before believing any negative result.** A cracker that finds nothing, a request that 404s and a rule that looks locked down are all the same claim — "there is nothing here" — made by a tool that was never asked whether it could see.

Three of those seventeen machines have no published write-up. One is an **active, unretired machine** — and that one is the single most useful result in the set, for a reason I did not plan: there are no published walkthroughs of it yet, so recall is not merely controlled for, it is unavailable. The other two are simply not confirmed retired. All three chains stay in the gitignored engagement directory until they are, which is the rule, and the rule does not bend for the result I would most like to show you.

## The second loop

This is the part I would keep if I could keep only one.

Every engagement is required to change the methodology before it closes out. Not because I sit down afterwards to improve it — because *finishing a run requires writing back into it*. A pattern that generalises goes to a memory file the next run reads before it starts. A tool installed mid-run becomes a package. A technique that worked and was undocumented becomes a reference section. A wrong turn becomes a rule that removes that turn from the search space.

Over eight days that took the reference set from 14 files to 26 — Kubernetes, LLM and agent platforms, container escape, cloud metadata, the full AD CS ESC1–ESC16 catalog, NTLM/Kerberos coercion and relay, binary exploitation, deployment infrastructure, C2 operations, multi-hop pivoting — roughly 4,000 lines of methodology, all of it produced by a run that needed it and did not have it.

One rule keeps that file from rotting: **write the method, never the box's answer.** A reference entry reading "product X version N → CVE-Y → run this payload" bakes one machine's solution into the methodology and turns the next run into recall. The transferable class goes into the shared references; the box-specific recipe stays in a local memory file that is not shared at all. For campaign mode the same rule doubles as a platform-rules boundary: `skills/` and `references/` are public and a Pro Lab never retires, so the test before an entry is committed is whether a reader could use it to identify the lab or skip a step on it.

The change I am proudest of is still a **deletion**. An early run left the rule "fix Kerberos clock skew with `ntpdate -u <dc>`." A later one proved that cannot work — the container has no `CAP_SYS_TIME`, so `ntpdate` measures the offset and then fails to apply it — and it would have blocked certificate authentication outright. The entry was removed and replaced with a `faketime` shim that shifts one process instead of the system clock.

A loop that only accumulates gets worse over time. The interesting property is that this one can also take something out.

## Discovery is paid once

Here is the thing I did not expect to be the headline of the week.

The loop does not give up. In seventeen engagements and one campaign it has not walked away from a target, and it has not stalled out cycling on the same three ideas — the ledger, the `DEAD` reasons and the fifteen-minute time-box exist precisely to make both of those impossible. What it does instead, when it meets something the methodology has no page for, is work the thing out on target: read the binary, read the script, pin the version, compose the primitive out of shell.

That costs time. It cost time *exactly once*.

Because every one of those moments ends the same way — the run cannot close until the thing it worked out is a rule, a package or a reference section. The expensive discovery happens on one machine. Every machine after it reads the answer before it starts. That is what makes a week of runs different from a week of demos, and it is why the reference set is the artifact I would point at rather than the completion count.

Three are worth naming, because in each one you can see the same trade: an hour spent once, a page that costs nothing forever.

**The version was pinned and the CVE hunt was not a step yet.** On one Linux box the loop enumerated cleanly, identified the service and pinned the product — and then went straight to hand-rolling exploitation, because nothing in the methodology said to stop and look for published vulnerabilities in the version it had just pinned. The intended path was a recent CVE whose public proof-of-concept contained the one delivery detail that mattered: which protocol field was the actual sink, as opposed to the three plausible fields it had already tried. Pinning the precise version with a protocol-specific probe rather than an `nmap -sV` guess, then hunting CVEs and weaponised public exploits, is now a primary step in the loop rather than something it might get to.

**The right binary, and no rule for choosing between its CVEs.** Root on another box was a custom daemon on localhost. The loop found a memory-corruption CVE in it, confirmed the overflow on target, defeated PIE with an address leak — real work, all of it correct — and then spent ninety minutes on an exploit whose primitive turned out to be a *linear-forward* write rather than an arbitrary one, which put the target address, at a lower offset, permanently out of reach. Public research on that CVE stops at the same wall for the same reason. A second, far cheaper CVE in the same daemon — an argument injection in a format string — was the intended path and is a one-liner. What was missing was not capability; it was a *selection rule*. The methodology now says: for a pinned version, enumerate **all** of its CVEs and reason about the set, ranked by cost and reliability — an auth bypass or an argument injection beats a memory-corruption bug on a hardened target — consider chaining them, and exhaust public weaponised exploits before writing your own.

**A tool that answered "nothing here", three times in one run.** The most recent release is the cleanest example of the pattern, because all three gaps have the same shape. A traversal that was live and read as patched, a hash that was crackable and read as an uncrackable password, a `sudo` rule that was a root grant and read as a restriction. In each case the loop had the finding in front of it and a tool told it there was nothing there. Nothing in the methodology said *that answer can be a lie, and here is how to prove it either way* — so the rule that went in is not any of the three techniques, it is the control input: construct a case you know the answer to, run it through the same pipeline, and only then believe a negative.

Note what those three have in common, because it is the actual argument of this post. In none of them was the loop incapable. What was missing each time was an *ordering* or a *check* — it already knew how to hunt a CVE, how to weigh one exploit against another, how to crack a hash. Nothing told it in what order, or when to distrust an answer. The methodology was silent at exactly one point in each run, and that silence is a thing you can fix permanently in about forty lines.

Which is why the write-back rule above is the one that carries the whole design. What each of those runs sent upstream was not the CVE it eventually used — that would be one box's answer, and a reference set full of answers turns the next run into recall. What went up was the ordering that finds that CVE without being told, and the check that stops a silent tool from closing a live lead. Those two rules keep working on machines I have never seen.

## What is next

**A bigger network.** Campaign mode is designed for nineteen hosts and has run against three. Per-host delegation to subagents never triggered at all — it only fires at three or more live hosts on one subnet — and the credential matrix ended with ten credentials and a single recorded attempt. The machinery meant to matter most at scale is, so far, untested at scale.

**Binary exploitation still has not closed a chain.** The first post listed it as a gap and it remains one. The loop has done real work at that layer — a confirmed overflow, a PIE defeat, a correct read of why a primitive was insufficient — and has never landed root through memory corruption, because on every box where it might have, a cheaper path existed and the methodology now correctly tells it to take the cheaper path. That is the right call per-run and it means the capability stays unmeasured. Fixing it needs a target where the cheap path is not there.

**Re-runs as the real instrument.** The honest measurement of "the loop got better" is the same target twice with a methodology that changed in between, recording what memory short-circuited and what stayed slow anyway. The second half is the useful one: a phase still slow across two runs is the next thing to fix. The format is specified and I have barely done it. One week is not enough to read that instrument, and I would rather say so than round it up.

**A reproducible container.** It builds from a rolling-release base with unpinned packages, so builds are resilient — a package that disappears is logged rather than breaking the image — but not bit-for-bit repeatable. Fine for lab work. For engagement tooling behind a report someone relies on, I would pin the base image by digest and freeze the package snapshot.

[**github.com/euriconicacio/pwnloop**](https://github.com/euriconicacio/pwnloop) — MIT. Machine results are in `writeups/`; campaign results are in `labs.md`, which explains exactly what may and may not go in a row.

If you run it against something that makes it work for its answer, that is still the most useful thing you could send me. A target that costs the loop an hour is worth more right now than another one it already has the page for.

*Eurico Nicacio —* [*@h3llh0und*](https://github.com/euriconicacio)
