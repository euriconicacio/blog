---
layout: post
title: "pwnloop: an autonomous engagement loop for lab machines"
description: Hand it an IP and it runs recon to root to cleanup to report without checking in — then rewrites its own methodology before it is allowed to finish.
date: "2026-08-03 09:00:00 -0300"
image: /assets/images/posts/pwnloop-an-autonomous-engagement-loop-for-lab-machines/01.png
---
> ***Disclaimer.*** *This is a personal open-source project, built and run on my own equipment and my own accounts. It has no connection to any current or former employer, and no client, employer or production environment was involved at any point. Every target was a* ***retired*** *Hack The Box machine, engaged over that platform’s own VPN and within its terms of service. No flag values appear in this post or in the repository — flag sharing is a platform violation regardless of a machine’s status.*

![pwnloop: an autonomous engagement loop for lab machines](/assets/images/posts/pwnloop-an-autonomous-engagement-loop-for-lab-machines/01.png)

`pwnloop` is a Claude Code skill plus a disposable Kali container. You spawn a lab machine, hand it the address — **just the address** — and it runs the engagement end to end:

```text
> /pwnloop 10.129.x.x
  [recon]    3 ports — 21 vsftpd 3.0.3, 22 OpenSSH 8.2p1, 80 Gunicorn
  [web]      /capture → 302 /data/1 — sequential id, no ownership check
  [web]      /data/0 belongs to another user. IDOR confirmed
  [analysis] pcap holds a cleartext FTP login
  [foothold] password reused on SSH
             user.txt — <32-hex-flag>
  [privesc]  cap_setuid on /usr/bin/python3.8
             root.txt — <32-hex-flag>
  [cleanup]  1 artifact removed, verified
```

It does not stop between phases. It announces each flag the moment it reads one, so you can submit while it keeps working. It removes what it created on the target before it finishes. It writes three documents: a live findings ledger, a defender-facing report, and a teaching write-up. And then it does the part I care about most — it changes its own methodology based on what the run taught it, because the run is not allowed to close until it has.

It is MIT-licensed: [**github.com/euriconicacio/pwnloop**](https://github.com/euriconicacio/pwnloop)

## Getting it running

```bash
git clone https://github.com/euriconicacio/pwnloop ~/pwnloop
cd ~/pwnloop && ./install.sh
```

That links the skill into `~/.claude/skills/` and builds the container. Then the VPN — which runs *inside* the container, so your host's routing table is never touched:

```bash
cp ~/Downloads/lab_yourname.ovpn ~/pwnloop/vpn/
pwnloop vpn lab_yourname.ovpn
pwnloop vpn-status          # expect an address on tun0
```

Spawn a machine, then `cd ~/pwnloop && claude` and `/pwnloop <ip>`.

The container ships the usual toolchain — nmap, ffuf, feroxbuster, netexec, impacket, evil-winrm, certipy, kerbrute, sqlmap, hashcat, responder, tshark, chisel, SecLists — plus linpeas, winPEAS and pspy staged for delivery to targets. On Apple Silicon it builds native arm64, no emulation.

## Why only the address

This is the first thing people push back on, so it is worth taking early.

The machine’s name is the strongest recall trigger there is. Hand a model a well-known box’s name and it can return the published chain before a single packet has been sent — and you will never know how much of the run was discovery and how much was retrieval. Working from an address alone means recognition can only happen *after* enumeration has earned the fingerprint, by which point the search order was already set honestly.

Three rules follow from it, and they are in the skill file rather than in my good intentions:

- **The engagement directory is named after the address**, and the run asks what the box is called only at the very end, when the work is done and the search order is on record.
- **Never look up the answer.** Researching a *technology* — a CVE, a protocol, an exploit’s source, what a capability actually grants — is expected and constant. Opening the target’s own walkthrough is not. The line is between “how does this thing work” and “what is the path on this box.”
- **Every action must trace to an artifact already collected.** Before running a command you should be able to name the file and the line of output that motivated it. If the reason is “boxes like this usually have X,” that is recall, not deduction — go collect the observation first.

And when recognition happens anyway, the run declares it in the ledger before the next command. A recognised machine is still worth running; it validates tooling and coverage. What it stops being is evidence that the loop *discovers*. Recording that distinction is the difference between a demo and a measurement.

## What you actually get

The output is the point, more than the root shell is.

`**FINDINGS.md**` is an append-only ledger, updated live. Every finding carries an evidence file and a status: `LEAD`, `CONFIRMED`, `PARKED`, or `DEAD` — and `DEAD` carries the reason it was ruled out. If you are demonstrating this to a room, project a `tail -f` of this file rather than the transcript. Watching findings land in real time is the thing that lands.

`**REPORT.md**` argues to a defender: the chain as a numbered path, findings with impact and remediation, and a section naming the single control that would have broken the chain earliest. That last part is the one that changes what gets funded.

`**WRITEUP.md**` teaches a reader: the narrative, including — and this is the section most write-ups skip — the leads that failed and why they were dead ends.

They are separate documents on purpose. A defender does not want your narrative, and a learner does not want your CVSS table. Merging them serves neither.

## Ten machines

I started this on a Friday with a container and a 200-line skill file. By Sunday it had rooted ten retired Hack The Box machines and gone through seven releases, every one of them driven by something a run discovered the methodology was missing.

![pwnloop: an autonomous engagement loop for lab machines](/assets/images/posts/pwnloop-an-autonomous-engagement-loop-for-lab-machines/02.png)

Redacted write-ups for all ten are in the repository. Times ranged from **three minutes** to **just over two hours**, and the spread is not really about difficulty — it is about how much of the chain was enumeration versus how much was code that had to be written on the spot.

Two of them are worth reading in full, for opposite reasons.

**Nexus** is the one where composition mattered. A world-readable maintenance script synchronised “template” repositories: for each one it walked the git tree and extracted every blob to `os.path.join(staging_dir, filepath)`, where `filepath` came straight out of `git ls-tree`. No normalisation, no containment check, and a systemd unit ran it as root every sixty seconds.

The exploit is a path that escapes the staging directory. Git will not build one through the index — `git add` rejects `..`. But the index is a convenience layer; tree objects are just name-to-hash maps, and `git mktree` writes them directly:

```text
BLOB=$(cat key.pub | git hash-object -w --stdin)
T=$(printf '100644 blob %s\tauthorized_keys\n' "$BLOB" | git mktree)
T=$(printf '040000 tree %s\t.ssh\n' "$T" | git mktree)
T=$(printf '040000 tree %s\troot\n' "$T" | git mktree)
for i in 1 2 3 4 5; do T=$(printf '040000 tree %s\t..\n' "$T" | git mktree); done
```

The forge accepted the push without running fsck on the objects. A minute later, in root’s log:

```yaml
Found 1 template repo(s)
  synced: ../../../../../root/.ssh/authorized_keys
```

To be precise about what that shows: the loop did not invent a technique. It read a script it could not modify, noticed a filesystem path came from a source an ordinary user controls, checked what the systemd unit ran as, and composed the primitive out of shell. Ordinary competent work — executed without stopping to ask, with an evidence file behind every step.

**Forest** is the one where the interesting part was a failure in the middle. The chain itself is one of the most documented paths in Active Directory. But the loop’s first attempt added itself to a group over an existing WinRM session and then could not use the new rights, because a Windows access token is fixed at logon: the session it already held did not contain the group it had just joined. And the box runs a reset job that reverts changes every few minutes, so a fresh login raced against it.

It diagnosed both — read `memberOf` back out of LDAP, saw the membership was gone — and switched to chaining the whole sequence from a single Linux-side window, re-authenticating on each step so that neither a stale token nor the reset window could interfere. That is the behaviour I was actually testing for: not the exploit, but what it does when the obvious approach silently fails.

## Why it is built this way

This is where the tool differs from a prompt that says “hack this box,” and it is worth explaining, because the difference is most of the value.

Almost every agent I have seen fail, fails the same way, and it is not a capability problem. It re-derives what it already worked out, because nothing was written down. It cycles through the same three ideas, because nothing records that idea two was ruled out. It spends an hour on the most *interesting* lead rather than the most *productive* one, because nothing tells it when to quit. It reports a step it never verified. Or it stops to ask a question it had everything it needed to answer.

Five failures, all of them control-flow. So the skill file is not written as a prompt. It is written as a loop specification, and every rule maps to a failure it prevents:

![pwnloop: an autonomous engagement loop for lab machines](/assets/images/posts/pwnloop-an-autonomous-engagement-loop-for-lab-machines/03.png)

The grounding invariant is the one I would keep if I could keep only one. It forces the loop to operate on *observed* state rather than *plausible* state. An agent permitted to write “the service appears vulnerable” will eventually build on a vulnerability that was never there. An agent required to attach the request and the response cannot.

The escalation predicate is the one people get wrong most often. Generous criteria for “ask the human” are how an autonomous agent quietly becomes a chat session.

## The second loop

Every engagement changed the tool — not because I sat down afterwards to improve it, but because **finishing a run requires writing back into it**. A pattern that generalises goes to a memory file the *next* run reads before it starts. A tool that had to be installed mid-run becomes a package. A technique that worked and was not documented becomes a reference section. A mistake becomes a rule.

Over the weekend that turned 14 reference files into 23 — Kubernetes, LLM and agent platforms, container escape, cloud metadata, the full AD CS ESC1–ESC16 catalog, NTLM/Kerberos coercion and relay, binary exploitation — roughly 3,500 lines of methodology, all of it produced by a run that needed it and did not have it.

The rule that keeps that file useful is a small one: **write the method, never the box’s answer.** A reference entry that reads “product X version N → CVE-Y → run this payload” bakes one machine’s solution into the methodology and turns the next run into recall. The transferable class goes into the shared references; the box-specific recipe stays in a local memory file that is not shared at all.

The single change I am proudest of is a **deletion**. An early run left the rule “fix Kerberos clock skew with `ntpdate -u <dc>`." A later one proved that cannot work — the container has no `CAP_SYS_TIME`, so `ntpdate` measures the offset and then fails to apply it — and it would have blocked certificate authentication outright. The entry was removed and replaced with a `faketime` shim that shifts one process instead of the system clock.

A loop that only accumulates gets worse over time. The interesting property is that this one can also take something out.

## What its failure looks like

The version of this post I wrote after three machines ended with an admission: none of them had defeated the loop, so I did not know what its failure looked like, and a loop you have never seen fail is a loop you do not understand.

Seven machines later I know. It failed three times, in three distinct ways, and these are the most valuable results of the weekend.

**It skipped the CVE hunt and paid for the whole run.** On one Linux box the loop enumerated well, found the service, pinned the product — and then started hand-rolling exploitation against it instead of going to look for published vulnerabilities in the version it had just pinned. The intended path was a recent CVE with a public proof-of-concept, and the delivery detail that made it work (which protocol field was the actual sink) was sitting in that PoC. The fix is now a primary step, not an afterthought: pin the precise version with a protocol-specific probe rather than an `nmap -sV` guess, then hunt CVEs and public exploits *before* improvising.

**It tunnelled on the wrong CVE in the right binary.** On the last machine of the weekend, root was a custom daemon running as root on localhost. The loop found a memory-corruption CVE in it, confirmed the overflow triggered on-target, got an address leak that defeated PIE — and spent an hour and a half designing an exploit. The primitive turned out to be a *linear-forward* write rather than an arbitrary one, which put the obvious target out of reach; public research on that CVE stops short of a working 64-bit exploit for exactly the same reason. Days of exploit development sat behind that door. Meanwhile a second, much cheaper CVE in the same daemon — an argument injection in a format string — was the intended path and is a one-liner.

That produced the rule I would hand to anyone building this kind of thing: for a pinned version, enumerate *all* of its CVEs and reason about the **set**. Rank by cost and reliability — an auth bypass or an argument injection beats a memory-corruption bug on a hardened target — consider chaining them, and exhaust public weaponised exploits before writing your own. Search tools index detection PoCs; the working exploit is often in a GitHub repository they do not index.

**It hit a genuine dead end and needed a human.** On a hardened domain controller, with every remote relay path closed — no egress, no WebClient, mandatory SMB and LDAP signing — the loop exhausted every evidenced lead and was still one technique short. That is a legitimate moment to come back to the operator, and the contract now says so explicitly: surface a *precise* status — what is confirmed, what is ruled out and why, and the specific fork you are stuck on — and accept a steer.

With one condition attached. If the steer brings in outside knowledge, the run **declares it in the ledger**, exactly as it declares recognition. The run then stops being evidence that the loop found that step on its own, and hiding it would make every other result untrustworthy. The escalation is also kept out of the shared methodology: a human-supplied technique does not get filed as a self-found pattern.

Three failures, three structural changes. That is the mechanism working, and it is worth more than the seven runs where nothing went wrong.

## Two design decisions people ask about

**Why not a Kali MCP server?** There are good ones, and I considered it. MCP gives you typed, discoverable, individually-permissioned tools. It also fixes your surface to an enumeration: you can only do what the server wrapped. Half of Nexus’s chain was shell composed on the spot — nested `git mktree` in a loop, a three-request session dance with CSRF tokens, parallel requests via `xargs -P`. No `run_nmap(target, ports)` builds a git tree. Arbitrary shell composition *is* the capability here.

The isolation people assume MCP provides comes from somewhere else anyway: the container is the boundary — the offensive toolchain and the lab VPN never touch the host — and the permission allowlist means Claude can invoke the wrapper and nothing else. MCP is an interface; the container is a fence. Worth not confusing them.

**Why not just install one of the existing plugins?** I read through them before starting. Several are excellent and far broader — one ships thirty-one skills covering command-and-control, EDR evasion, shellcode development and mobile testing. I took four ideas and rejected the rest on purpose. A lab machine needs none of that, and a repository shipping it is a different kind of artifact with different obligations. There was also a practical reason: for a tool I would demonstrate to a security team, I wanted something whose entire surface can be read in an afternoon rather than a marketplace plugin with automatic hooks.

The most interesting rejection was a “look up the write-up” capability. It would raise the completion rate immediately, and destroy the only thing being measured. If the loop cannot find the path on its own, *that* is the result I want.

## Honest limitations

**Recognition is a confound and I cannot remove it.** Several of these are well-known retired machines, and a model that has read the public internet has read their write-ups. Withholding the name is the one control I can actually enforce rather than merely trust, and declaring recognition when it happens is the other half. But “ten machines rooted” is not “ten machines discovered,” and I would not present it as such. The runs where the loop reasoned its way through a failure — a fixed access token, a reset job, a linear-forward write — are better evidence than the completion count is.

**Ten machines is still a small sample.** Two of them are AD CS variants, three are credential-reuse chains. There is a lot of the space nobody has pointed this at: no serious binary exploitation succeeded end to end, nothing with an active defender, nothing multi-host.

**The container is not reproducible.** It builds from a rolling-release base with unpinned packages, so builds are resilient — a package that disappears is logged rather than breaking the image — but not bit-for-bit repeatable. Fine for lab work. For engagement tooling behind a report someone relies on, I would pin the base image by digest and freeze the package snapshot.

**Re-running is the measurement I have barely done.** The honest test of “the loop got better” is the same machine twice, with a methodology that changed in between, recording what memory short-circuited and what stayed slow anyway. The skill supports it and the format is specified. I have one weekend of it, which is not enough to claim a trend.

[**github.com/euriconicacio/pwnloop**](https://github.com/euriconicacio/pwnloop) — MIT. Issues and forks welcome; if you run it against something that breaks it, that is the most useful thing you could send me. A target that beats the loop is worth more to me right now than another one it solves.

*Eurico Nicacio —* [*@h3llh0und*](https://github.com/euriconicacio)
