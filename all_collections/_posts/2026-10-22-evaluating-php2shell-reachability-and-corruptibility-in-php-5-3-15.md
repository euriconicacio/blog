---
layout: post
title: "Evaluating PHP2Shell reachability and corruptibility in PHP 5.3.15"
description: "Every PHP2Shell write-up gives the same floor — PHP 5.4.0 — and none say why, while the official advisories give no lower bound at all. I wanted to know whether PHP 5.3 is actually affected. It is not: the out-of-memory fault is reachable on 5.3, but the corruption that turns it into code execution lives in PHP 7+ engine internals that 5.3's HashTable does not have. Reachable is not corruptible. Reproduction: euriconicacio/php2shell-validation-toolkit."
date: "2026-10-22 15:01:34 -0300"
---

> ***Disclaimer.*** *This is independent, personal security research, carried out on my own equipment. It has no connection to any current or former employer, and no client, employer or production environment was involved at any point. I did not discover these vulnerabilities. Every test in this post ran in throwaway containers on my own machine, bound to loopback, against the official PHP source tags, the official PHP Docker images, and a PHP 5.3.15 build from the official php.net museum tarball. No exploit was written or run; the one dynamic test drives an ordinary large upload to an out-of-memory fault and observes the result. The post was published after PHP's advisories, fixed releases and CVE records were public.*

> ***A note on credit.*** *PHP's advisories credit `lcfr-hax`, `sagitzadik-wiz`, `tomash-areion` and `amyb-oai` as reporters, and `arnaud-lb`, `iluuu1994`, `bukka` and `ndossche` with the fix. The code-level description of the corruption primitive that this post builds on comes from Calif's write-up. My contribution here is only the question in the title and the way I answered it.*

The two PHP engine bugs released on 21 October — [CVE-2026-109618](https://github.com/php/php-src/security/advisories/GHSA-w8hw-w9fx-j79w) and [CVE-2026-109619](https://github.com/php/php-src/security/advisories/GHSA-6wmf-4rv5-p3rp), together nicknamed PHP2Shell — let an unauthenticated request run code as the web worker. The fix shipped in 8.5.12, 8.4.27, 8.3.36 and 8.2.35. I covered how they work and built [a toolkit](https://github.com/euriconicacio/php2shell-validation-toolkit) in [an earlier post](/php2shell-in-depth/).

This post answers a narrower question that the coverage left open: **does it reach PHP 5.3?**

The official advisories list only the supported branches, each with an upper bound and no lower bound — they say nothing about 5.x. Every secondary write-up that gives a floor gives the same one: PHP 5.4.0. Calif, whose analysis the others draw on, writes that "the earliest affected path we found dates to PHP 5.4.0" — *we found*, a boundary of their search, not a stated property of the code. None of them say why 5.4.0 and not earlier. So I took the most awkward target I could — PHP 5.3.15, from July 2012 — and checked.

The short answer is that there are two different questions hiding inside "is 5.3 affected", and they have different answers.

## Reachable is not corruptible

CVE-2026-109618 is not "the parser runs out of memory". Running out of memory is the trigger. The vulnerability is what happens next: a bailout leaves a piece of request memory half-built but still reachable, and request shutdown reads it. The bug is the corruption, not the fault that precedes it.

So "is 5.3 affected" splits in two:

- **Reachability** — can an unauthenticated request drive PHP's multipart parser to the out-of-memory fault, before any application code runs?
- **Corruptibility** — does that fault leave the specific half-built, still-reachable state that shutdown then consumes?

On PHP 5.3.15, the first is yes and the second is no.

## Reachability: yes, and it proves less than it looks

I built PHP 5.3.15 as `mod_php` on CentOS 6.10 — the stack it actually ran on — from the museum tarball (SHA-256 `f9b7256…92f1ef`, byte-for-byte identical to the `php-5.3.15` git tag in the files that matter). Alongside it I ran two official images as controls: `php:8.3.0-apache` (pre-fix) and `php:8.3.36-apache` (fixed).

Each got the same request: one ordinary, well-formed multipart text field of 7 MB, under a 2 MB `memory_limit`, so the parser crosses the limit while reading the body. No crafted payload — a large form value.

```
memory_limit = 2M, one 7 MB multipart field

5.3.15        HTTP 500  "Allowed memory size of 2097152 bytes exhausted … in Unknown on line 0"
8.3.0 (pre)   HTTP 500  "Allowed memory size of 2097152 bytes exhausted … in Unknown on line 0"
8.3.36 (fix)  HTTP 500  "Allowed memory size of 2097152 bytes exhausted … in Unknown on line 0"
```

On all three, the fault happens inside the parser — `in Unknown on line 0`, before the receiving script runs at all. So reachability holds on 5.3.15: an unauthenticated request does drive the parser to OOM before application code.

But look at the three lines again. They are identical, and one of them is the **fixed** build. The runtime test cannot tell a vulnerable PHP from a patched one, because the fix does not stop the OOM — it stops the OOM from corrupting state. Reachability is necessary for the bug and proves nothing about it on its own. On PHP 5.3, with no fix to even compare against, a reachable OOM in the parser is a fatal error: a denial-of-service path on an end-of-life runtime, not remote code execution.

To decide corruptibility you have to leave the black box and read the engine.

## Corruptibility: no, and the reason is the HashTable

The corruption primitive, from Calif's analysis, is precise. When PHP builds the `$_POST` / `$_FILES` arrays, `_zend_hash_str_add_or_update_i` increments the array's used-element counter **before** it calls `zend_string_init()` for the key. If that allocation is the one that runs the heap out of memory, the engine bails out with the counter already bumped and the bucket never filled in. Request shutdown then destroys the array, and `zend_array_destroy()` walks the bucket storage up to that counter — treating the half-built, never-initialized slot as a live value.

Every moving part of that is a PHP 7+ construct:

- a flat `arData` bucket array with inline values,
- an `nNumUsed` high-water counter that `zend_array_destroy()` walks,
- `zend_string` keys,
- `zend_array_destroy()` itself.

PHP 5.3's `HashTable` (`Zend/zend_hash.h`) is a different data structure entirely:

```c
/* php-5.3.15, Zend/zend_hash.h */
typedef struct _hashtable {
    uint     nNumOfElements;        /* count — but no nNumUsed high-water mark */
    Bucket  *pListHead, *pListTail; /* a doubly linked list of buckets */
    Bucket **arBuckets;             /* hash chains — no flat arData array */
    /* … */
} HashTable;
```

```c
/* php-8.3.36, Zend/zend_types.h */
struct _zend_array {
    /* … */
    Bucket  *arData;      /* flat bucket array, walked up to nNumUsed */
    uint32_t nNumUsed;    /* the primitive lives here */
};
```

A PHP 5.3 array is a linked list of **separately allocated** `Bucket` nodes. There is no `nNumUsed`, no flat `arData`, no `zend_string`, and the destructor (`zend_hash_destroy`, not `zend_array_destroy`) walks `pListHead`, not a counter. More to the point, PHP 5.3 allocates a bucket in full and *then* links it into the table — if that allocation OOMs, nothing has been published, so there is nothing for shutdown to walk into. The "counter bumped before the value is set, destructor walks the dead slot" primitive cannot form there. The exact thing the fix guards against does not exist in 5.3's memory model.

And this is not a quirk of 5.3.15. The flat `arData`/`nNumUsed` HashTable is the PHP 7.0 "phpng" rewrite; it appears at exactly `php-7.0.0` and nowhere earlier. Walking the struct forward, tag by tag:

| PHP tag | HashTable | `nNumUsed` / `arData` | `zend_array_destroy` |
|---|---|---|---|
| 5.3.15 | `_hashtable` (linked list) | no | no |
| 5.4.0 · 5.4.45 | `_hashtable` | no | no |
| 5.5.38 | `_hashtable` | no | no |
| 5.6.0 · 5.6.40 (last 5.6) | `_hashtable` | no | no |
| **7.0.0** | `_zend_array` | **yes** | **yes** |
| 7.0.33 · 8.3.36 | `_zend_array` | yes | yes |

Both halves of the primitive — the flat array the destructor walks and `zend_array_destroy()` itself — arrive together at 7.0.0. The entire PHP 5 branch, through the final 5.6.40, lacks the substrate the bug is built on.

CVE-2026-109619 is even shorter to dismiss. Its main pre-authentication trigger is the `session.upload_progress` handler, which runs session code during request startup. That feature was [introduced in PHP 5.4.0](https://wiki.php.net/rfc/session_upload_progress); it does not exist in 5.3, and a search of the 5.3.15 source finds zero occurrences of it. The other confirmed route, fatal-error backtraces, is a PHP 8.5 feature.

## So why does everyone say 5.4.0?

Because the floor got copied off the wrong mechanism. Calif's write-up — the source the others draw on — gives *two* floors, for two different paths. For the hash-table corruption it says the RCE works against "mod_php and **PHP >= 7.3**". For the `session.upload_progress` path it says "mod_php and **PHP >= 5.4.0**". The 5.4.0 that propagated into "affects every PHP since 5.4" is the floor for the *upload-progress* path — CVE-2026-109619 — pinned to the version its trigger was introduced. It was never Calif's floor for the hash-table RCE, which they put two majors higher, at 7.3. The secondary write-ups flattened the two into one headline.

The official records don't disagree so much as decline to say. PHP's advisories list only the supported branches, each with an upper bound and no lower bound. NVD then translates that into a CPE range whose oldest entry is `cpe:…:php:php:*` up to `< 8.2.35` with **no `versionStartIncluding`** — which a scanner reads as "every PHP below 8.2.35", and dutifully flags PHP 5.3. That is a mechanical artifact of a branch-only advisory, not a finding that 5.x was tested. If a scanner tells you your legacy PHP 5 host is exposed to CVE-2026-109618, this unbounded range is why. *(NVD has since added lower bounds to both records; see the update at the end.)*

I examined 5.3.15 end-to-end and the HashTable struct across every PHP 5 release; I did not run 5.4–5.6 in a lab, but the struct is identical across all of them, so the substrate argument carries the whole branch. What I have not done — and what nobody has shown — is a *different* OOM-bailout path reachable in PHP 5.x. The general property it would need (a fatal `longjmp` that skips cleanup) does exist there, so I will say "not affected by the known vector", not "immune".

## What this leaves

- **CVE-2026-109618** — not affected by the known vector, and not only on 5.3.15: the corruption primitive is the PHP 7.0 HashTable, which no PHP 5.x release has. The OOM is reachable, but on PHP 5 that reachable fault is a denial-of-service surface on end-of-life software, not RCE.
- **CVE-2026-109619** — not affected on 5.3. Its pre-auth triggers (`session.upload_progress`, 5.4+; fatal-error backtraces, 8.5) do not exist here.

None of this is a reason to keep running PHP 5. It went end-of-life years ago and carries a decade of other unpatched bugs; the right answer is still to migrate. It is only a reason not to spend an incident on this particular pair, and not to trust a scanner that flags it off an unbounded version range. "The vulnerable code pattern looks present" and "the vulnerability is present" are not the same claim. Reachable is not corruptible.

The container build, the three-way differential, and the source comparison are in the toolkit, under [`lab/reproduction-php-5.3.15/`](https://github.com/euriconicacio/php2shell-validation-toolkit), so you can run the test and read the two struct definitions side by side rather than take my word for it.

## Update, 23 October: NVD added the floors

The afternoon this post went out, I sent NVD the struct comparison and the session-module evidence and asked for a lower bound on each record. They revised both CPE configurations on 23 October:

- **[CVE-2026-109619](https://nvd.nist.gov/vuln/detail/CVE-2026-109619)**: the oldest range is now `>= 5.4.0, < 8.2.35`. That is the floor I proposed, the release that introduced `session.upload_progress`.
- **[CVE-2026-109618](https://nvd.nist.gov/vuln/detail/CVE-2026-109618)**: the oldest range is now `>= 7.3.0, < 8.2.35`. I had asked for 7.0.0, the structural floor below which the primitive cannot exist. NVD went to 7.3.0, which matches Calif's scope for the hash-table RCE and [CrowdStrike's dating](https://www.crowdstrike.com/en-us/blog/crowdstrike-protects-customers-from-php-rce-zero-days/) of the exact ordering to an October 2017 change first shipped in 7.3.0, [raised on oss-security](https://www.openwall.com/lists/oss-security/2026/10/23/5) the same morning. My analysis rules out everything below 7.0 and says nothing about 7.0 to 7.2, so the two agree, and theirs is the tighter bound.

A scanner working from current NVD data should no longer flag PHP 5.x for either CVE, or PHP 7.0 to 7.2 for CVE-2026-109618. If yours still does, its feed predates the change.

*Eurico Nicacio —* [*@h3llh0und*](https://github.com/euriconicacio)
