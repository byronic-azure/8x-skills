# Incident Advisory — Generic.PHP.WebShell.X.635F263C

**PYRACLAW Cybersecurity Division**
Ref: PCD-2026-0825-01 · Opened 2026-08-25 · Analyst: Claude Code (automated triage)

---

## Verdict

**No breach. Benign test fixture, correctly detected, wrong context.**

Bitdefender's detection is a true positive against the *file* and a false
positive against the *situation*. The flagged file is a deliberately
vulnerable PHP sample that ships inside a security-scanner package's own test
suite. It was unpacked into `%TEMP%` by a package install or build step and
scanned on write.

Confidence: **High**, from path structure and signature class.
Not **Certain** — see *Residual risk* — host artifacts are required to close.

No evidence of malicious assault was found. No indicators of compromise, no
attacker-controlled infrastructure, no persistence mechanism, and no
credential-access activity are present in the material reviewed.

---

## The detection

```
C:\Users\byron\AppData\Local\Temp\
  backend-plugin-jxJ3Je\
    packages\core\tests\fixtures\
      check-security-scan\
        ported-database-and-command-risks\
          public\plugin.php
```

Signature: `Generic.PHP.WebShell.X.635F263C` — not cleaned.

## Why the path refutes the breach hypothesis

Six independent signals, all from the path itself:

1. **`backend-plugin-jxJ3Je`** — the six-character random suffix is the
   signature of Node's `fs.mkdtemp()`. This is an ephemeral extraction
   directory created by tooling, not a location an attacker chooses.
2. **`packages/core/`** — npm/pnpm workspace layout. The file arrived as part
   of a structured package, not as a dropped payload.
3. **`tests/fixtures/`** — the file is test data by construction.
4. **`check-security-scan/`** — the fixture belongs to the test suite of a
   *security-scanning feature*.
5. **`ported-database-and-command-risks/`** — the directory is named after the
   risk classes it deliberately contains: SQL injection and command
   execution. This is a positive-control sample. A scanner that does not
   detect it is broken.
6. **`public/plugin.php`** — the fixture simulates a web root so the
   scanner under test sees a realistic layout.

A security scanner's test corpus must contain real malicious patterns to be
testable. Antivirus engines scanning that corpus will flag it. This collision
is routine and expected.

## Signature analysis

`Generic.PHP.WebShell.X.635F263C` is a **generic heuristic cluster
identifier**, not a family attribution. Bitdefender assigns no family name
here — no China Chopper, WSO, b374k, or r57. The match is on *shape*: an
execution primitive (`eval`, `system`, `passthru`, `shell_exec`) reachable
from request input (`$_GET`, `$_POST`, `$_REQUEST`). That shape is precisely
what the fixture must contain to be a valid positive control.

## "Could not clean" is expected, not alarming

Cleaning means excising malicious code from an otherwise-legitimate host file
and restoring the remainder. Here the file *is* the pattern — there is no
clean remainder to restore. Bitdefender's correct fallback is quarantine or
delete, and that is what it did. This is not a sign of an entrenched,
resistant, or self-repairing threat.

## Counter-indicators to a live webshell

- **The location is not web-served.** `%LOCALAPPDATA%\Temp` is not a document
  root on a default Windows workstation. A PHP webshell is inert text unless
  both a PHP interpreter and an HTTP server mapping that directory are
  present.
- **No persistence surface.** `mkdtemp` directories are transient and are
  reclaimed. Attackers seeking persistence do not stage there.
- **Wrong target for a real intrusion.** A webshell placed by an intruder
  goes into a reachable docroot — `C:\inetpub\wwwroot`, an `htdocs` tree, a
  container volume — where it can be requested over HTTP.

---

## Residual risk — what this triage cannot rule out

Path analysis establishes provenance, not integrity. Two scenarios remain
open and are worth ten minutes to close:

- **Supply-chain compromise.** The fixture tree is genuine, but a tampered
  package version added a *functional* shell alongside it. Distinguishable by
  hashing the file against the upstream repository.
- **Untrusted package origin.** The `backend-plugin` tarball may not have come
  from the registry you believe it did.

Verify. Do not assume.

---

## Verification — run on the affected host

PowerShell, as the affected user. Steps 1–3 identify the package; 4–5 confirm
the file is inert; 6 closes out.

**1. Locate the extraction directory and its timestamp.**
```powershell
Get-ChildItem "$env:LOCALAPPDATA\Temp" -Filter "backend-plugin-*" -Directory |
  Select-Object FullName, CreationTime, LastWriteTime
```
Correlate `CreationTime` against what you were doing ten hours before the
alert — an install, a build, a plugin load. A match closes most of the doubt.

**2. Identify the owning package.**
```powershell
Get-ChildItem "$env:LOCALAPPDATA\Temp\backend-plugin-*" -Recurse -Filter package.json -Depth 3 |
  ForEach-Object { Get-Content $_.FullName -Raw | ConvertFrom-Json |
  Select-Object name, version, repository }
```

**3. Hash the flagged file and compare with upstream.**
```powershell
Get-FileHash "<full path to plugin.php>" -Algorithm SHA256
```
Retrieve the same fixture from the package's public repository at the same
version and compare. Equal digests close the supply-chain question outright.

**4. Confirm no PHP execution capability exists on the host.**
```powershell
Get-Command php -ErrorAction SilentlyContinue
Get-Service W3SVC, Apache*, nginx* -ErrorAction SilentlyContinue
Get-NetTCPConnection -State Listen |
  Where-Object LocalPort -in 80,443,8000,8080,8888
```
Empty results mean the file could never have executed. This is the single
strongest confirmation available.

**5. Check for persistence created around the alert window.**
```powershell
$since = (Get-Date).AddDays(-2)
Get-ScheduledTask | Where-Object { $_.Date -gt $since } |
  Select-Object TaskName, TaskPath, Date
Get-CimInstance Win32_StartupCommand | Select-Object Name, Command, Location
```

**6. Close out with a full offline scan.**
Bitdefender → Protection → Antivirus → **Rescue Environment**. This scans
outside the running OS and settles any question of an active resident threat.

---

## Protective measures

### Endpoint — immediate

- **Leave the file quarantined.** There is no operational reason to restore it.
  If the package's test suite later fails on the missing fixture, that is a
  clean signal and the right time to act — not now.
- **Do not exclude `%TEMP%` from scanning.** This is the standard
  overcorrection and it is dangerous: `%TEMP%` is a primary staging directory
  for real malware. If an exclusion proves necessary after step 3 verifies the
  package, scope it to the narrowest possible path pattern for that fixture
  directory, and document it with an expiry date.
- **Submit the sample to Bitdefender as a false positive** once step 3 confirms
  the hash matches upstream. This removes the recurrence for every developer
  using that package.

### Build pipeline

- **Pin and verify.** Commit the lockfile; enable `npm ci` in CI so installs are
  reproducible. Verify package integrity hashes rather than trusting version
  ranges.
- **Move package installs and builds off the workstation** where practical.
  A container or CI runner both removes the AV collision and reduces the blast
  radius of any genuine supply-chain event.
- **Expect this class of alert again.** Any security-tooling dependency with a
  fixture corpus will trip endpoint AV. Record this advisory so the next
  occurrence is triaged in two minutes, not two hours.

### Conditional — only if verification fails

If step 3 shows a hash mismatch against upstream, or step 4 finds a PHP
interpreter and a listening web server, treat the host as suspect and
escalate: isolate it from the network, then rotate `CLOUDFLARE_API_TOKEN`,
`VAULT_SIGNING_KEY`, and any ORCID/Zenodo credentials that have been present
in that host's environment. Do not rotate pre-emptively — an unnecessary
`VAULT_SIGNING_KEY` rotation breaks every signed client until they re-key.

---

## Secondary review — PyraClaw edge stack

The five submitted artifacts were reviewed for malicious content as part of
this triage.

**None contain malicious code.** No obfuscation, no dynamic evaluation, no
outbound exfiltration primitives, no encoded payloads. The only match in a
dangerous-primitive sweep was the literal string `"curl/7.1"` inside a WAF
bad-bot denylist — a detection pattern, not an invocation. These files are
not the vector and are unrelated to the detection.

The review did surface defensive weaknesses worth fixing on their own merits.
Ordered by severity.

### 1. Local secrets are indexed into the RAG store and sent outbound — HIGH

`rag_system.py` · `LocalVectorStore.index_directory`

`supported` includes `.toml`, `.json`, `.yaml`, `.yml`, and the only skip rule
is `part.startswith(".")`. So `.env` is excluded incidentally, but
`wrangler.toml` (which can carry a `[vars]` block), `secrets.json`,
`credentials.json`, and service-account keys under `H:\pyraclaw` are all
chunked and keyword-indexed. `get_context()` then returns those chunks
verbatim, and `augment_task()` prepends them to a prompt sent to the Claude
API.

That is a live path from on-disk secrets to an outbound request. It is
accidental, not malicious, and it is the highest-value fix in this batch.

Remediate in `index_directory`, before `add_document`:

- Deny by filename: `wrangler.toml`, `*secret*`, `*credential*`, `*token*`,
  `*.pem`, `*.key`, `*.pfx`, `id_rsa*`, `*.tfvars`.
- Run a secret-pattern regex over every chunk and drop or redact matches —
  cover AWS keys, `ghp_`/`github_pat_`, Cloudflare tokens, PEM blocks, and
  generic high-entropy `KEY=`/`SECRET=`/`TOKEN=` assignments.
- Honour `.gitignore` when walking. What is not fit to commit is not fit to
  index.

### 2. SQLi rule will block legitimate traffic — HIGH

`waf_rules.py` · `_sqli_protection_rule`

The pattern list is matched with `contains` against `http.request.uri` and
includes `--`, `0x`, `CAST(`, `CHAR(`, and `EXEC(`. `--` occurs in ordinary
slugs (`/investor-brief--q3`); `0x` occurs in any base64 or hex query
parameter. Deployed at `action="block"`, this is a self-inflicted outage on
the investor homepage.

Cloudflare's Managed Ruleset already performs tokenizing SQLi detection that
does not rely on substring matching. Drop the naive list, keep the managed
ruleset. If a custom rule is still wanted, scope it to named parameters, use
anchored `matches` regexes over `lower(...)`, and run it at `action="log"`
for seven days before promoting it to `block`.

### 3. Encoded traversal patterns can never match — MEDIUM

`waf_rules.py` · `_path_traversal_rule`

`%2e%2e%2f`, `%2e%2e/`, `..%2f`, and `%2e%2e%5c` are matched against
`http.request.uri.path`, but Cloudflare normalizes and percent-decodes that
field before rule evaluation. Those five patterns are dead code, and the rule
gives false assurance against exactly the evasion it was written to stop.

Match encoded forms against the raw `http.request.full_uri`; keep the decoded
`../` check on `http.request.uri.path`.

### 4. HMAC rule proves presence, not authenticity — MEDIUM

`waf_rules.py` · `_hmac_validation_rule`

The docstring is honest about this; the operational consequence deserves
stating. `X-Pyraclaw-Signature: x` passes the edge. All real assurance sits in
the Worker, so the Worker must: compare in constant time, require a companion
`X-Pyraclaw-Timestamp`, reject clock skew beyond 300s, and track a nonce
window to prevent replay. Without the timestamp and nonce, a captured valid
request is replayable indefinitely.

### 5. COEP breaks fonts on the public homepage — MEDIUM

`security_headers.py` · `_BASE_HEADERS`

`Cross-Origin-Embedder-Policy: require-corp` is applied to every profile,
while the `INVESTOR_HOMEPAGE` CSP permits `fonts.googleapis.com` and
`fonts.gstatic.com`. Under `require-corp`, those cross-origin subresources are
blocked unless they carry an explicit CORP opt-in and are requested with
`crossorigin`. The likely production symptom is fonts silently failing on the
investor-facing page.

Use `credentialless` for the homepage profile, or self-host the font files —
which is also better for latency and removes a third-party privacy dependency.

### 6. `'unsafe-inline'` guts the homepage CSP — MEDIUM

`security_headers.py` · `_CSP_DIRECTIVES["INVESTOR_HOMEPAGE"]`

`script-src 'self' 'unsafe-inline'` re-permits exactly the injection class the
CSP exists to prevent. Generate a per-request nonce in the Worker and rewrite
the script tags, or move to hashes for the handful of inline blocks. The
strict profiles (`API_ENDPOINT`, `VAULT_ENDPOINT`) are correctly configured.

### 7. `X-XSS-Protection` should be `0` — LOW

`security_headers.py` · `_BASE_HEADERS`

The legacy XSS auditor is removed from Chrome and Edge, and its filter
historically introduced exploitable side channels. Current guidance is to send
`X-XSS-Protection: 0` explicitly and rely on CSP.

### 8. Signing material exposed via shell history and environment — MEDIUM

`deploy_all.sh` · documented usage, lines 28–30

`CLOUDFLARE_API_TOKEN=xxx VAULT_SIGNING_KEY=yyy bash deploy_all.sh` places
long-lived signing material into shell history and into the process
environment, where any process running as that user can read it. Given this
advisory concerns that same class of workstation, the exposure is worth
closing.

- Source secrets from a `chmod 600` env file held outside the history path, or
  from Windows Credential Manager.
- Scope the Cloudflare token to `Workers:Edit` on the specific projects only,
  never account-wide.
- Support two concurrently valid `VAULT_SIGNING_KEY` values so rotation does
  not require a synchronized client cutover.

The secret-passing inside the script is sound — `echo "$v" | wrangler secret
put` writes to stdin and keeps values out of the process argument list.

---

## Evidence handling

The `pyraclaw-evidence-mcp-server` is **not connected to this session**, so
this advisory has not been sealed into the ledger and no ledger index or entry
hash exists for it. No seal has been simulated.

SHA-256 digests of the five submitted artifacts, computed locally during
triage and available for sealing when the server is reachable:

```
daa0021f2b8821a67725a6b710b1a4974b4b0f992557e37fa8820a4c25b08c33  waf_rules.py
10773a0758faba25860496ab564d71aea81d204726571dc2b02f216c9dc356bb  agent_registry.py
f05cc5dac3723048d7743d91d9d2259f96d5b05b9cf6d31eace43c9a7e8423cc  rag_system.py
f976564d20df50cc909ab42c983a0faf3bee5e59d978cb36042f25a51ec3cf1c  security_headers.py
a7d2baacc6f35be4b01d3459481bae595ac0592e235dd7782d66bc99076ab8be  deploy_all.sh
```

The digest of the flagged `plugin.php` is not included: the file is on the
affected Windows host, which this session cannot reach. Capture it with step 3
above.

---

## Scope of this triage

Assessed: the detection path, the signature class, and the five submitted
artifacts by static review.

Not assessed — requires host access: Bitdefender quarantine and event logs,
the flagged file's contents and digest, running processes, network
connections, autoruns, and Windows event logs.

The verdict rests on path structure and signature class, which are strong but
indirect. Steps 3 and 4 convert it from High confidence to settled.
