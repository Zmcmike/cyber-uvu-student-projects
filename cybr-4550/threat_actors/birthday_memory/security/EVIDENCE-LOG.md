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
| H-01 | A01 | Unauthenticated READ, full dump | `curl.exe -i http://localhost:4000/api/birthdays` | TODO |
| H-02a | A01 | Unauthenticated CREATE | POST a record you invent | TODO |
| H-02b | A01 | Unauthenticated UPDATE | PUT on the record you created | TODO |
| H-02c | A01 | Unauthenticated DELETE | DELETE on the record you created | TODO |
| H-03 | A05 | CORS allows any origin | `curl.exe -i -H "Origin: http://evil.test" ...` plus a page served from a second local origin | TODO |
| H-04 | A05 | Missing security headers, X-Powered-By | Header block from any `curl.exe -i`, compare to Helmet defaults | TODO |
| H-05 | A05 | Health endpoint leaks error text | Stop the DB container briefly, call /api/health | TODO |
| H-06 | Availability | No rate limit, no pagination | Small controlled request burst only, count records returned | TODO |
| H-07 | Availability | Email regex slow on long input | Small timing test only (optional) | TODO |
| H-15 | A07 | No authentication layer exists at all | Request with no credentials succeeds, no login route, no auth middleware in code | TODO |
| H-16 | A09 | No audit trail for destructive actions | Delete a record, then show app console output and `docker logs thebirthdates-db`: who and when? | TODO |
| H-24 | A04 | Insecure design: never designed in | Code references: no roles, no ownership, no data minimization, no threat model | TODO |

## Tested, expected not vulnerable
| ID | OWASP | Test | Proof to capture | Status |
|----|-------|------|------------------|--------|
| N-01 | A03 | SQL injection on q, month, id | Payload requests and responses | TODO |
| N-02 | A03 | Stored XSS rendered in the UI | Store a script payload, screenshot the UI | TODO |
| N-03 | A01 | Mass assignment and ID guessing | Extra fields ignored, random UUIDs | TODO |
| N-04 | A10 | SSRF | Search server/src for outbound request code (fetch, axios, http.request), expect none | TODO |

## Domain 2: Database
| ID | OWASP | Hypothesis | Proof to capture | Status |
|----|-------|-----------|------------------|--------|
| H-08 | A05 | Default credentials work, and appear in repo | psql login as birthday/birthday, plus db.js lines 9-13 and docker-compose.yml lines 7-9 | TODO |
| H-09 | A01 | App role is superuser | `SELECT rolsuper FROM pg_roles WHERE rolname='birthday';` | TODO |
| H-10 | A02 | PII stored in plaintext | `SELECT * FROM birthdays LIMIT 3;` | TODO |
| H-11 | A05 | DB reachable beyond loopback | Connect to port 5544 via this machine's own LAN IP, own machine only | TODO |
| H-17 | A02 | PII in transit unencrypted (browser to API over HTTP, API to DB without TLS) | `SHOW ssl;` in psql, plus the http:// URL in use | TODO |
| H-18 | A02 | TLS cert not verified when PGSSL=true | db.js line 17 (`rejectUnauthorized: false`), code evidence | TODO |
| H-21 | A02 | PII readable directly off the volume | Mount volume in a throwaway container and search the data files for a seeded name | TODO |

## Domain 3: Container
| ID | OWASP | Hypothesis | Proof to capture | Status |
|----|-------|-----------|------------------|--------|
| H-12 | A05 | Port published on all interfaces | `docker port thebirthdates-db` | TODO |
| H-13 | A05 | No runtime hardening (caps, read-only, limits) | `docker inspect thebirthdates-db` | TODO |
| H-14 | A05 | Plaintext creds in compose | docker-compose.yml lines 7-9 | TODO |
| H-19 | A05 | Secrets readable via standard Docker commands | `docker inspect` and `docker exec ... env` show the password | TODO |
| H-20 | A05 | Runtime user inside the container | `docker exec thebirthdates-db id` and process list | TODO |
| H-23 | A08 | Image tag mutable, no SBOM | Compose uses `postgres:16-alpine` (tag), compare to image digest, generate SBOM if Syft available, check lockfile hashes | TODO |

## Scanners
| ID | OWASP | Tool | Command | Status |
|----|-------|------|---------|--------|
| S-01 | A06 | npm audit | `npm audit` in server and client | TODO |
| S-02 | A02/A05 | gitleaks on full history | `gitleaks detect` | TODO |
| S-03 | A06 | Trivy on postgres image, with CVE count | `trivy image postgres:16-alpine` | TODO |
| S-04 | Surface | nmap, localhost only | `nmap -p 4000,5173,5544 127.0.0.1` (optional) | TODO |

## Coverage matrix (every required category needs a verdict)
| Category | Evidence IDs | Verdict |
|----------|--------------|---------|
| A01 Broken Access Control | H-01, H-02a, H-02b, H-02c, H-09, N-03 | TODO |
| A02 Cryptographic Failures | H-10, H-17, H-18, H-21 | TODO |
| A03 Injection | N-01, N-02 | TODO |
| A04 Insecure Design | H-24 | TODO |
| A05 Security Misconfiguration | H-03, H-04, H-05, H-08, H-11, H-12, H-13, H-14, H-19, H-20 | TODO |
| A06 Vulnerable and Outdated Components | S-01, S-03 | TODO |
| A07 Identification and Authentication Failures | H-15 | TODO |
| A08 Software and Data Integrity Failures | H-23 | TODO |
| A09 Logging and Monitoring Failures | H-16 | TODO |
| A10 SSRF | N-04 | TODO |
| Container-specific | H-12, H-13, H-19, H-20, H-21, S-03 | TODO |
| Availability | H-06, H-07 | TODO |
