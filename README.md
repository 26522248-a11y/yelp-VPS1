# Yelp credential pipeline

Fast and stable credential validation against `www.yelp.com`: DataDome bypass, invisible hCaptcha inside a real
browser, no solver, 15 rows per IP, 2 workers.

| Metric | Measured |
|---|---|
| Per credential | 2.0-3.1 s |
| Rows per minute | 11-30 |
| 429 / 403 on a 15-row chunk | 0 / 0 |
| Hit rate on used combo lists | ~1.2 % |

## Why it is stable

Four failure modes are designed out:

1. **Wrong captcha field** - `hcaptcha_response`, not `h-captcha-response`. The wrong name looks exactly like a
   rejected token.
2. **Wrong IP budget** - 15 POST per IP. 429 starts at ~16/min, 403 at ~32/IP.
3. **Too many browsers** - 2 workers. A third starves on 4 GB RAM.
4. **Full temp disk** - profile deleted on exit, hard stop below 2 GB free. A full disk looked like DataDome
   for hours.

## Run

```powershell
.\runner\check-proxy.ps1 -Accounts (Get-Content .\proxies.txt)
.\runner\run-all.ps1 -Source combo.txt -PoolFile proxies.txt -Skip 0 -Take 150 -Chunk 15 -Workers 2
.\runner\report.ps1  -RunDir .\runs -HitsFile hits.txt
```

Proxies: `user:pass`, one per line. Results: `runs/progress.jsonl`, `runs/state.txt` (resume), `runs/missed.txt`
(rows whose wave died, for a re-run).

## Validity

`valid` = `success: true` + `zss` cookie + `GET /profile` 200. Everything else is unresolved or `needs-email`,
never "invalid" by default.

## Files

```
SKILL.md                    full method, constants, failure playbook
runner/batch-fast.js        worker (Chrome + relay + CDP + token factory + fetch)
runner/pool.ps1             2-worker orchestrator, 15-row chunks
runner/run-all.ps1          resumable full-list runner with missed-row queue
runner/check-proxy.ps1      session health gate
runner/report.ps1           verdict aggregation, CSV + hits export
reference/MEASUREMENTS.md   measured rates and limits
reference/PITFALLS.md       20 failure modes, symptom -> fix
```
