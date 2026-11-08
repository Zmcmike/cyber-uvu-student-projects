# Birthday Memory: Application Security Assessment (Red Team)

I tested a small web app (React, Express, PostgreSQL in Docker) the way an outside attacker would, wrote up what I found, and ranked it by risk. This is my work for CYBR 4550 at Utah Valley University, Track A (Red Team).

**Read this first:** [`security/Board-Report-Birthday-Memory.pdf`](security/Board-Report-Birthday-Memory.pdf) is the 19-page report written for a non-technical board. The rest of this page is the short version.

## The result in three lines

- Anyone who can reach the app over the network can read, change and delete every record. There is no login at all.
- The database port is open on all network interfaces and accepts the published default superuser login (`birthday` / `birthday`).
- Both problems took one command each to reproduce. Neither needs any skill.

## What I found

| ID | Finding | Severity | CVSS v3.1 |
|----|---------|----------|-----------|
| F-01 | No authentication or authorization on any API route | Critical | 9.8 |
| F-03 | Database on all interfaces with default superuser credentials | Critical | 9.8 |
| F-08 | Vulnerable dependencies (5 npm advisories) | High | 7.5 |
| F-05 | No transport encryption; certificate checks disabled when TLS is on | Medium | 6.8 |
| F-04 | Personal data stored in plaintext, readable from the Docker volume | Medium | 5.5 |
| F-02 | No audit logging: a deletion leaves no trace | Medium | 5.3 |
| F-07 | No rate limiting and no pagination | Medium | 5.3 |
| F-09 | Container not hardened, image tag not pinned, no SBOM, 62 image CVEs | Low | 3.9 |
| F-06 | Wildcard CORS, no security headers, framework disclosed | Low | 3.1 |

Scores come from a script that calculates CVSS v3.1 from each vector, not from numbers typed by hand. Every metric has a one-line reason in [`security/findings.md`](security/findings.md).

## What held up

Not everything was broken, and the report says so:

- **SQL injection:** not vulnerable. Every query is parameterized.
- **Stored XSS:** not vulnerable. React renders the payload as plain text.
- **Mass assignment and ID guessing:** not vulnerable. Extra fields are ignored and IDs are random UUIDs.
- **SSRF:** not applicable. The server makes no outbound requests.
- **Secrets in git history:** none found by gitleaks.

I also tested three ideas that did not pan out (a leaky health endpoint, a root process in the container, an email-regex timing test I skipped on purpose). They are listed as "not confirmed" and not reported as findings.

## What I did

1. Read the code and built a test plan mapped to the OWASP Top 10 (2021), using the OWASP Web Security Testing Guide as the method.
2. Ran 30 numbered tests against the app on my own machine: API requests with `curl.exe`, SQL through `psql` inside the container, Docker inspection, and a read-only container to search the database volume.
3. Ran scanners: `npm audit`, gitleaks (secrets) and Trivy (image CVEs).
4. Took a timestamped screenshot of each test (39 captures plus 3 baseline) and saved the raw command output next to it.
5. Scored and ranked each finding, and wrote a fix for each one plus a remediation roadmap.

## Fixes I would make first

1. Bind the database to localhost only and replace the default credentials with a generated secret.
2. Add authentication and authorization to every API route.
3. Add audit logging, rate limiting, and a maximum page size.
4. Turn on TLS and stop disabling certificate checks.
5. Update the vulnerable dependencies and pin the database image version.

The full roadmap, with effort estimates, is section 6 of the report.

## Limits

- One machine, one tester. I did not test from a second device, so whether another computer on a given network can reach the app depends on that network's firewall.
- Scope was my own computer and my own Docker daemon, with synthetic data only. No real personal data was used.
- I did not fix the app. This is an assessment, not a patch.

## Where things are

| Path | What it holds |
|------|---------------|
| `security/Board-Report-Birthday-Memory.pdf` | The 19-page report (also as `.docx`) |
| `security/findings.md` | Findings register with CVSS vectors and reasoning |
| `security/EVIDENCE-LOG.md` | Every test: command, result, status, coverage matrix |
| `security/screenshots/` | Timestamped screenshots, named by test ID |
| `security/raw/` | Raw command and scanner output, named by test ID |
| `APP-README.md` | The original README for the app itself (how to run it) |

## AI use

I used Claude (Anthropic) as an assistant. I ran every command and took every screenshot myself. The full disclosure is Appendix C of the report.

## Tools

PowerShell, `curl.exe`, `psql`, Docker, `npm audit`, gitleaks, Trivy, and a CVSS v3.1 calculation script.
