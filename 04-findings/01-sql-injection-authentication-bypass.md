# Finding 01 — SQL Injection Leading to Authentication Bypass

**Severity:** Critical

**Affected Endpoint:** `POST /rest/user/login`

**Affected Parameter:** `email`

---

## Description

The login endpoint of OWASP Juice Shop was found to be vulnerable to SQL Injection in the `email` parameter. By submitting a crafted SQL injection payload instead of a valid email address, it was possible to bypass the application's authentication mechanism entirely and obtain a valid authentication token for an administrative account, without knowing its password.

This occurs because user-supplied input from the `email` field appears to be used directly within a SQL query without proper sanitization or the use of parameterized queries, allowing an attacker to alter the logic of the underlying database query.

## Prerequisites / Environment

- **Target:** OWASP Juice Shop, running locally via Docker
- **Target URL:** `http://127.0.0.1:3000`
- **Tooling:** Burp Suite (Repeater), used to craft and send modified login requests
- **Access level required:** None (unauthenticated endpoint)

## Testing Method

Testing followed three progressive steps against the login endpoint: a baseline test to confirm normal behavior, an error-based test to observe how the application handles malformed input, and a validation test using a standard SQL injection payload to confirm the impact.

## Baseline Authentication Test

A normal login request was sent with an invalid but well-formed email and password:

```
POST /rest/user/login
Content-Type: application/json

{
  "email": "test@test.com",
  "password": "test"
}
```

**Response:**

```
HTTP/1.1 401 Unauthorized
```

The application responded with "Invalid email or password," confirming standard, expected authentication behavior. This served as the baseline for comparison.

## SQL Injection Error Test

The `email` field was modified to contain a single quote character, a common technique used to test whether user input is being interpreted as part of a SQL query:

```
POST /rest/user/login
Content-Type: application/json

{
  "email": "'",
  "password": "test"
}
```

**Response:**

```
HTTP/1.1 500 Internal Server Error
```

The response body exposed a SQLite database error and internal stack trace details. This confirmed that the single quote was being interpreted as part of a SQL statement rather than being safely escaped, indicating a SQL injection point. On its own, this error confirms unsafe input handling and information disclosure, but it does not by itself prove that authentication can be bypassed.

## Authentication Bypass Validation

To validate the impact, the `email` field was modified using a classic SQL injection authentication bypass payload:

```
POST /rest/user/login
Content-Type: application/json

{
  "email": "' OR 1=1--",
  "password": "test"
}
```

**Response:**

```
HTTP/1.1 200 OK
```

The response returned a valid authentication token associated with the account `admin@juice-sh.op`. The application authenticated the request successfully despite no valid credentials being supplied, confirming a full authentication bypass via SQL injection.

**Note:** The JWT/authentication token observed in this response is represented as `[REDACTED]` throughout this project. The real token value is never published.

## Observed Result

| Test | Request | Result |
|---|---|---|
| Baseline | Valid-format email/password | `401 Unauthorized` |
| Error test | `email: "'"` | `500 Internal Server Error` (DB error exposed) |
| Bypass validation | `email: "' OR 1=1--"` | `200 OK` — authenticated as `admin@juice-sh.op` |

## Impact

An attacker able to exploit this vulnerability could:

- Bypass authentication for any account, including administrative accounts, without valid credentials
- Gain unauthorized access to the application as an administrator
- Potentially access, modify, or exfiltrate sensitive application data depending on the privileges of the compromised account

Given that this affects the primary login mechanism and can lead to full administrative access, this finding is rated **Critical**.

## Evidence

See [`screenshots/06-sql-injection-error.png`](../screenshots/06-sql-injection-error.png) and [`screenshots/07-sqli-authentication-bypass.png`](../screenshots/07-sqli-authentication-bypass.png). Authentication token values are redacted/blurred in all evidence.

## Root Cause

User-supplied input from the `email` parameter appears to be concatenated directly into a SQL query rather than being handled through parameterized queries or an ORM with proper input binding. This allows attacker-controlled input to change the structure and logic of the executed SQL statement.

## Remediation

- Use parameterized queries or prepared statements for all database operations involving user input.
- Never concatenate user-controlled input directly into SQL query strings.
- Apply strict server-side input validation on all authentication-related fields.
- Return generic authentication error messages that do not reveal internal application or database details.
- Ensure error handling does not expose stack traces or database error messages to end users.

## Recommendation

It is recommended that the development team review all database queries across the application for similar unsafe input handling patterns, not just the login endpoint, and adopt parameterized queries as a standard practice across the codebase.

## Limitations

This finding was validated manually using a single, well-known SQL injection payload. A full assessment would involve testing additional injection points, payloads, and endpoints, which was outside the scope of this project.

## References

- OWASP Top 10 — Injection: https://owasp.org/www-project-top-ten/
- OWASP Juice Shop project: https://owasp.org/www-project-juice-shop/
