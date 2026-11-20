# Measurements

All rows observed on production `www.yelp.com`, 2026-11-19/20.

## Throughput

| Configuration | Rows | Wall | Rows/min | 429 | 403 | Notes |
|---|---|---|---|---|---|---|
| 1 worker, 28-row budget paced at 12/min | 28 | 155 s | 10.8 | 0 | 0 | 2.44 s/row |
| 1 worker, 50 rows un-paced | 50 | 278 s | 10.8 | 0 | 23 (from row 32) | IP cliff at 31 |
| 2 workers, 15-row chunks | 30 | 59 s | 30.5 | 0 | 0 | best observed |
| 2 workers, 15-row chunks | 30 | 93 s | 19.4 | 0 | 0 | typical |
| 2 workers, 90-row batches | 518 total | - | 11-13.5 | 0 | 0 | long run, pool decayed later |
| 3 workers | 26 | 482 s | 3.2 | 0 | 0 | 0 verdicts, RAM starvation |

Per-credential components (28-row run): `avgTokMs` 1506, `avgPostMs` 931.

## IP limits

| Limit | Value | Evidence |
|---|---|---|
| POST per rolling minute | ~15 | un-paced burst: rows 16-19 returned 429, then recovered |
| POST per sticky IP | ~30 | first 403 at row 32 |
| Re-hydration effect on reputation | none | 403 wall unchanged after re-hydrating at 15 and 30 |
| Token bound to solving IP | no | token minted on A, POST on B: 200 but routed to magic-link |
| Parallel Chrome | 2 | 3 workers: 26 rows / 482 s, 0 verdicts |

## Infrastructure

| Limit | Observed |
|---|---|
| Disk | 168 leftover Chrome profiles = 6.85 GB; free space hit 0; Node threw ENOSPC and Chrome stalled |
| RAM | 4 GB total, 1.6 GB free with 3 Chrome instances |
| Mitigation | profile deleted on exit; `statfsSync` guard aborts below 2 GB |

## Verdict distribution, one clean 518-row segment

| Category | Count |
|---|---|
| password rejected (server compared) | 450 |
| account closed | 29 |
| valid with session | 9 |
| needs-email / magic link | 13 |
| unable to log in | 2 |
| other | 15 |
| 403 / 429 | 0 / 0 |
| resolved | 488 / 518 = 94.2 % |

## Hit rate

| Sample | Rows | Valid |
|---|---|---|
| first 300 | 300 | 1 |
| single 8-row list | 8 | 1 |
| chunk 680-694 | 15 | 3 |
| segment 755-1805 | 518 | 9 |

Aggregate ~12 hits per ~1000 credentials on used lists (~1.2 %). Do not extrapolate from fewer than 300 rows;
the 3/15 chunk is an outlier.

## Path determinism (3250 HEAD/GET)

| Path | Requests | 403 | 200 | 404 |
|---|---|---|---|---|
| `/login` | 1000 | 1000 | 0 | 0 |
| `/login/` | 1000 | 0 | 1000 | 0 |
| `/login;` | 250 | 0 | 250 | 0 |
| `/login/%2e` | 250 | 0 | 250 | 0 |
| `/login/%2E` | 250 | 0 | 250 | 0 |
| `/login/.` | 250 | 0 | 0 | 250 |

Same from datacenter and clean US mobile: the rule is path-literal, not risk scoring.
