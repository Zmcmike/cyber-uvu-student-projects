# Evidence Log (Track A, Red Team)

Working log of every piece of evidence for the board report. IDs starting with H are hypotheses until proven.
A hypothesis only becomes a finding once the status is CONFIRMED and the raw output and screenshot exist.
Status values: TODO / CONFIRMED / NOT VULNERABLE (tested) / NOT TESTED (reason)

Rules for every capture:
- Run `Get-Date` in the same terminal first so a timestamp is in the screenshot.
- Use `curl.exe -i` (plain `curl` in PowerShell is a different command).
- Save raw output to security/raw/<ID>.txt (use `| Tee-Object security\raw\<ID>.txt`).
- Save the screenshot to security/screenshots/<ID>.png.
- Redact anything sensitive. Never commit server/.env.
- Scope: own machine only. Do not test from another device or network without asking the instructor first.

## Step progress
| Step | Captures | Status |
|------|----------|--------|
| 1 | B-03 baseline git state | DONE |

## Baseline (before any change)
| ID | What to capture | Command or view | Status |
|----|-----------------|-----------------|--------|
| B-01 | Container state | `docker compose ps` (run inside the project folder, after db:up) | CONFIRMED |
| B-02 | App running with seeded data | Browser at http://localhost:5173, taskbar clock visible | CONFIRMED |
| B-03 | Unmodified config and git history | `git status`, `git log --oneline` | CONFIRMED |

## Domain 1: API
| ID | OWASP | Hypothesis | Proof to capture | Status |
|----|-------|-----------|------------------|--------|
| H-01 | A01 | Unauthenticated READ, full dump | `curl.exe -i http://localhost:4000/api/birthdays` | CONFIRMED |
| H-02a | A01 | Unauthenticated CREATE | POST a record you invent | CONFIRMED |
| H-02b | A01 | Unauthenticated UPDATE | PUT on the record you created | CONFIRMED |
| H-02c | A01 | Unauthenticated DELETE | DELETE on the record you created | CONFIRMED |
| H-03 | A05 | CORS allows any origin | `curl.exe -i -H "Origin: http://evil.example" ...` (header check only) | CONFIRMED |
| H-04 | A05 | Missing security headers, X-Powered-By | Header block from any `curl.exe -i`, compare to Helmet defaults | CONFIRMED |
| H-05 | A05 | Health endpoint leaks error text | Stop the DB container briefly, call /api/health | NOT CONFIRMED (error text was empty on connection refused; code returns error.message, so other failures could leak) |
| H-06 | Availability | No rate limit, no pagination | Small controlled request burst only, count records returned | CONFIRMED |
| H-07 | Availability | Email regex slow on long input | Small timing test only (optional) | NOT TESTED (skipped by choice, low value) |
| H-15 | A07 | No authentication layer exists at all | Request with no credentials succeeds, no login route, no auth middleware in code | CONFIRMED |
| H-16a | A09 | No audit trail: database logs show nothing about the delete | `docker logs --timestamps thebirthdates-db` after the delete | CONFIRMED |
| H-16b | A09 | No audit trail: server code logs only startup and errors | Search server/src for `console.` | CONFIRMED |
| H-24 | A04 | Insecure design: never designed in | Code references: no roles, no ownership, no data minimization, no threat model | CONFIRMED |

## Tested, expected not vulnerable
| ID | OWASP | Test | Proof to capture | Status |
|----|-------|------|------------------|--------|
| N-01 | A03 | SQL injection on q, month, id | Payload requests and responses | NOT VULNERABLE (q and id safe; month=1 OR 1=1 returns all 8 only because Number() gives NaN and the filter is skipped, same as no filter; table intact) |
| N-02 | A03 | Stored XSS rendered in the UI | Store a script payload, screenshot the UI | NOT VULNERABLE (payload stored unsanitized by API but rendered as literal text by React; no script ran) |
| N-03 | A01 | Mass assignment and ID guessing | Extra fields ignored, random UUIDs | NOT VULNERABLE (supplied id and isAdmin ignored, server generated random UUID; unknown id returns 404) |
| N-04 | A10 | SSRF | Search server/src for outbound request code (fetch, axios, http.request), expect none | NOT APPLICABLE (no outbound request code in server/src) |

## Domain 2: Database
| ID | OWASP | Hypothesis | Proof to capture | Status |
|----|-------|-----------|------------------|--------|
| H-08 | A05 | Default credentials work, and appear in repo | psql login as birthday/birthday, plus db.js lines 11-12 and docker-compose.yml lines 8-9 | CONFIRMED |
| H-09 | A01 | App role is superuser | `SELECT rolsuper FROM pg_roles WHERE rolname='birthday';` | CONFIRMED |
| H-10 | A02 | PII stored in plaintext | `SELECT * FROM birthdays LIMIT 3;` | CONFIRMED |
| H-11 | A05 | DB reachable beyond loopback | Connect to port 5544 via this machine's own LAN IP, own machine only | CONFIRMED (listening on all interfaces; reachability from other devices not tested, out of scope) |
| H-17 | A02 | PII in transit unencrypted (browser to API over HTTP, API to DB without TLS) | `SHOW ssl;` in psql, plus the http:// URL in use | CONFIRMED |
| H-18 | A02 | TLS cert not verified when PGSSL=true | db.js line 17 (`rejectUnauthorized: false`), code evidence | CONFIRMED |
| H-21 | A02 | PII readable directly off the volume | Mount volume in a throwaway container and search the data files for a seeded name | CONFIRMED (name found in base/16384/16432 and pg_wal on volume; first exact-email grep returned nothing, case-insensitive name search found it) |

## Domain 3: Container
| ID | OWASP | Hypothesis | Proof to capture | Status |
|----|-------|-----------|------------------|--------|
| H-12 | A05 | Port published on all interfaces | `docker port thebirthdates-db` | CONFIRMED |
| H-13 | A05 | No runtime hardening (caps, read-only, limits) | `docker inspect thebirthdates-db` | CONFIRMED (not privileged, which is the one good result) |
| H-14 | A05 | Plaintext creds in compose | docker-compose.yml lines 7-9 | CONFIRMED (credentials at lines 8-9, not 7-9) |
| H-19 | A05 | Secrets readable via standard Docker commands | `docker inspect` and `docker exec ... env` show the password | CONFIRMED |
| H-20 | A05 | Runtime user inside the container | `docker exec thebirthdates-db id` and process list | NOT CONFIRMED (postgres server processes run as the postgres user, not root; the root uid is only the docker exec default shell; informational only) |
| H-23 | A08 | Image tag mutable, no SBOM | Compose uses `postgres:16-alpine` (tag), compare to image digest, generate SBOM if Syft available, check lockfile hashes | CONFIRMED |

## Scanners
| ID | OWASP | Tool | Command | Status |
|----|-------|------|---------|--------|
| S-01 | A06 | npm audit | `npm audit` in server and client | CONFIRMED (server: 3 = 1 critical, 1 high, 1 moderate; client: 2 high, both dev-server tooling) |
| S-02 | A02/A05 | gitleaks on full history | `gitleaks detect` | NOT VULNERABLE (gitleaks: no leaks in the 12 commits with file changes; 13 on the branch, one only adds an empty file; rescanned after the evidence commit) |
| S-03 | A06 | Trivy on postgres image, with CVE count | `trivy image postgres:16-alpine` | CONFIRMED (62 total: alpine 3.24.4 OS packages 1 medium; usr/local/bin/gosu Go binary 61 = 1 unknown, 3 low, 31 medium, 25 high, 1 critical) |
| S-04 | Surface | nmap, localhost only | `nmap -p 4000,5173,5544 127.0.0.1` (optional) | CONFIRMED (PowerShell port check, nmap not installed: 4000 and 5544 open on 127.0.0.1; control ports 5432 and 8080 closed; 5173 refused on 127.0.0.1 because Vite listens on IPv6 [::1] only, HTTP 200 via localhost, so web app is loopback-only unlike API and DB) |

## Coverage matrix (every required category needs a verdict)
| Category | Evidence IDs | Verdict |
|----------|--------------|---------|
| A01 Broken Access Control | H-01, H-02a, H-02b, H-02c, H-09, N-03 | CONFIRMED (F-01); N-03 NOT VULNERABLE |
| A02 Cryptographic Failures | H-10, H-17, H-18, H-21 | CONFIRMED (F-04, F-05) |
| A03 Injection | N-01, N-02 | NOT VULNERABLE (tested) |
| A04 Insecure Design | H-24 | CONFIRMED (root cause of F-01) |
| A05 Security Misconfiguration | H-03, H-04, H-05, H-08, H-11, H-12, H-13, H-14, H-19, H-20 | CONFIRMED (F-03, F-06, F-09); H-05 and H-20 NOT CONFIRMED |
| A06 Vulnerable and Outdated Components | S-01, S-03 | CONFIRMED (F-08, F-09) |
| A07 Identification and Authentication Failures | H-15 | CONFIRMED (F-01) |
| A08 Software and Data Integrity Failures | H-23 | CONFIRMED (F-09) |
| A09 Logging and Monitoring Failures | H-16a, H-16b | CONFIRMED (F-02) |
| A10 SSRF | N-04 | NOT APPLICABLE (no outbound request code) |
| Container-specific | H-12, H-13, H-19, H-20, H-21, S-03 | CONFIRMED (F-03, F-04, F-09); H-20 NOT CONFIRMED |
| Availability | H-06, H-07 | CONFIRMED (F-07); H-07 NOT TESTED (skipped, low value) |
