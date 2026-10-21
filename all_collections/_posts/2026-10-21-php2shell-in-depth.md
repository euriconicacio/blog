---
layout: post
title: "PHP2Shell in depth: two PHP engine bugs, the patch that fixed them, and a toolkit to check your servers"
description: "On 21 October PHP fixed two engine bugs that let a request with no login run code as the web worker. I did not find them. This post covers how they work, using PHP's own advisories and the patch itself, and what the fix changed in how PHP handles running out of memory. It also introduces the open-source toolkit I built to check servers and protect them until they can be upgraded. Toolkit: euriconicacio/php2shell-validation-toolkit"
date: "2026-10-21 18:52:00 -0300"
image: /assets/images/posts/php2shell-in-depth/01.png
---

> ***Disclaimer.*** *This is independent, personal security research, carried out on my own equipment. It has no connection to any current or former employer, and no client, employer or production environment was involved at any point. I did not discover these vulnerabilities. My contribution is the analysis of PHP's advisories and patch, and the open-source toolkit described here. Every test in this post ran in throwaway containers on my own machine, bound to loopback, against the official PHP Docker images and an Ubuntu 24.04 package. No exploit was written or run. The post was published after PHP's advisories, fixed releases and CVE records were public.*

> ***A note on credit.*** *PHP's advisories credit `lcfr-hax`, `sagitzadik-wiz`, `tomash-areion` and `amyb-oai` as reporters. They credit `arnaud-lb`, `iluuu1994`, `bukka` and `ndossche` with the fix. The toolkit contains no exploit, and it is meant only for systems you own or are authorized in writing to assess.*

![PHP2Shell Validation & Defense Toolkit]({{ site.baseurl }}/assets/images/posts/php2shell-in-depth/01.png)

On 20 October, two CVE identifiers for the PHP engine, CVE-2026-109618 and CVE-2026-109619, began circulating before any fix was public. PHP moved its security release forward by a day. On 21 October it shipped PHP 8.5.12, 8.4.27, 8.3.36 and 8.2.35, along with two advisories, each scored 9.2 Critical.

Cloudflare reported early attacks before the releases went out, and watchTowr reported exploitation attempts against both bugs. At release time, PHP said that none of the proofs of concept it had inspected since the leak actually worked.

The bugs are in the interpreter, not in any application. Updating WordPress, Drupal, Magento or a plugin changes nothing, and neither does putting an application into maintenance mode. The only fix is a PHP build that contains the patch.

This post goes through the two bugs, what the patch changed, and the [toolkit](https://github.com/euriconicacio/php2shell-validation-toolkit) I built to check servers and protect them until they can be upgraded.

## Some background: how PHP serves a request

Three properties of the PHP engine explain both bugs.

**Workers outlive requests.** Apache's `mod_php`, PHP-FPM and FrankenPHP keep worker processes alive and hand each one request after request. Whatever a worker keeps between requests is shared, over time, by everyone whose requests it serves.

**Request memory is thrown away in bulk.** During a request, PHP allocates from a per-request heap governed by `memory_limit`. When the request ends, that heap is released wholesale. Anything that still points into it afterwards is a dangling pointer.

**Fatal errors do not unwind.** When PHP hits a fatal error, including running out of memory, it does not return step by step through the functions in progress. It performs a "bailout": a `longjmp()` straight back to the top of the request. None of the cleanup that those intermediate functions would have done gets to run.

Each of these is a reasonable design choice. Together they allow two failures. A bailout can leave memory half-built and still reachable, which is CVE-2026-109618. Or it can leave a worker's global pointing into a heap that is about to be released, which is CVE-2026-109619.

## CVE-2026-109618: running out of memory at the wrong moment

Some code paths make request memory reachable before they have finished initializing it. If any allocation fails during that window, even an unrelated one, the bailout leaves the memory half-built. PHP then runs its shutdown procedure, which reads that memory and makes decisions based on it.

To use this, an attacker needs the request to run out of memory at that exact point, while staying inside PHP's configured limits. The advisory lists three ways to do it through multipart parsing:

- A multipart request with no usable declared length can pass the `post_max_size` check and keep sending body data.
- A single part can carry any number of headers, and PHP's internal representation of them takes far more memory than the bytes on the wire.
- Empty file fields do not count toward `max_file_uploads`, but they still build the usual file arrays, and nested field names multiply that work.

Under Apache's `mod_php`, all of this happens while PHP parses the request body, before any application code runs. `mod_php` also runs the shutdown procedure even when request startup has failed. PHP validated full remote code execution over HTTP against `mod_php` with the default input and memory limits.

## CVE-2026-109619: one request's leftovers, read by the next

A worker keeps global state between requests: error state, session state, object state and extension state. Some of it points into the current request's heap, and it has to be reset before that heap is released. Two things can break this:

- A fatal error can skip the reset.
- A warning raised late in shutdown can fill the state in again after it was cleared.

Either way, the heap is released while a global still points into it. If the same worker's next request touches that global before its normal initialization, it uses freed memory.

The attack needs two requests on the same worker. The first leaves the stale pointer behind (PHP calls this the producer), and the second reads it (the consumer). PHP confirms two concrete cases:

- **Fatal-error backtraces in PHP 8.5.** With `fatal_error_backtraces` on, the backtrace stored for a fatal error is never reset during shutdown. A warning in the next request reaches it first. This is the case behind PHP's confirmed `mod_php` exploit.
- **Session upload progress.** The upload-progress feature runs session code during request startup, before the session module's normal initialization. If a request is cut off, for example by running out of memory, it leaves session and decoder state pointing into the released heap, and the next upload-progress request picks it up. The feature is enabled by default and works without the application ever calling `session_start()`. PHP validated this path against both `mod_php` and FPM.

The warning that consumes stale error state can come from configuration applied per request before initialization:
- under `mod_php`, `.htaccess` files and Apache `php_value` directives;
- under FPM, `PHP_VALUE` parameters that the front end sends.

Deprecated or invalid settings are enough to trigger it. The read happens before PHP decides whether to report the warning, so `@`, `error_reporting` and `display_errors` do not help. PHP lists memory disclosure, as well as code execution, as a confirmed impact of this bug.

Several write-ups published before PHP's advisories described the pair differently; for example, some assigned the upload-progress bug to CVE-2026-109618. This post follows PHP's advisories, because PHP is the CNA that issued both identifiers.

## Not every server is exposed the same way

How exposed a server is depends on how PHP runs, because a crash kills a different amount of state under each model. This table summarizes what the two advisories say.

| How PHP runs | CVE-2026-109618 | CVE-2026-109619 |
| --- | --- | --- |
| Apache `mod_php` | Confirmed code execution with default limits, before any application code runs | Confirmed through both the 8.5 backtrace case and the session case |
| PHP-FPM | Susceptible by source analysis after startup; no complete exploit demonstrated. FPM kills a worker whose startup fails, so the startup path does not carry over | Session case validated, but it needs a suitable operation in the application. The configuration consumer applies only if the front end sends `PHP_VALUE` |
| FrankenPHP | Likely susceptible: it runs shutdown unconditionally, as `mod_php` does | No early configuration consumer established in the default setup; others may exist |
| LSAPI | Behaves like FPM; attacks through the application likely, unvalidated | A conditional candidate, through environment settings applied early |

Running many workers does not prevent the attack. The attacker's two requests have to land on the same worker, so more workers mean more attempts, not a barrier.

## What the patch changed

On the 8.5 branch, both bugs are fixed by a single commit, [`31c0ea7d7`](https://github.com/php/php-src/commit/31c0ea7d7ff1d1fccb6492e52dab544011628065), titled "Avoid state corruption via bailout". It was written by Arnaud Le Blanc together with Ilija Tovilo, Jakub Zelenka, Nora Dossche and Nicolas Grekas, and it touches 423 files. Of those, 306 are tests, most of them running a single function out of memory on purpose. The design change is in the allocator.

**Running out of memory no longer jumps.** When an allocation would cross `memory_limit`, the allocator now records a pending out-of-memory error (`EG(delayed_oom)`) and asks the virtual machine to stop at its next safe point. It raises the error there, once the code that was running has finished its work. Code that runs outside the virtual machine, such as the multipart parser in `main/rfc1867.c`, now checks for a pending error (`ZEND_HAS_OOM()`) after each step and backs out on its own. New fallible allocation functions return `NULL` instead of bailing out. This is what closes CVE-2026-109618: the jump that left memory half-built is gone.

**A second, higher limit ends the process.** A delayed error means a request can briefly use more than `memory_limit`, so the patch adds a hard ceiling. The new setting, `memory_hard_limit_percent`, defaults to twice `memory_limit`, or `memory_limit` plus 1 GB when the limit is above 1 GB. A process that reaches the ceiling writes `Hard memory limit of … bytes exhausted` to standard error and exits immediately with status 137. It does not unwind and it does not run shutdown.

**Shutdown leaves nothing behind for the next request.** After it releases the request heap, shutdown now clears the last error, marks the superglobals as empty and marks the modules as inactive, with the comment "May be accessed before php_request_startup()". The session module resets its upload-progress flag as well. That takes away what the consumers of CVE-2026-109619 were reading. As extra hardening, while the upload-progress handler decodes a session during startup, objects in that session are kept as `__PHP_Incomplete_Class`. No autoloading runs and no `__wakeup()` or `__unserialize()` is called.

The hard limit gives defenders something to watch for after upgrading. On a patched server, a `Hard memory limit … exhausted` line or a worker exiting with status 137 means some request pushed PHP far past its memory limit. That is worth looking at while attacks against unpatched servers are still common.

## Affected and fixed versions

| Branch | Fixed in |
| --- | --- |
| 8.5 | 8.5.12 |
| 8.4 | 8.4.27 |
| 8.3 | 8.3.36 |
| 8.2 | 8.2.35 |
| 7.4 to 8.1 | No release. PHP's security team published minimal source backports for people who build PHP themselves, and distributions received the patches in advance. |
| 5.4 to 7.3 | No fix from PHP. Move to a supported branch, or ask your distribution or LTS vendor. |

Calif's write-up traces the earliest affected code to PHP 5.4.0. Packages from third-party repositories may lag behind the upstream releases. Confirm that the package you install actually contains the fix rather than assuming it does.

## The toolkit

The [php2shell-validation-toolkit](https://github.com/euriconicacio/php2shell-validation-toolkit) is for the hours between a disclosure like this and the moment every server is upgraded. It has four parts:

- **Exposure checks.** A checker for a single server, a scanner for a fleet, and an active, non-destructive scanner for a single target. None of them sends exploit payloads.
- **Mitigations.** A configuration quick fix, plus edge rules for nginx, Apache (ModSecurity and `.htaccess`), lighttpd, FrankenPHP and Cloudflare.
- **Detection.** Process, log and network signals that show exploitation.
- **A lab.** An isolated Docker target running an unpatched PHP, cut off from the internet. You can use it to confirm that the checker reports it as vulnerable, that your edge rules block the trigger and that your detection fires. Then you switch the image to a patched tag and watch the checker report it as patched.

### Step 1: find out whether you are exposed

On the server itself, the checker can give a definitive answer, thanks to the patch. The new `memory_hard_limit_percent` setting exists only in builds that carry the fix:

- it appears in `main/main.c` at the `php-8.2.35`, `php-8.3.36`, `php-8.4.27` and `php-8.5.12` tags, and not before them;
- on the official Docker images, `ini_get("memory_hard_limit_percent")` returns `false` on 8.3.35 and 8.5.11, and an empty string on 8.3.36 and 8.5.12.

So the checker compares the version with the fixed releases, and it also looks for that setting. This matters because distributions usually backport a fix without changing the upstream version number, so a version comparison alone would flag a fixed package as vulnerable. On Ubuntu 24.04, for example, the patched `php8.3-cli` package, `8.3.6-0ubuntu0.24.04.13`, still reports PHP 8.3.6. A version check flags it, but it has the new setting, and the checker reports it as patched. The checker also audits the three settings PHP's stopgaps rely on.

```bash
# The local PHP install: version and configuration
python3 exposure/check_php2shell.py

# Judge a specific version string
python3 exposure/check_php2shell.py --version 8.3.12

# Header-only fingerprint of a host you are authorized to assess
python3 exposure/check_php2shell.py --url https://your-host.example

# A whole fleet, one host per line, header-only
python3 exposure/scan_fleet.py hosts.txt
```

The remote mode reads only the version a host advertises in its response headers. A host that hides its version comes back as `UNKNOWN`. That does not mean it is safe; it means the version has to be checked on the server itself. Over HTTP, the version is the only safe signal for PHP2Shell, and it cannot see backports. Confirming the bugs remotely would mean driving PHP out of memory or into a use-after-free, and that is exploitation.

### A note on "PHP2Shell confirmed"

Some public tools report a target as confirmed vulnerable by doing the following:
- writing text into a session file through the upload-progress field;
- reading that text back through a file-inclusion endpoint.

That is an older technique, and it is not either of these CVEs. It needs three things:
- a file-inclusion bug in the application;
- `session.use_strict_mode` off, so that PHP accepts a session id the attacker made up;
- `session.upload_progress.cleanup` off, so that the data survives the end of the upload.

The default for that last setting is on. The toolkit's active scanner, `exposure/php2shell_scan.py`, keeps this test, labelled as what it is. It also checks something more useful while you wait for the upgrade: whether your edge rule actually stops the upload-progress field before it reaches PHP.

```bash
python3 exposure/php2shell_scan.py --authorized --target http://host-you-own:8080 \
    --session-path /index.php
```

On a throwaway `php:8.3.0-apache` container, the scanner reports three things. PHP2Shell shows as vulnerable by version. The edge-rule check shows the upload-progress field reaching PHP until the toolkit's `.htaccess` rule is in place, and as blocked afterwards. The older chain shows its precondition, strict mode off. With `cleanup` off and a test page that includes files, the scanner's inert marker comes back through the inclusion.

### Step 2: stopgaps until the upgrade lands

PHP is explicit that upgrading is the only complete fix and that no configuration change makes a server safe. Each of the settings below removes one known route. Set them administratively, in `php.ini` or the FPM pool, so applications cannot override them. Restart every PHP worker afterwards.

| Setting | What it removes | What it costs |
| --- | --- | --- |
| `session.upload_progress.enabled = 0` | The session case of CVE-2026-109619 | Upload progress bars, which few applications use |
| `fatal_error_backtraces = 0` (PHP 8.5 only) | The backtrace case of CVE-2026-109619 | Backtraces in fatal-error logs |
| `memory_limit = -1` | The controllable out-of-memory condition behind CVE-2026-109618 | A single request can now use all of the server's memory. Use a container or cgroup limit in its place, and restore `memory_limit` after upgrading |
| If the application does not need `multipart/form-data`, block it at the front end; otherwise cap body size, parts and headers per part | The multipart routes into CVE-2026-109618 | File uploads, if the application has them |

PHP's advisories add two warnings. First, do not cap memory with `RLIMIT_AS` or `RLIMIT_DATA`. When they make an allocation fail, PHP takes the same `longjmp()` path the bug relies on. Second, even with `memory_limit = -1`, a failed underlying allocation can still reach that path. That is why this setting is a mitigation and not a fix.

The quick-fix script applies the first row:

```bash
sudo mitigations/quick-fix/run.sh --check     # report current state
sudo mitigations/quick-fix/run.sh --dry-run   # preview the change
sudo mitigations/quick-fix/run.sh             # apply
sudo mitigations/quick-fix/run.sh --revert    # undo
```

The other rows are one line each. Decide on `memory_limit` deliberately, because it trades an exploitable condition for a denial-of-service risk.

The edge rules block multipart uploads and the `PHP_SESSION_UPLOAD_PROGRESS` field. Do not filter on the `PHPSESSID` cookie: neither bug needs an existing session.

### Step 3: look for exploitation

Both bugs end with code running as the web worker. The clearest sign of exploitation is a PHP worker starting a process it never should, such as a shell, an interpreter or `curl`. The toolkit includes:
- an auditd rule keyed on the worker's user;
- command-line patterns for reverse shells;
- an illustrative Suricata rule for the upload-progress field.

Treat a worker that opens a shell right after a multipart POST as a compromise.

PHP's advisory for CVE-2026-109619 also recommends a log audit. Look for:
- warnings repeated before scripts run;
- diagnostics during shutdown;
- `Maximum execution time exceeded`;
- fatal errors on upload, import and error-page routes.

Fixing those errors removes links in the chain. But a clean log is not proof of safety, because the failure can happen before PHP writes anything. After you upgrade, add the hard-limit signal from the section above.

## Why the toolkit has no exploit

Code that claims to exploit these bugs is already circulating. The most widely shared repository ships three exploits, and none of them matches the advisories:
- one is the older file-inclusion chain described above;
- one needs the application to call `unserialize()` on attacker data;
- one needs two directories configured with different session serializers.

The two chains that do match the advisories are behind a paid subscription. I deliberately left any exploit out of the toolkit. These bugs need no login, sit in software that runs a large share of the web, face active exploitation attempts, and the patches are only hours old. A working exploit released to everyone helps attackers far more than it helps anyone checking their own servers. Checking your own servers does not need one: the version, the configuration, the edge rules and the detection signals answer the question.

If the checker reports a version below the fixed release, everything in Steps 2 and 3 only buys time. Upgrade PHP.

---

**Toolkit** [euriconicacio/php2shell-validation-toolkit](https://github.com/euriconicacio/php2shell-validation-toolkit) · **Fix** [`31c0ea7d7`](https://github.com/php/php-src/commit/31c0ea7d7ff1d1fccb6492e52dab544011628065) · **Fixed in** 8.2.35, 8.3.36, 8.4.27, 8.5.12

**Advisory** [GHSA-w8hw-w9fx-j79w](https://github.com/php/php-src/security/advisories/GHSA-w8hw-w9fx-j79w) · **CVE** [CVE-2026-109618](https://www.cve.org/CVERecord?id=CVE-2026-109618) · out-of-memory corruption · **CVSS** 9.2 Critical (v4.0)

**Advisory** [GHSA-6wmf-4rv5-p3rp](https://github.com/php/php-src/security/advisories/GHSA-6wmf-4rv5-p3rp) · **CVE** [CVE-2026-109619](https://www.cve.org/CVERecord?id=CVE-2026-109619) · stale global state · **CVSS** 9.2 Critical (v4.0)

Identifiers leaked 20 October · advisories, fixed releases and CVE records published 21 October · further reading: [Tenable's FAQ](https://www.tenable.com/blog/php2shell-cve-2026-109618-cve-2026-109619-frequently-asked-questions-about-critical-php-remote), [Sansec](https://sansec.io/research/php2shell).

*Eurico Nicacio —* [*@h3llh0und*](https://github.com/euriconicacio)
