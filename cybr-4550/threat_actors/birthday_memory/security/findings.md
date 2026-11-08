# Findings Register: Birthday Memory (Track A, Red Team)

Scope: localhost and my own Docker daemon only, synthetic data only. Method: OWASP WSTG, results mapped to OWASP Top 10 2021. Scoring: CVSS v3.1 base scores. Every finding below is backed by a request and response, a command output, or a scanner report in `security/raw/`, with a timestamped screenshot in `security/screenshots/` of the same ID. Raw log: `EVIDENCE-LOG.md`.

Test date: 2026-11-07 (local). Container and log timestamps are UTC, 7 hours ahead of local time (MST).

## Summary

| ID | Title | Severity | CVSS v3.1 | Status |
|----|-------|----------|-----------|--------|
| F-01 | No authentication or authorization on any API route | Critical | 9.8 | Confirmed |
| F-02 | No audit logging: a record can be deleted with no trace | Medium | 5.3 | Confirmed |
| F-03 | Database exposed on all interfaces with default superuser credentials | Critical | 9.8 | Confirmed (see caveat) |
| F-04 | Personal data stored in plaintext, readable from the Docker volume | Medium | 5.5 | Confirmed |
| F-05 | No transport encryption, and certificate checks disabled when TLS is on | Medium | 6.8 | Confirmed (code and config evidence) |
| F-06 | Wildcard CORS, no security headers, framework disclosed | Low | 3.1 | Confirmed |
| F-07 | No rate limiting and no pagination | Medium | 5.3 | Confirmed |
| F-08 | Vulnerable dependencies (5 npm advisories) | High | 7.5 | Confirmed (reachability varies) |
| F-09 | Container not hardened, image tag not pinned, no SBOM, 62 image CVEs | Low | 3.9 | Confirmed |

Root cause of F-01 (A04 Insecure Design, H-24): roles, record ownership and data minimization were never designed in. Scored under F-01, not separately.

## Not vulnerable (tested, reported as such on purpose)

| ID | Test | Result |
|----|------|--------|
| N-01 | SQL injection on `q`, `month`, `id` | Not vulnerable. All queries use `$1..` parameters. `month=1 OR 1=1` returned all 8 rows only because `Number()` gives `NaN` and the filter is skipped, the same as no filter (control in N-01b). Table intact after a `DROP TABLE` payload in `id`. |
| N-02 | Stored XSS (`<img onerror>` in first name) | Not vulnerable. Stored unsanitized by the API, rendered as literal text by React. Defense in depth note: nothing server-side prevents it if the UI ever used `dangerouslySetInnerHTML`. |
| N-03 | Mass assignment and ID guessing | Not vulnerable. Supplied `id` and `isAdmin` ignored, random UUID generated, unknown id returns 404. |
| N-04 | SSRF | Not applicable. No outbound request code in `server/src`. |
| S-02 | gitleaks, full history | No leaks found in the 12 commits with file changes (the branch has 13; one only adds an empty file). Rescanned after the evidence commit with the same result. Weak defaults `birthday/birthday` are not flagged by design and are reported in F-03. |

## Tested and not confirmed (not reported as findings)

| ID | Hypothesis | Result |
|----|------------|--------|
| H-05 | `/api/health` leaks database error text | Not confirmed. Error text was empty on a refused connection. The code returns `error.message` (`index.js` line 18), so another failure type could leak. Unauthenticated callers can tell when the database is down. |
| H-20 | Database runs as root in the container | Not confirmed. All Postgres processes run as `postgres`. `uid=0` is only the default `docker exec` user. |
| H-07 | Email regex timing | Not tested, skipped by choice (low value, risk of an unverified claim). |

## Findings

### F-01 No authentication or authorization on any API route
- **Category:** A01 Broken Access Control, A07 Identification and Authentication Failures, A04 root cause
- **Severity:** Critical, **9.8**, `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`
- **Component:** `server/src/index.js`, `server/src/routes.js`
- **Evidence:** H-01 (read all 8 records with no credentials), H-02a/b/c (create, update, delete with no credentials, 201/200/204), H-15 (no login route or auth middleware in code), H-24 (no role, owner or user_id anywhere in the code)
- **Status:** Confirmed

| Metric | Value | Justification |
|--------|-------|---------------|
| AV | N | The API listens on all interfaces (H-11) and answers plain HTTP, so it is reachable over the network. |
| AC | L | One `curl` call, no conditions, no timing or race. |
| PR | N | No login route, token, session or middleware exists (H-15). PR:L would need an account to hold, and none exists. |
| UI | N | No victim action is needed. |
| S | U | Impact stays in the app's own data. No jump to another component was demonstrated. |
| C | H | Full list returns every name, birthdate, phone and email (H-01). |
| I | H | Any caller can create, change or delete any record (H-02). |
| A | H | Any caller can delete any record, using the ids from H-01. One delete was demonstrated, the rest follow from the same call. |

- **Impact:** Anyone who can reach port 4000 can read, alter or erase all personal data. No account is needed.
- **Remediation:** Add authentication (sessions or JWT), require it on every route, add a per-record owner and role checks, and return only the columns a caller needs (`SELECT_COLUMNS` in `routes.js` line 10).

### F-02 No audit logging
- **Category:** A09 Security Logging and Monitoring Failures
- **Severity:** Medium, **5.3**, `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:L/A:N`
- **Component:** `server/src/*.js`, Postgres container logs
- **Evidence:** H-16a (Postgres log shows no entry for the create, update or delete; only checkpoint lines), H-16b (the only logging is 9 `console.*` startup and error lines)
- **Status:** Confirmed

| Metric | Value | Justification |
|--------|-------|---------------|
| AV | N | The missing control applies to every network request. |
| AC | L | Nothing is needed to take advantage of it. |
| PR | N | It applies to unauthenticated callers. |
| UI | N | No user action involved. |
| S | U | Stays within the app. |
| C | N | Logging gaps do not expose data by themselves. |
| I | L | Changes cannot be traced to a source, so the integrity of the record of events is weakened. |
| A | N | No effect on availability. |

- **Note:** CVSS fits a missing control poorly. This is scored as an enabling weakness: it hides the effects of F-01.
- **Impact:** A deletion or tampering leaves no who, what or when. Incident response and accountability are not possible.
- **Remediation:** Add request logging with a request id, caller identity and action, log all writes, and ship logs to a store the app cannot alter. Enable Postgres statement logging for writes.

### F-03 Database exposed on all interfaces with default superuser credentials
- **Category:** A05 Security Misconfiguration, A01
- **Severity:** Critical, **9.8**, `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`
- **Component:** `docker-compose.yml` (lines 8, 9, 11), `server/src/db.js` (lines 11, 12)
- **Evidence:** H-08a/b (login with `birthday/birthday`, same values in code and compose), H-09 (role is a superuser: `rolsuper = t`), H-11 (ports 5544 and 4000 answer on the LAN address 192.168.0.47), H-12 (`5432/tcp -> 0.0.0.0:5544`), H-14 (credentials in `docker-compose.yml`), H-19 (password readable with `docker inspect`)
- **Status:** Confirmed. Caveat: reachability from a second device was not tested (out of scope), so "listening on all interfaces" is certain and "reachable by others" depends on the network and firewall.

| Metric | Value | Justification |
|--------|-------|---------------|
| AV | N | Port 5544 is published on 0.0.0.0 (H-12) and answers on the LAN address (H-11). |
| AC | L | The credentials are the published defaults, so nothing needs guessing beyond reading the repo. |
| PR | N | No account is required: the password is a public default. |
| UI | N | No victim action. |
| S | U | Impact stays with the database. No escape from the container was demonstrated. |
| C | H | Superuser reads every table (H-09, H-10). |
| I | H | Superuser can modify any data. |
| A | H | Superuser can drop the database. |

- **Impact:** Anyone on the same network can log in to the database as a superuser and read, change or destroy everything, bypassing the API entirely.
- **Remediation:** Bind to loopback (`"127.0.0.1:5544:5432"`), use a generated secret from a secrets store, create a least-privilege application role (no superuser, no createdb, no createrole), and remove fallback credentials from `db.js`.

### F-04 Personal data in plaintext at rest
- **Category:** A02 Cryptographic Failures
- **Severity:** Medium, **5.5**, `CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N`
- **Component:** `birthdays` table, Docker volume `birthday_memory_thebirthdates-data`
- **Evidence:** H-10 (names, phones, emails in plain `text` columns), H-21 (seeded name found in `base/16384/16432` and in `pg_wal`, read through a throwaway read-only container)
- **Status:** Confirmed

| Metric | Value | Justification |
|--------|-------|---------------|
| AV | L | Requires access to the host to mount the volume. |
| AC | L | A simple `grep`. |
| PR | L | Needs the ability to run Docker or read the volume, which is a local privilege. |
| UI | N | No user action. |
| S | U | Stays on the same host. |
| C | H | All name, phone and email data is readable. |
| I | N | Read only. |
| A | N | Read only. |

- **Impact:** Anyone with host or backup access reads the data without touching Postgres or its credentials. Backups and the write-ahead log hold the same plaintext.
- **Remediation:** Encrypt the volume or disk, encrypt sensitive columns where appropriate, protect and encrypt backups, and limit who can run Docker on the host.

### F-05 No transport encryption, and certificate checks disabled when TLS is on
- **Category:** A02 Cryptographic Failures
- **Severity:** Medium, **6.8**, `CVSS:3.1/AV:A/AC:H/PR:N/UI:N/S:U/C:H/I:H/A:N`
- **Component:** `server/src/db.js` (lines 14, 17, 20, 27), `server/src/index.js` (listens on `http://`)
- **Evidence:** H-17 (`SHOW ssl;` = off, `pg_stat_ssl` ssl = f, API logs a plain `http://` address), H-18 (`rejectUnauthorized: false`, and TLS only turns on when `PGSSL === 'true'`)
- **Status:** Confirmed from configuration and code. No interception was performed.

| Metric | Value | Justification |
|--------|-------|---------------|
| AV | A | The attacker must be on the same network segment as the traffic. |
| AC | H | The attacker must be positioned on the path between client and server. |
| PR | N | No privileges on the app are needed. |
| UI | N | No user action. |
| S | U | Stays within the app and its database. |
| C | H | Contact data and credentials cross the network in clear text. |
| I | H | Unprotected traffic can be altered in transit. |
| A | N | No effect on availability. |

- **Impact:** On any shared network, traffic between browser, API and database can be read or changed. Even with `PGSSL=true`, the certificate is not verified, so a fake database server is accepted.
- **Remediation:** Terminate TLS in front of the API, require TLS for Postgres connections, and set `rejectUnauthorized: true` with a trusted CA.

### F-06 Wildcard CORS, no security headers, framework disclosed
- **Category:** A05 Security Misconfiguration
- **Severity:** Low, **3.1**, `CVSS:3.1/AV:N/AC:H/PR:N/UI:R/S:U/C:L/I:N/A:N`
- **Component:** `server/src/index.js` (`app.use(cors())`, no Helmet)
- **Evidence:** H-03 (`Access-Control-Allow-Origin: *` returned for `Origin: http://evil.example`), H-04 (`X-Powered-By: Express`, none of CSP, HSTS, X-Content-Type-Options, X-Frame-Options, Referrer-Policy)
- **Status:** Confirmed

| Metric | Value | Justification |
|--------|-------|---------------|
| AV | N | Exploited through a victim's browser over the network. |
| AC | H | The API has no cookies or credentials to ride, so a malicious page gains little beyond what F-01 already gives. |
| PR | N | No privileges needed. |
| UI | R | The victim must visit an attacker-controlled page. |
| S | U | Stays within the app. |
| C | L | Limited additional read exposure through the browser. |
| I | N | No integrity effect shown. |
| A | N | No availability effect. |

- **Impact:** Low on its own. Mostly relevant if authentication is added later, because a wildcard origin would then weaken it.
- **Remediation:** Restrict CORS to the known front-end origin, add Helmet, and disable `X-Powered-By`.

### F-07 No rate limiting and no pagination
- **Category:** Availability (A04 design)
- **Severity:** Medium, **5.3**, `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:L`
- **Component:** `server/src/routes.js` (`GET /`)
- **Evidence:** H-06 (30 rapid requests, all HTTP 200, no 429; the list returns all 8 rows with no limit parameter)
- **Status:** Confirmed. Impact at scale was not tested, on purpose.

| Metric | Value | Justification |
|--------|-------|---------------|
| AV | N | Network requests. |
| AC | L | Simple repeated requests. |
| PR | N | No account needed. |
| UI | N | No user action. |
| S | U | Stays within the app. |
| C | N | No confidentiality effect. |
| I | N | No integrity effect. |
| A | L | Resource use grows with table size and request volume. Only 8 rows and a 30-request burst were tested, so this is a potential reduction, not a demonstrated outage. |

- **Remediation:** Add rate limiting (for example `express-rate-limit`), pagination with a maximum page size, and a request body size limit.

### F-08 Vulnerable dependencies
- **Category:** A06 Vulnerable and Outdated Components
- **Severity:** High, **7.5** (for the reachable server item), `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:H`
- **Component:** `server/package.json` (Express 4.21.2 and its dependencies), `client/package.json` (Vite)
- **Evidence:** S-01a (server: 3 vulnerabilities, 1 critical, 1 high, 1 moderate), S-01b (client: 2 high)
- **Status:** Confirmed. None of these was exploited.

| Package | Scanner severity | My reachability assessment |
|---------|------------------|-----------------------------|
| body-parser < 1.20.9 | High | Likely reachable: `express.json()` is used on every request. Denial of service. Scored here. |
| proxy-addr 1.1.0 to 2.0.7 | Critical | Not reachable as configured: no `trust proxy` setting in `server/src`. The critical label overstates the risk for this app. |
| qs 6.7.0 to 6.16.0 | Moderate | Denial of service in `qs.stringify` with a specific option; not used by the app code. |
| vite 8.3.0 to 8.3.2 | High | Development server only, and it listens on IPv6 loopback only (S-04b), so it is not exposed to the network. |
| source-map-js 1.0.0 to 1.2.1 | High | Build and dev tooling only. |

| Metric | Value | Justification |
|--------|-------|---------------|
| AV | N | Triggered by a crafted request over the network. |
| AC | L | A single crafted compressed body. |
| PR | N | No authentication on the API (F-01). |
| UI | N | No user action. |
| S | U | Stays within the app. |
| C | N | Denial of service only. |
| I | N | Denial of service only. |
| A | H | A crash or stall takes the API offline. |

- **Remediation:** Run `npm audit fix` (a fix is available for all five), commit lockfiles, and add `npm audit` to CI.

### F-09 Container not hardened, image not pinned, no SBOM, image CVEs
- **Category:** A05, A08 Software and Data Integrity Failures, A06
- **Severity:** Low, **3.9**, `CVSS:3.1/AV:L/AC:H/PR:H/UI:N/S:U/C:L/I:L/A:L`
- **Component:** `docker-compose.yml` (line 3 `image: postgres:16-alpine`), the running container
- **Evidence:** H-13 (`CapDrop` empty, `ReadonlyRootfs=false`, `Memory=0`, `NanoCpus=0`, no PID limit, no security options; `Privileged=false` is the one good result), H-23 (mutable tag, digest not in compose, no SBOM file in the repo), S-03 (Trivy: 62 CVEs, 1 in OS packages, 61 in the bundled `gosu` binary: 1 critical, 25 high, 31 medium, 3 low, 1 unknown)
- **Status:** Confirmed

| Metric | Value | Justification |
|--------|-------|---------------|
| AV | L | Needs local or host-level access to benefit. |
| AC | H | Several preconditions are needed, and `gosu` is not exposed to network input. |
| PR | H | Needs privileged access on the host. |
| UI | N | No user action. |
| S | U | No scope change shown. |
| C | L | Limited effect. |
| I | L | Limited effect. |
| A | L | Limited effect. |

- **Note:** The raw CVE count looks alarming and the real exposure is low: `gosu` runs once at startup to drop privileges. Report both numbers.
- **Remediation:** Pin the image by digest, update to a current `postgres` image, add `cap_drop: [ALL]`, `read_only: true` with a tmpfs for the data directory needs, and memory and CPU limits, and generate an SBOM in CI.
