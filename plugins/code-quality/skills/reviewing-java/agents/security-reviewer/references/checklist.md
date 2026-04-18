# Java Security Bug Patterns, OWASP Top 10 Mapping, and Code Review Guide

## Table of Contents
1. Injection (4 subsections) — SQL, LDAP, OS command, expression language; mapped to `A03:2021` / `A05:2025`
2. Cross-Site Scripting (XSS) (3 subsections) — contextual output encoding, strict CSP, payload channels; mapped to `A03:2021` Injection
3. CSRF (3 subsections) — Spring Security tokens, SPAs, cookie vs header tokens
4. Server-Side Request Forgery (SSRF) (3 subsections) — mapped to `A10:2021`; folded into `A01:2025` Broken Access Control
5. Input Validation (5 subsections) — Jakarta Bean Validation, allowlists, path traversal, CRLF, uploads
6. Authentication & Authorization (4 subsections) — Spring Security 6.4, JWT (RFC 8725), OAuth (RFC 9700), sessions; mapped to `A01:2021` + `A07:2021` / `A01:2025` + `A07:2025`
7. Cryptography (4 subsections) — passwords (Argon2id 19 MiB baseline), randomness, encryption (AES-GCM), TLS (NIST SP 800-52 Rev. 2); mapped to `A02:2021` / `A04:2025`
8. Secrets Management (3 subsections) — TruffleHog, Gitleaks, Vault, Spring Cloud configtree
9. Serialization & Deserialization (3 subsections) — JEP 290 / JEP 415 filters, Jackson polymorphic types
10. Dependency & Supply Chain Security (5 subsections) — OWASP Dependency-Check, SBOM, Dependabot, xz-utils / Log4Shell / Spring4Shell post-mortems; mapped to `A06:2021` / `A03:2025` Software Supply Chain Failures (new in 2025 RC1)
11. Spring Boot Actuator & Operational Hardening (4 subsections) — CVE-2025-22235, endpoint exposure matrix, separate `management.server.port`
12. Exceptional Condition Handling (4 subsections) — mapped to `A10:2025` Mishandling of Exceptional Conditions (new category in 2025 RC1)
13. Security Logging & Alerting (4 subsections) — mapped to `A09:2021` / `A09:2025` (renamed with Alerting emphasis)
14. Quick Security Review Heuristic (12-step scan order)
15. Bug-Pattern Code Pairs (`bad:` / `good:`) — 15 canonical anti-patterns with fixes (15.1–15.15)
16. Review Checklist Items — 13 section-mapped `- [ ]` rubrics (16.1–16.13)
17. Key References — inline foundational list
18. Research Questions to Cross-Check Against Sources — 60 questions across 11 buckets (A–K), each mapped to section + source
19. Sources — normative / foundational / tooling / checklists / industry case studies (5 subgroups)
20. Gotchas — 20 common wrong assumptions with stable IDs (G-01..G-20)

---

## 1. Injection

Maps to `A03:2021` Injection / `A05:2025` Injection. Corresponds to CWE-89 (SQL Injection, #2 on CWE Top 25 2025), CWE-78 (OS Command Injection, #9), CWE-94 (Code Injection, #10), and CWE-943 (LDAP Injection).

### 1.1 SQL Injection
- **Never** concatenate user input into SQL queries. Use `PreparedStatement` with parameterized queries or JPA named parameters exclusively
- **Ban** `Statement.execute(String)` and `Statement.executeQuery(String)` with dynamic SQL
- **Native queries in JPA**: `@Query(nativeQuery = true, value = "SELECT * FROM users WHERE name = :name")` — safe. `"... WHERE name = '" + name + "'"` — vulnerable
- **Criteria API / QueryDSL**: generally safe. But dynamic `ORDER BY` clauses often bypass parameterization — validate against an allowlist of column names

Source: OWASP SQL Injection Prevention Cheat Sheet; OWASP Query Parameterization Cheat Sheet; CWE-89.

### 1.2 LDAP Injection
- Never pass unsanitized input to `DirContext.search()` or LDAP filter strings
- Use Spring LDAP's `LdapQueryBuilder` which auto-escapes filter values

Source: OWASP LDAP Injection Prevention Cheat Sheet; CWE-943.

### 1.3 OS Command Injection
- **Never** use `Runtime.exec(String)` with shell string interpolation (CWE-78, #9 on CWE Top 25 2025)
- Use `ProcessBuilder` with argument arrays: `new ProcessBuilder("cmd", arg1, arg2)` — arguments are never interpreted as shell syntax
- If shell execution is unavoidable, validate input against a strict allowlist pattern

Source: OWASP OS Command Injection Defense Cheat Sheet; CERT IDS07-J "Sanitize untrusted data passed to the Runtime.exec() method".

### 1.4 Expression Language Injection
- Never evaluate user input as SpEL, OGNL, or EL expressions
- Spring `@Value("${...}")` is safe (property resolution). `@Value("#{...}")` evaluates SpEL — never use with user input

Source: CWE-94 Code Injection; Spring Security reference "Expression-Based Access Control" (docs.spring.io/spring-security/reference/6.4/).

---

## 2. Cross-Site Scripting (XSS)

Maps to `A03:2021` Injection / `A05:2025` Injection. Corresponds to CWE-79 (XSS, #1 on CWE Top 25 2025 — highest-ranked weakness).

### 2.1 Contextual Output Encoding
- **Templating engines**: use auto-escaping by default. Thymeleaf `th:text` (safe) vs `th:utext` (unsafe — raw HTML). Flag any `th:utext` with user-controlled data
- **REST APIs returning HTML**: encode output with OWASP Java Encoder per context:

| Context | Method | Example |
|---|---|---|
| HTML body | `Encode.forHtml(s)` | `<div>X</div>` |
| HTML attribute (quoted) | `Encode.forHtmlAttribute(s)` | `<input value="X">` |
| HTML content (same as body) | `Encode.forHtmlContent(s)` | text children of an element |
| JavaScript string literal | `Encode.forJavaScript(s)` | `var name = "X"` |
| CSS string value | `Encode.forCssString(s)` | `background: url("X")` |
| URL component | `Encode.forUriComponent(s)` | `?q=X` |
| XML text | `Encode.forXml(s)` | XML body |

Each context has distinct metacharacters; `Encode.forHtml` is NOT safe for JavaScript contexts and vice versa. See Gotcha G-11.

### 2.2 Security Headers (Strict CSP)
Configure in `SecurityFilterChain`:

| Header | Recommended Value | Purpose |
|---|---|---|
| `Content-Security-Policy` | `default-src 'none'; script-src 'nonce-{random}' 'strict-dynamic'; object-src 'none'; base-uri 'self'; frame-ancestors 'none'` | Script-execution allowlist (OWASP strict CSP) |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains; preload` | TLS enforcement (RFC 6797) |
| `X-Content-Type-Options` | `nosniff` | Block MIME sniffing |
| `X-Frame-Options` | `DENY` | Legacy clickjacking defense; prefer `frame-ancestors` in CSP |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Reduce data leak |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` | Disable unused sensors |

### 2.3 Payload Channels
- **JSON APIs**: generally safe from reflected XSS, but stored XSS still applies if HTML clients render the data without escaping
- **SPAs**: React/Vue/Angular escape by default in text interpolation but `dangerouslySetInnerHTML` / `v-html` / `[innerHTML]` reintroduce XSS

Source: OWASP Cross Site Scripting Prevention Cheat Sheet; OWASP Content Security Policy Cheat Sheet; OWASP Java Encoder project (owasp.org/www-project-java-encoder/); CWE-79.

---

## 3. CSRF

Maps to `A01:2021` Broken Access Control / `A01:2025` Broken Access Control. Corresponds to CWE-352 (CSRF, #3 on CWE Top 25 2025).

### 3.1 Baseline Protection
- Spring Security enables CSRF protection by default for session-based auth
- Verify CSRF tokens on **all state-changing endpoints** (POST, PUT, DELETE, PATCH)
- `GET` endpoints must be **safe** (no side effects) — otherwise CSRF attacks via `<img>` tags

### 3.2 SPAs and Token Placement
- **SPAs with JWT in `Authorization: Bearer` header**: CSRF tokens are NOT required because browsers do not auto-attach the `Authorization` header cross-origin (see Gotcha G-13)
- **JWT in cookies**: CSRF protection IS required — cookies are auto-attached
- Use `CookieCsrfTokenRepository.withHttpOnlyFalse()` for SPAs that need to read the token via JavaScript
- Pair with `XorCsrfTokenRequestAttributeHandler` for BREACH-attack mitigation (Spring Security 6.4 default recommendation)

### 3.3 Known Gaps
- `CookieCsrfTokenRepository` does not set `SameSite` on the CSRF cookie via `saveToken` path (Spring Security issue #14131). Set it via a custom `CookieSerializer` or servlet container config.

Source: OWASP Cross-Site Request Forgery Prevention Cheat Sheet; Spring Security reference 6.4 "CSRF" section (docs.spring.io/spring-security/reference/6.4/servlet/exploits/csrf.html); CWE-352.

---

## 4. Server-Side Request Forgery (SSRF)

Maps to `A10:2021` Server-Side Request Forgery (dedicated category in 2021). Folded into `A01:2025` Broken Access Control in 2025 RC1. Corresponds to CWE-918.

### 4.1 Allowlist Strategy
- **Allowlist** destination hosts and schemes for any server-side HTTP call (not a denylist — new cloud metadata endpoints appear regularly)
- Never let user input control full URLs for server-side HTTP calls (`RestTemplate`, `WebClient`, `HttpClient`)
- If user provides a URL (webhooks, callbacks): parse and validate scheme (`https` only), host (against allowlist), and port

### 4.2 Reject Internal/Link-Local Ranges
Reject resolution targeting any of:

| Range | Purpose |
|---|---|
| `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | RFC 1918 private IPv4 |
| `127.0.0.0/8` | Loopback |
| `169.254.0.0/16` | Link-local (cloud instance metadata — AWS `169.254.169.254`, GCP, Azure) |
| `0.0.0.0` | Meta-address |
| `::1` | IPv6 loopback |
| `fc00::/7`, `fd00::/8` | IPv6 unique-local |
| `fe80::/10` | IPv6 link-local |

### 4.3 DNS Rebinding
- Resolve DNS **before** checking, AND re-resolve on the actual connect (attacker-controlled DNS can return a safe IP to the allowlist check then flip to `169.254.169.254` on connect)
- Disable HTTP redirects or re-validate the redirect target (see CWE-601 Open Redirect)

Source: OWASP Server-Side Request Forgery Prevention Cheat Sheet; CWE-918; PortSwigger SSRF research.

---

## 5. Input Validation

Cross-cuts multiple OWASP categories. Corresponds to CWE-20 (Improper Input Validation) and CWE-22 (Path Traversal, #6 on CWE Top 25 2025).

### 5.1 Where to Validate
- **Controller layer**: structural validation with Jakarta Bean Validation 3.1 (`@NotNull`, `@NotBlank`, `@Size`, `@Pattern`, `@Email`, `@Min`/`@Max`, `@Positive`, `@Future`/`@Past`) + `@Valid` on request bodies/parameters; reference implementation Hibernate Validator 9.x
- **Service layer**: business rule validation (e.g., "order total cannot exceed credit limit")
- **Never** trust client-side validation alone

Source: Jakarta Validation 3.1 specification (jakarta.ee/specifications/bean-validation/3.1/); Hibernate Validator 9 reference.

### 5.2 Allowlist over Denylist
- Define what IS permitted (alphanumeric pattern for 90%+ of fields). Reject everything else.

Source: OWASP Input Validation Cheat Sheet.

### 5.3 Path Traversal
CWE-22, #6 on CWE Top 25 2025. Never pass user input to filesystem APIs (`new File(userInput)`). If unavoidable:

```java
Path resolved = basePath.resolve(userInput).normalize();
if (!resolved.startsWith(basePath)) {
    throw new SecurityException("Path traversal attempt");
}
```

Better: map file references to database IDs, never expose filesystem paths. Never pass user input to `ServletContext.getRealPath(userInput)` — it performs no sandbox check.

Source: OWASP File Upload Cheat Sheet; CERT FIO16-J "Canonicalize path names before validating them"; CWE-22.

### 5.4 Header Injection (CRLF)
- Reject HTTP header values containing `\r` (CR, `%0D`) or `\n` (LF, `%0A`)
- Servlet containers since 2017 enforce this at `response.setHeader()` — do not rely on it alone for redirect `Location` headers built from user input

Source: CWE-113 Improper Neutralization of CRLF Sequences in HTTP Headers.

### 5.5 File Uploads
- Validate content type (magic bytes, not just extension — e.g., Apache Tika `Tika.detect(InputStream)`)
- Enforce size limits (`spring.servlet.multipart.max-file-size`, `max-request-size`)
- Store outside webroot with random names; serve via controller that sets `Content-Disposition: attachment`
- Scan for malware on a quarantine mount before move-to-serve

Source: OWASP File Upload Cheat Sheet.

---

## 6. Authentication & Authorization

Maps to `A01:2021` Broken Access Control + `A07:2021` Identification and Authentication Failures / `A01:2025` Broken Access Control + `A07:2025` Authentication Failures (renamed in 2025 RC1). Corresponds to CWE-862 Missing Authorization (#4 on CWE Top 25 2025), CWE-287 Improper Authentication, CWE-798 Hardcoded Credentials.

### 6.1 Spring Security Configuration (6.4)
- `SecurityFilterChain` bean replaces the legacy `WebSecurityConfigurerAdapter` (removed in Spring Security 6.x)
- **Catch-all rule**: `.anyRequest().authenticated()` as the last matcher in `SecurityFilterChain`. More specific matchers must precede general ones (first-match-wins — see Gotcha G-18)
- **Audit open endpoints**: flag any `.permitAll()` or `@PermitAll` without documented justification
- **Method-level security**: use `@PreAuthorize` / `@PostAuthorize` on service methods, not just controllers. In Spring Security 6.4 these are implemented by `AuthorizationManagerBeforeMethodInterceptor` / `AuthorizationManagerAfterMethodInterceptor`
- **Object-level authorization**: enforce ownership checks (`repository.findByIdAndUserId()`), not just role checks. "User A cannot access User B's orders" is NOT covered by `@RolesAllowed` — this is CWE-639 Authorization Bypass Through User-Controlled Key / CWE-862 Missing Authorization

Source: Spring Security reference 6.4 (docs.spring.io/spring-security/reference/6.4/); OWASP Authorization Cheat Sheet; OWASP Access Control Cheat Sheet.

### 6.2 JWT Pitfalls (RFC 7519 + RFC 8725)
- **Always verify signature** server-side. Never use JJWT `parseClaimsJwt()` (unsigned) — use `parseClaimsJws()` (signed). See Gotcha G-05
- **Enforce expected algorithm**: reject `alg: none` (Gotcha G-04). Pin expected algorithm (e.g., RS256) in validation config. Treat the JWT header `alg` claim as attacker-controlled — compare against a server-pinned allowlist
- **Algorithm confusion (RS256 ↔ HS256)**: if a public RSA key is accepted as an HMAC secret, attacker forges tokens. Pin algorithm AND separate key stores per algorithm family
- **Validate claims**: `iss` (issuer), `aud` (audience), `exp` (expiration), `nbf` (not before), `jti` (for replay protection if revocation needed)
- **Token storage**: `HttpOnly` + `Secure` + `SameSite=Strict` cookies for web. `Authorization: Bearer` header for API clients. `HttpOnly` alone is NOT encryption — see Gotcha G-08
- **Token revocation**: maintain a blocklist for logout/compromised tokens, or use short-lived tokens + refresh tokens

Source: RFC 7519 (JWT); RFC 8725 JSON Web Token Best Current Practices (2020); JJWT reference (github.com/jwtk/jjwt).

### 6.3 OAuth 2.0 (RFC 9700)
- Published January 2025 as RFC 9700, replaces BCP and updates RFC 6819
- Mandates: exact redirect URI string matching (no wildcards, no suffix check); PKCE (RFC 7636) on all flows including confidential clients; NO Resource Owner Password Credentials grant; NO Implicit grant
- Prefer sender-constrained tokens: DPoP (RFC 9449) or mTLS (RFC 8705)

Source: RFC 9700 OAuth 2.0 Security Best Current Practice; Spring Authorization Server reference.

### 6.4 Session Management
- Call `sessionFixation().migrateSession()` to prevent session fixation attacks (CWE-384)
- Set session timeout appropriately (e.g., 30 minutes for web apps per OWASP ASVS 5.0 §3.3)
- Configure CORS at the Spring Security filter level, not just Spring MVC level (filter order matters — see Spring Security reference)
- Regenerate session ID on privilege change (login, MFA step-up)

Source: OWASP ASVS v5.0.0 §3 Session Management; OWASP Session Management Cheat Sheet; Spring Security reference 6.4 "Session Management".

---

## 7. Cryptography

Maps to `A02:2021` Cryptographic Failures / `A04:2025` Cryptographic Failures. Corresponds to CWE-327 (Broken/Risky Crypto Algorithm), CWE-328 (Reversible One-Way Hash), CWE-330 (Insufficient Randomness), CWE-295 (Improper Cert Validation), CWE-326 (Inadequate Encryption Strength).

### 7.1 Password Storage

Use **Argon2id** (preferred) with OWASP Password Storage Cheat Sheet 2024 parameters. Any one of:

| m (memory) | t (iterations) | p (parallelism) | Notes |
|---|---|---|---|
| 47104 KiB (46 MiB) | 1 | 1 | Strongest of the recommended profiles |
| **19456 KiB (19 MiB)** | **2** | **1** | **Minimum recommended baseline** — replaces the outdated "15 MiB" figure |
| 12288 KiB (12 MiB) | 3 | 1 | |
| 9216 KiB (9 MiB) | 4 | 1 | |
| 7168 KiB (7 MiB) | 5 | 1 | Lowest profile on legacy hardware |

Alternatives:
- **scrypt**: N=2^17 (128 MiB), r=8, p=1 baseline. Alternates: N=2^16 r=8 p=2; N=2^15 r=8 p=3; N=2^14 r=8 p=5; N=2^13 r=8 p=10
- **bcrypt**: minimum work factor 10; "as large as verification server performance will allow"
- **PBKDF2**: PBKDF2-HMAC-SHA256 = 600 000 iterations; PBKDF2-HMAC-SHA512 = 210 000 iterations; PBKDF2-HMAC-SHA1 = 1 400 000 iterations (SHA-1 still allowed here because the algorithm's security rests on HMAC, not SHA-1 collision resistance)

**Ban** MD5, SHA-1, SHA-256 alone for password storage — these are fast hash functions, not password hashing algorithms (CWE-328, CWE-916).

Use Spring Security's `DelegatingPasswordEncoder` (PasswordEncoderFactories.createDelegatingPasswordEncoder()) for forward-compatible hashing with automatic migration on next successful login.

Source: OWASP Password Storage Cheat Sheet (cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html); RFC 9106 Argon2; Spring Security reference 6.4 "Password Storage".

### 7.2 Randomness
- **`java.security.SecureRandom`**: for tokens, session IDs, nonces, CSRF tokens, password reset links, key generation (CWE-330)
- **`java.util.Random`**: NEVER for security contexts — predictable 48-bit LCG with observable seed recovery
- `UUID.randomUUID()` uses `SecureRandom` internally — safe for tokens. `UUID.nameUUIDFromBytes()` is MD5-based and deterministic — NOT for secrets

Source: `java.security.SecureRandom` Javadoc (JDK 21+); CERT MSC02-J "Generate strong random numbers".

### 7.3 Encryption

**Use**:
- AES-GCM authenticated encryption: `Cipher.getInstance("AES/GCM/NoPadding")` with a 12-byte (96-bit) unique IV per message (NEVER reuse an IV with the same key)
- AES-GCM-SIV for nonce-reuse resistance in environments where counter management is hard
- For hybrid scenarios: `ChaCha20-Poly1305` (JDK 11+)

**Forbidden ciphers / modes** (CWE-327):

| Forbidden | Reason | Allowed replacement |
|---|---|---|
| `DES`, `3DES` (`DESede`) | 56-/112-bit keys, NIST sunset Jan 2024 | AES-256 |
| `AES/ECB/*` (incl. bare `"AES"`, which defaults to ECB in OpenJDK) | Deterministic — reveals patterns | `AES/GCM/NoPadding` |
| `RC4` | Biased keystream | `AES/GCM` or `ChaCha20-Poly1305` |
| MD5-HMAC | Collision-broken | HMAC-SHA-256 or HMAC-SHA-512 |
| `NULL`, `EXPORT`, anonymous DH | No confidentiality/auth | Modern AEAD |
| Static / reused IVs | Breaks CTR/GCM confidentiality | Per-message `SecureRandom` 12-byte IV |

Never implement your own cryptographic algorithms.

Source: NIST SP 800-175B Rev. 1; NIST FIPS 197 (AES); RFC 7539 ChaCha20-Poly1305; OWASP Cryptographic Storage Cheat Sheet.

### 7.4 TLS (NIST SP 800-52 Rev. 2)
- **TLS 1.3** — RFC 8446 (August 2018). NIST SP 800-52 Rev. 2 required support for TLS 1.3 by **January 1, 2024**
- **Minimum allowed**: TLS 1.2 with AEAD cipher suites
- **Deprecated** (RFC 8996, 2021): SSLv3, TLS 1.0, TLS 1.1
- Use the platform trust store (`cacerts`) for certificate validation — NEVER install a custom `TrustManager` that returns without checking (`TrustAllCerts` is a P0 finding — CWE-295)
- Pin TLS minimum version explicitly in `HttpClient.newBuilder().sslContext(...)` / `RestClient` — don't rely on JVM defaults across versions

Source: NIST SP 800-52 Rev. 2 "Guidelines for the Selection, Configuration, and Use of Transport Layer Security (TLS) Implementations" (Aug 2019); RFC 8446 TLS 1.3; RFC 8996 Deprecating TLS 1.0 and TLS 1.1 (Mar 2021); OWASP Transport Layer Security Cheat Sheet.

---

## 8. Secrets Management

Maps to `A02:2021` Cryptographic Failures + `A05:2021` Security Misconfiguration / `A02:2025` Security Misconfiguration + `A04:2025` Cryptographic Failures. Corresponds to CWE-798 (Hardcoded Credentials), CWE-522 (Insufficiently Protected Credentials).

### 8.1 Detection
- Run **TruffleHog** (trufflesecurity.com) or **Gitleaks** (gitleaks.io) as a pre-commit hook and in CI. Scan full git history, not just HEAD — rewriting history after leak is required because the blob remains in the object store otherwise
- TruffleHog combines entropy + regex + 700+ verifier plugins (live-probes the API to confirm validity). Gitleaks uses regex + TOML rule sets
- Ban hardcoded strings matching patterns: AWS keys (`AKIA[0-9A-Z]{16}`), private keys (`-----BEGIN (RSA|EC|OPENSSH|PGP) PRIVATE KEY-----`), `password\s*=\s*["']\S+["']`, JDBC URLs with embedded credentials

Source: TruffleHog project (github.com/trufflesecurity/trufflehog); Gitleaks project (github.com/gitleaks/gitleaks).

### 8.2 Storage
- **Never** commit: `.env` files, `application-prod.yml` with real credentials, keystores, private keys
- Inject secrets at runtime via:
  - HashiCorp Vault (spring-cloud-starter-vault-config)
  - AWS Secrets Manager / Parameter Store (spring-cloud-aws-starter-secrets-manager)
  - Azure Key Vault (spring-cloud-azure-starter-keyvault-secrets)
  - Google Secret Manager (spring-cloud-gcp-starter-secretmanager)
  - Kubernetes Secrets mounted as files via `spring.config.import=configtree:/etc/secrets/` (Spring Boot 2.4+)
- Spring Cloud Config: use encrypted backend (`{cipher}...`), not plaintext properties
- In logs/exceptions: mask or redact any potentially sensitive values (SLF4J MDC + custom converter, or Spring Cloud Sleuth baggage sanitizers)

Source: Spring Boot reference "Externalized Configuration — Using ConfigTree"; Spring Cloud Vault reference.

### 8.3 What to Flag in Review
- `application.yml` / `application.properties` with values that look like real credentials (plaintext passwords, API keys, OAuth client secrets)
- Connection strings with embedded username/password (`jdbc:postgresql://user:pass@host/db`)
- API keys or tokens in source code (even in test code — use test-specific secrets or WireMock)
- `src/test/resources` with production-like credentials
- Dockerfile `ENV SECRET=...` lines (layer metadata is plaintext)
- CI config (`.github/workflows/`) referencing secrets as string literals instead of `${{ secrets.X }}`

Source: OWASP Secrets Management Cheat Sheet.

---

## 9. Serialization & Deserialization

Maps to `A08:2021` Software and Data Integrity Failures / `A08:2025` Software or Data Integrity Failures. Corresponds to CWE-502 (Deserialization of Untrusted Data).

### 9.1 Java Native Serialization
- **Ban `ObjectInputStream.readObject()` on untrusted input** — a top source of Java RCE vulnerabilities
- If unavoidable: use `ObjectInputFilter` (**JEP 290**, Java 9+) with an explicit allowlist of permitted classes
- For complex, library-aware filtering use a filter factory (**JEP 415**, Java 17+; system property `jdk.serialFilterFactory`) which lets each deserialization call install its own context-specific filter
- Set JVM-wide filter via `-Djdk.serialFilter=maxdepth=5;maxarray=100000;!*` (deny-all, then allow specific classes with `com.example.Allowed1;com.example.Allowed2;...;!*`)

Source: Oracle "Serialization Filtering" (docs.oracle.com/en/java/javase/21/core/java-serialization-filters.html); JEP 290 (JDK 9); JEP 415 (JDK 17); CERT SER12-J "Prevent deserialization of untrusted data".

### 9.2 Jackson Polymorphic Deserialization

| Configuration | Safety | Notes |
|---|---|---|
| `ObjectMapper` default (no default typing) | Safe | Each `readValue(json, Class)` deserializes only the target type |
| `@JsonTypeInfo(use = Id.NAME)` with explicit `@JsonSubTypes` | Safe | Named discriminator; attacker cannot choose arbitrary class |
| `@JsonTypeInfo(use = Id.CLASS)` + no validator | UNSAFE (gadget RCE) | Class name in JSON — attacker controls which class is instantiated |
| `enableDefaultTyping()` (deprecated since 2.10) | UNSAFE | Root cause of dozens of Jackson CVEs |
| `activateDefaultTyping(ptv)` with `BasicPolymorphicTypeValidator` | Conditional | Only safe if validator restricts to a known allowlist of base types |
| `@JsonTypeInfo(use = Id.CLASS)` + validator via `activateDefaultTyping` alone | UNSAFE | Validator NOT called for `@JsonTypeInfo` annotations — must also call `ObjectMapper.setPolymorphicTypeValidator()` on the builder |

Additional rules:
- Enable `DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES` (default in Spring Boot) — breaks "add new field to exploit" gadgets
- Avoid deserializing to `Object`-typed fields, `Map<String, Object>`, or raw `List`. Always specify concrete target types in `readValue()`
- Keep Jackson updated — FasterXML maintains `SubTypeValidator` with a growing list of known dangerous gadget classes (c3p0, Logback JNDI, Hibernate, Spring AOP, etc.)

Source: FasterXML Jackson wiki "Polymorphic Deserialization CVE Criteria" (github.com/FasterXML/jackson/wiki/Jackson-Polymorphic-Deserialization-CVE-Criteria); Snyk research "Serialization and deserialization in Java: explaining the Java deserialize vulnerability" (Brian Vermeer, snyk.io).

### 9.3 Protobuf / Avro / gRPC
- Set message size limits (`maxInboundMessageSize` in gRPC `ServerBuilder` / `ManagedChannelBuilder`) to prevent denial-of-service via huge payloads (CWE-400)
- Validate deserialized values — Protobuf doesn't enforce constraints (required-range, non-empty), only structure
- Avro schema evolution: reject unknown schema IDs; pin expected schema at consumer boundary

Source: gRPC-Java reference (grpc.io); OWASP Deserialization Cheat Sheet.

---

## 10. Dependency & Supply Chain Security

Maps to `A06:2021` Vulnerable and Outdated Components / `A03:2025` Software Supply Chain Failures (NEW and broader category in 2025 RC1 — covers build pipeline, package registries, and CI/CD integrity in addition to CVE tracking). Corresponds to CWE-1104 (Use of Unmaintained Third-Party Components).

### 10.1 Scanning
- Run **OWASP Dependency-Check** (`dependency-check-maven`/`dependency-check-gradle`) or **Snyk** or **Dependency-Track** in CI on every PR
- Fail the build on **critical** and **high** CVEs (CVSS 3.1 >= 7.0)
- Enable **GitHub Dependabot** for automated upgrade PRs with CVE annotations
- Prefer tools with **reachability analysis** (Snyk Code, Semgrep Supply Chain) to prioritize CVEs in code paths you actually invoke

Source: OWASP Dependency-Check (owasp.org/www-project-dependency-check/); OWASP Dependency-Track (dependencytrack.org); GitHub Dependabot docs.

### 10.2 SBOM and Provenance
- Generate a Software Bill of Materials (SBOM) per build: CycloneDX (cyclonedx.org) via `cyclonedx-maven-plugin` or SPDX
- Store SBOMs with release artefacts for CVE correlation (Dependency-Track consumes CycloneDX)
- For stronger supply-chain guarantees: SLSA provenance (slsa.dev), sigstore/cosign signatures on artefacts

Source: OWASP CycloneDX; SLSA framework (slsa.dev); sigstore/cosign project.

### 10.3 Suppression Management
- Suppress false positives only with documented justification and expiration date
- Review suppressions quarterly
- Track suppressed CVEs in a file committed to the repo (`dependency-check-suppressions.xml`)

### 10.4 Transitive Dependencies
- Run `mvn dependency:tree` / `gradle dependencies` regularly — know what you're shipping
- Transitive dependencies inherit their vulnerabilities. A CVE in `commons-text` pulled in by `spring-boot-starter` affects you even if you never import it directly. See Gotcha G-16.

### 10.5 Historical Supply-Chain Post-Mortems
- **xz-utils backdoor / CVE-2024-3094** (disclosed March 29, 2024): sshd backdoor in `xz` 5.6.0/5.6.1 via obfuscated autoconf + test-fixture binary payload. Inserted by `Jia Tan` account that built maintainer trust over ~2 years (2021–2024). Discovered by Andres Freund (Microsoft/PostgreSQL) via SSH login latency + Valgrind errors. CVSS 10.0. Lesson: long-tail maintainer trust is now a supply-chain attack surface
- **Log4Shell / CVE-2021-44228** (December 2021): Log4j 2.x JNDI lookup + LDAP remote class loading. Patch: 2.16.0 minimum; 2.17.1 final. Lesson: transitive dependency with string-interpolated logging is a gadget surface
- **Spring4Shell / CVE-2022-22965** (March 31, 2022): Spring Framework 5.2.x < 5.2.20, 5.3.x < 5.3.18 data-binding RCE on JDK 9+. CVSS 9.8. Lesson: `WebDataBinder` default allowlist was insufficient against `class.module.classLoader`-style gadget chains
- **Oracle CPU cadence**: Critical Patch Updates on the third Tuesday of January, April, July, October. January 2026 CPU includes 11 Java SE patches (all remote-no-auth), including CVE-2026-21945 (mTLS client-cert issue). Track via oracle.com/security-alerts/

Source: CVE records on nvd.nist.gov; original disclosure posts (Andres Freund on oss-security mailing list for CVE-2024-3094; Apache Log4j Security Vulnerabilities page); Oracle Critical Patch Updates.

---

## 11. Spring Boot Actuator & Operational Hardening

Maps to `A05:2021` Security Misconfiguration / `A02:2025` Security Misconfiguration. Corresponds to CWE-200 (Exposure of Sensitive Information), CWE-16 (Configuration).

### 11.1 Endpoint Exposure Matrix

| Endpoint | Unauthenticated | Authenticated (admin only) | Reason |
|---|---|---|---|
| `/actuator/health` (with `show-details=never`) | OK | OK | Liveness/readiness probe |
| `/actuator/info` | OK (only if curated) | OK | Build metadata |
| `/actuator/prometheus` | LAN-only | OK | Metrics scraping |
| `/actuator/env` | NEVER | Only admin | Dumps config + secrets masked imperfectly |
| `/actuator/heapdump` | NEVER | Only admin | Entire JVM heap (session tokens, keys) |
| `/actuator/threaddump` | NEVER | Only admin | Thread names leak user IDs, SQL |
| `/actuator/loggers` | NEVER | Only admin | Log-level changes enable noisy recon |
| `/actuator/mappings` | NEVER | Only admin | Enumerates every controller route |
| `/actuator/beans` | NEVER | Only admin | Full Spring context |
| `/actuator/configprops` | NEVER | Only admin | Effective config incl. DB URLs |
| `/actuator/shutdown` | NEVER | NEVER | Remote DoS |
| `/actuator/gateway/routes` | NEVER | Only admin | Spring Cloud Gateway route table |

Wiz 2025 cloud scan found 1 in 4 Actuator deployments publicly exposed with dangerous endpoints active.

### 11.2 Separation of Management Traffic
- Set `management.server.port` to a different port (e.g., 9090) distinct from `server.port`
- Bind `management.server.address=127.0.0.1` or a private network
- Apply `SecurityFilterChain` with `EndpointRequest.toAnyEndpoint().hasRole("ACTUATOR_ADMIN")` for everything except the narrow safe list

### 11.3 CVE-2025-22235
`EndpointRequest.to(<endpoint>)` matcher creates the WRONG Spring Security matcher when the named endpoint is not exposed under `management.endpoints.web.exposure.include`. Result: security rules may not apply as intended. Mitigation: upgrade to patched Spring Boot 3.4.x / 3.3.x release per the Spring advisory; audit every `EndpointRequest.to()` usage.

Source: Spring Security Advisory CVE-2025-22235.

### 11.4 Other Debug / Dev Endpoints
Remove or auth-gate in production profiles:
- `/h2-console` (H2 web console — arbitrary SQL)
- `/swagger-ui`, `/v3/api-docs` (OpenAPI schema leak)
- `/graphiql`, `/graphql` introspection (schema + resolver leak)
- `/error` with `server.error.include-stacktrace=always` (information disclosure — CWE-209)

Source: Spring Boot Actuator reference (docs.spring.io/spring-boot/reference/actuator/); OWASP API Security Top 10 (API7:2023 Security Misconfiguration).

---

## 12. Exceptional Condition Handling

New category in `A10:2025` Mishandling of Exceptional Conditions (RC1 — covers 24 CWEs including CWE-703 Improper Check or Handling of Exceptional Conditions, CWE-248 Uncaught Exception, CWE-755 Improper Handling of Exceptional Conditions).

### 12.1 Never Silently Swallow
- Empty `catch (Exception e) {}` / `catch (Throwable t) { /* ignore */ }` is a P1 finding
- Re-throw, wrap with context, or log at `error` level with full stack trace. Silent swallowing hides both bugs and active attacks (scanner probes, injection attempts)

Source: CWE-390 Detection of Error Condition Without Action; CERT ERR00-J "Do not suppress or ignore checked exceptions".

### 12.2 Don't Leak Internals
- Client-facing error responses must NOT include stack traces, SQL, file paths, class names, or framework versions (CWE-209 Information Exposure Through an Error Message)
- Spring Boot: set `server.error.include-stacktrace=never`, `server.error.include-message=never`, `server.error.include-binding-errors=never` in production profile
- Provide a correlation ID (MDC `traceId`) in the response so support can find the full trace in logs

Source: OWASP Error Handling Cheat Sheet; Spring Boot reference "Error Handling".

### 12.3 Fail Closed
- Authorization / authentication / crypto paths: on exception, DENY. Never default to "allow" on unexpected state
- Circuit breakers and fallbacks must not widen trust (e.g., fallback returning `Optional<User>` with elevated role)

Source: OWASP ASVS v5.0.0 §1 Secure Software Development.

### 12.4 Log Exceptions, Don't Leak Stack
- Log full exceptions server-side (`log.error("context", e)`) — do NOT return them in the HTTP body
- Mask sensitive values in exception messages (`IllegalArgumentException("invalid token xxx-redacted")`, not the full token)

Source: OWASP Logging Cheat Sheet.

---

## 13. Security Logging & Alerting

Maps to `A09:2021` Security Logging and Monitoring Failures / `A09:2025` Security Logging & Alerting Failures (renamed in 2025 RC1 to emphasise alerting, not just capture). Corresponds to CWE-778 (Insufficient Logging), CWE-532 (Insertion of Sensitive Information into Log File).

### 13.1 Events to Log
- Authentication: success, failure, MFA step, password reset request
- Authorization denials (403) with the actor + resource + action
- Session lifecycle: login, logout, session invalidation
- Input validation failures that indicate attack (SQL-injection-shaped payload, path traversal attempt)
- Configuration changes, privilege changes, user create/delete
- Crypto key rotation / use of fallback legacy algorithm

Source: OWASP Logging Cheat Sheet; OWASP ASVS v5.0.0 §7 Logging and Monitoring.

### 13.2 What NOT to Log (CWE-532)
- Passwords, plaintext OR hashed (even hashes are sensitive for rainbow-table work)
- Session tokens, JWTs, CSRF tokens, API keys
- Personally identifiable information (PII) beyond what is required for the audit trail. Consider GDPR / CCPA scope
- Full credit-card numbers (PCI DSS Req 3.3 — mask to first 6 + last 4)

Source: OWASP Logging Cheat Sheet; PCI DSS v4.0 Req 3.

### 13.3 Log Integrity
- Forward to an append-only sink (SIEM, immutable cloud logging) — do NOT rely only on local `/var/log` which attackers can rotate
- Timestamp with monotonic + wall clock + timezone. Use `Instant` + `ZoneOffset.UTC`

Source: OWASP Logging Cheat Sheet; NIST SP 800-92 "Guide to Computer Security Log Management".

### 13.4 Alerting
- Detections must page a human — logs without alerting are "write-only memory"
- Common threshold alerts: failed-login rate per user / per IP, 403 burst, unusual admin-endpoint access, spike in 5xx after a deploy

Source: `A09:2025` guidance (OWASP Top 10 2025 RC1 — owasp.org/Top10/2025/).

---

## 14. Quick Security Review Heuristic

When reviewing Java code for security, scan in this order:

1. **User input flows**: trace every request parameter, header, path variable, request body — where does it go? SQL? File system? HTTP call? Shell? Log?
2. **Authentication bypass**: any new endpoint missing auth? Any `.permitAll()` without justification? (See Gotcha G-18)
3. **Authorization gaps**: role check present? Object-level ownership enforced? (CWE-862 #4 on CWE Top 25 2025)
4. **SQL / LDAP / command injection**: any string concatenation with external input near query/command execution?
5. **Deserialization**: any `readObject()` without `ObjectInputFilter`, `enableDefaultTyping()`, `@JsonTypeInfo(use = CLASS)` without validator?
6. **Secrets in code**: any hardcoded passwords, API keys, tokens? In config files, test resources, comments, Dockerfile?
7. **Cryptography**: password hashing with proper algorithm (Argon2id 19 MiB baseline)? `SecureRandom` for tokens? No `TrustAllCerts`? AEAD mode (AES-GCM), no ECB?
8. **Dependency CVEs**: has the dependency scan passed? Any new dependencies added — including transitive ones?
9. **Error responses**: do error messages leak internal details (stack traces, SQL, file paths)? (§12)
10. **Security headers**: strict CSP, HSTS, `X-Content-Type-Options`, `X-Frame-Options`/`frame-ancestors` configured?
11. **Logging**: security-relevant events captured? No secrets in logs? Alerts wired? (§13)
12. **Actuator**: production profile disables or auth-gates `/env`, `/heapdump`, `/threaddump`, `/loggers`, `/mappings`, `/beans`, `/configprops`, `/shutdown`? (§11)

---

## 15. Bug-Pattern Code Pairs (`bad:` / `good:`)

Each pattern shows the anti-pattern and its fix. Reviewers should search diffs for the `bad:` shape.

### 15.1 SQL Concatenation
bad:
```java
String sql = "SELECT * FROM users WHERE email = '" + email + "'";
stmt.executeQuery(sql);
```
good:
```java
String sql = "SELECT * FROM users WHERE email = ?";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    ps.setString(1, email);
    try (ResultSet rs = ps.executeQuery()) { ... }
}
```
Source: OWASP SQL Injection Prevention Cheat Sheet; CWE-89.

### 15.2 `Runtime.exec(String)` with Shell Interpolation
bad:
```java
Runtime.getRuntime().exec("sh -c 'convert " + userFile + " out.pdf'");
```
good:
```java
new ProcessBuilder("convert", userFile, "out.pdf").start();
```
Source: CERT IDS07-J; CWE-78.

### 15.3 Unescaped `th:utext` in Thymeleaf
bad:
```html
<div th:utext="${userBio}"></div>
```
good:
```html
<div th:text="${userBio}"></div>
<!-- or if HTML is required, sanitize server-side first: -->
<div th:utext="${@htmlSanitizer.clean(userBio)}"></div>
```
Source: Thymeleaf reference "Using texts" §3.4; OWASP XSS Prevention Cheat Sheet.

### 15.4 `TrustAllCerts` `X509TrustManager`
bad:
```java
TrustManager tm = new X509TrustManager() {
    public void checkClientTrusted(X509Certificate[] c, String a) {}
    public void checkServerTrusted(X509Certificate[] c, String a) {}
    public X509Certificate[] getAcceptedIssuers() { return new X509Certificate[0]; }
};
SSLContext ctx = SSLContext.getInstance("TLS");
ctx.init(null, new TrustManager[]{tm}, new SecureRandom());
```
good:
```java
// Use the default platform trust manager; do NOT override validation.
SSLContext ctx = SSLContext.getDefault();
HttpClient client = HttpClient.newBuilder().sslContext(ctx).build();
```
Source: CERT SEC07-J; CWE-295; OWASP Transport Layer Security Cheat Sheet.

### 15.5 `java.util.Random` for Tokens
bad:
```java
Random r = new Random();
byte[] token = new byte[32];
r.nextBytes(token);
```
good:
```java
SecureRandom r = new SecureRandom();
byte[] token = new byte[32];
r.nextBytes(token);
```
Source: CERT MSC02-J; CWE-330; `java.security.SecureRandom` Javadoc.

### 15.6 `Cipher.getInstance("AES")` Defaults to ECB
bad:
```java
Cipher c = Cipher.getInstance("AES"); // defaults to AES/ECB/PKCS5Padding in OpenJDK
c.init(Cipher.ENCRYPT_MODE, key);
byte[] ct = c.doFinal(pt);
```
good:
```java
byte[] iv = new byte[12];
new SecureRandom().nextBytes(iv);
Cipher c = Cipher.getInstance("AES/GCM/NoPadding");
c.init(Cipher.ENCRYPT_MODE, key, new GCMParameterSpec(128, iv));
byte[] ct = c.doFinal(pt);
// prepend iv to ct for transport; never reuse iv with same key
```
Source: NIST SP 800-38D (GCM); OWASP Cryptographic Storage Cheat Sheet; CWE-327.

### 15.7 `MessageDigest.getInstance("MD5")` for Passwords
bad:
```java
String hash = Hex.encodeHexString(MessageDigest.getInstance("MD5").digest(password.getBytes()));
```
good:
```java
PasswordEncoder enc = PasswordEncoderFactories.createDelegatingPasswordEncoder();
String hash = enc.encode(password); // uses bcrypt by default; migration-friendly
```
Source: OWASP Password Storage Cheat Sheet; Spring Security 6.4 reference; CWE-328.

### 15.8 `ObjectInputStream.readObject()` Without JEP 290 Filter
bad:
```java
try (ObjectInputStream ois = new ObjectInputStream(request.getInputStream())) {
    Object payload = ois.readObject();
}
```
good:
```java
try (ObjectInputStream ois = new ObjectInputStream(request.getInputStream())) {
    ObjectInputFilter filter = ObjectInputFilter.Config.createFilter(
        "maxdepth=5;maxarray=1024;com.example.Allowed;!*");
    ois.setObjectInputFilter(filter);
    Object payload = ois.readObject();
}
```
Source: Oracle "Serialization Filtering" (JDK 21 docs); JEP 290; CERT SER12-J.

### 15.9 Jackson `enableDefaultTyping()`
bad:
```java
ObjectMapper om = new ObjectMapper();
om.enableDefaultTyping(); // deprecated since 2.10 for a reason
User u = om.readValue(json, User.class);
```
good:
```java
PolymorphicTypeValidator ptv = BasicPolymorphicTypeValidator.builder()
    .allowIfBaseType(com.example.Event.class)
    .build();
ObjectMapper om = JsonMapper.builder()
    .polymorphicTypeValidator(ptv)
    .activateDefaultTyping(ptv, ObjectMapper.DefaultTyping.NON_FINAL, JsonTypeInfo.As.PROPERTY)
    .build();
// Prefer: avoid default typing entirely and use @JsonTypeInfo(use = Id.NAME) + @JsonSubTypes on the base class.
```
Source: FasterXML Jackson wiki "Polymorphic Deserialization CVE Criteria".

### 15.10 JJWT `parseClaimsJwt` (Unsigned)
bad:
```java
Jws<Claims> jws = Jwts.parser()
    .setSigningKey(key)
    .parseClaimsJwt(token); // NOTE: "Jwt" not "Jws" — accepts unsigned tokens
```
good:
```java
Jws<Claims> jws = Jwts.parser()
    .verifyWith(publicKey)
    .requireIssuer("https://auth.example.com")
    .requireAudience("api")
    .build()
    .parseSignedClaims(token); // JJWT 0.12+ signed variant
```
Source: JJWT reference (github.com/jwtk/jjwt); RFC 8725 §3.1 (reject `alg: none`).

### 15.11 Wide-Open `.permitAll()` Before `.authenticated()`
bad:
```java
http.authorizeHttpRequests(a -> a
    .requestMatchers("/api/**").permitAll()   // opens the whole API
    .anyRequest().authenticated()
);
```
good:
```java
http.authorizeHttpRequests(a -> a
    .requestMatchers("/api/public/**").permitAll()  // narrow prefix only
    .requestMatchers("/api/admin/**").hasRole("ADMIN")
    .anyRequest().authenticated()
);
```
Source: Spring Security reference 6.4 "Authorize HttpServletRequests"; Gotcha G-18.

### 15.12 CSRF Disabled on State-Changing Endpoint
bad:
```java
http.csrf(AbstractHttpConfigurer::disable); // blanket disable
```
good:
```java
// Session-based: keep CSRF on.
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
    .csrfTokenRequestHandler(new XorCsrfTokenRequestAttributeHandler())
);
// API with JWT in Authorization header (not cookies): CSRF disable is OK because browsers don't auto-attach that header cross-origin.
http.csrf(csrf -> csrf.ignoringRequestMatchers("/api/**")).sessionManagement(s -> s.sessionCreationPolicy(STATELESS));
```
Source: Spring Security reference 6.4 "CSRF"; OWASP CSRF Prevention Cheat Sheet.

### 15.13 Plaintext Secrets in `application.yml`
bad:
```yaml
spring:
  datasource:
    url: jdbc:postgresql://prod-db/app
    username: app
    password: hunter2                 # plaintext in repo
  security:
    oauth2:
      client:
        registration:
          github:
            client-secret: ghp_abc123  # plaintext in repo
```
good:
```yaml
spring:
  config:
    import: "configtree:/etc/secrets/"
  datasource:
    url: jdbc:postgresql://prod-db/app
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```
Source: Spring Boot reference "Externalized Configuration — ConfigTree"; OWASP Secrets Management Cheat Sheet.

### 15.14 `HttpURLConnection` Without TLS Minimum Version Pinning
bad:
```java
HttpURLConnection c = (HttpURLConnection) new URL("https://api.example.com").openConnection();
// relies on JVM default TLS protocols, which may include 1.0/1.1 on older JDKs
```
good:
```java
SSLContext ctx = SSLContext.getInstance("TLSv1.3");
ctx.init(null, null, new SecureRandom());
HttpClient client = HttpClient.newBuilder()
    .sslContext(ctx)
    .connectTimeout(Duration.ofSeconds(10))
    .build();
```
Source: NIST SP 800-52 Rev. 2; RFC 8446; RFC 8996 (deprecate TLS 1.0/1.1).

### 15.15 `ServletContext.getRealPath(userInput)` Path Traversal
bad:
```java
String real = request.getServletContext().getRealPath(userPath); // no sandbox check
Files.copy(Path.of(real), response.getOutputStream());
```
good:
```java
Path base = Path.of(request.getServletContext().getRealPath("/downloads")).toRealPath();
Path target = base.resolve(userPath).normalize();
if (!target.startsWith(base)) {
    throw new SecurityException("Path traversal attempt");
}
Files.copy(target, response.getOutputStream());
```
Source: OWASP File Upload Cheat Sheet; CERT FIO16-J; CWE-22.

---

## 16. Review Checklist Items

Per-section actionable checks. Use as a diff-review rubric.

### 16.1 Injection
- [ ] Every SQL query is parameterized (`PreparedStatement` / JPA `:name`); no string concatenation with user input. Source: OWASP SQL Injection Prevention Cheat Sheet
- [ ] No `Statement.execute(String)` or `Statement.executeQuery(String)` with dynamic SQL. Source: CWE-89
- [ ] Dynamic `ORDER BY` / column names validated against an allowlist. Source: CWE-89
- [ ] `LdapQueryBuilder` used for LDAP; no raw filter-string concatenation. Source: OWASP LDAP Injection Prevention Cheat Sheet
- [ ] `ProcessBuilder` with argument arrays used instead of `Runtime.exec(String)`. Source: CERT IDS07-J
- [ ] No `@Value("#{...}")` SpEL with user input; no `SpelExpressionParser.parseExpression(userInput)`. Source: CWE-94

### 16.2 XSS
- [ ] No `th:utext` on user-controlled data. Source: Thymeleaf reference
- [ ] `Encode.forJavaScript` used in script contexts (not `Encode.forHtml`). Source: OWASP XSS Prevention Cheat Sheet
- [ ] Strict CSP with nonce or hash; no `unsafe-inline`, no `unsafe-eval`. Source: OWASP CSP Cheat Sheet
- [ ] `X-Content-Type-Options: nosniff`, `frame-ancestors 'none'`/`X-Frame-Options: DENY` set. Source: OWASP Secure Headers Project

### 16.3 CSRF
- [ ] CSRF protection enabled for session-based auth; no blanket `csrf().disable()`. Source: Spring Security reference 6.4
- [ ] If JWT in cookie, CSRF on. If JWT in `Authorization` header, CSRF can be off for that matcher. Source: OWASP CSRF Prevention Cheat Sheet; Gotcha G-13
- [ ] All state-changing verbs require a CSRF token when using cookies. Source: Spring Security reference

### 16.4 SSRF
- [ ] Outbound URLs validated against allowlist of hosts + schemes; link-local + RFC 1918 ranges rejected. Source: OWASP SSRF Prevention Cheat Sheet
- [ ] DNS re-validated on connect, not just at check. Source: OWASP SSRF Prevention Cheat Sheet
- [ ] Follow-redirects disabled or redirect target re-validated (CWE-601). Source: OWASP

### 16.5 Input Validation
- [ ] Every `@RequestBody` / `@RequestParam` has Jakarta Bean Validation constraints + `@Valid`. Source: Jakarta Validation 3.1
- [ ] Path canonicalization + prefix check before any filesystem read/write. Source: CERT FIO16-J; CWE-22
- [ ] File uploads: content-type detected via magic bytes; size limit enforced; stored outside webroot. Source: OWASP File Upload Cheat Sheet
- [ ] Header values rejecting `\r`/`\n`. Source: CWE-113

### 16.6 Authentication & Authorization
- [ ] `SecurityFilterChain` bean present; no legacy `WebSecurityConfigurerAdapter`. Source: Spring Security reference 6.4
- [ ] `.anyRequest().authenticated()` is the last matcher. Source: Spring Security reference 6.4
- [ ] Every `.permitAll()` has a comment explaining why. Source: OWASP Access Control Cheat Sheet
- [ ] `@PreAuthorize` on service layer, not only controller. Source: Spring Security reference 6.4
- [ ] Ownership checks on every data-access by ID (`findByIdAndUserId` or equivalent). Source: CWE-639
- [ ] JWT: `parseSignedClaims` / `parseClaimsJws` only; algorithm pinned; `iss`/`aud`/`exp`/`nbf` validated. Source: RFC 8725
- [ ] Session fixation mitigation enabled (`sessionFixation().migrateSession()`). Source: Spring Security reference
- [ ] OAuth flows: exact redirect URI match, PKCE on, no ROPC, no Implicit. Source: RFC 9700

### 16.7 Cryptography
- [ ] Password storage: Argon2id with ≥19 MiB memory / 2 iters (or stronger profile). Source: OWASP Password Storage Cheat Sheet
- [ ] No MD5 / SHA-1 / bare SHA-256 for passwords. Source: CWE-328
- [ ] `SecureRandom` for any security-relevant random. Source: CERT MSC02-J
- [ ] `Cipher.getInstance("AES/GCM/NoPadding")` — no ECB, no bare `"AES"`. Source: NIST SP 800-38D
- [ ] Unique 12-byte IV per AES-GCM message. Source: NIST SP 800-38D
- [ ] TLS minimum 1.2; TLS 1.3 supported. Source: NIST SP 800-52 Rev. 2
- [ ] No custom `TrustManager` that accepts all. Source: CWE-295

### 16.8 Secrets
- [ ] TruffleHog or Gitleaks scan configured in CI + pre-commit. Source: trufflesecurity.com / gitleaks.io
- [ ] `application-*.yml` reference env vars / configtree, not literals. Source: Spring Boot reference
- [ ] No secrets in test resources, Dockerfile `ENV`, CI config literals. Source: OWASP Secrets Management Cheat Sheet

### 16.9 Serialization
- [ ] Every `ObjectInputStream` on untrusted data uses `ObjectInputFilter` (JEP 290) with explicit allowlist. Source: JEP 290; CERT SER12-J
- [ ] No `ObjectMapper.enableDefaultTyping()`; polymorphic types use `@JsonTypeInfo(use = Id.NAME)` + `@JsonSubTypes`. Source: FasterXML Jackson wiki
- [ ] `FAIL_ON_UNKNOWN_PROPERTIES` left on (Spring Boot default). Source: Jackson docs
- [ ] gRPC `maxInboundMessageSize` set. Source: gRPC-Java reference

### 16.10 Dependencies
- [ ] OWASP Dependency-Check / Snyk configured; build fails on high/critical. Source: OWASP Dependency-Check
- [ ] Dependabot or equivalent PR bot enabled. Source: GitHub Dependabot docs
- [ ] Suppression file reviewed quarterly; every suppression has expiration. Source: OWASP Dependency-Check suppression schema
- [ ] SBOM generated per release (CycloneDX). Source: OWASP CycloneDX

### 16.11 Actuator / Hardening
- [ ] `management.server.port` separate from `server.port`; bound to private address. Source: Spring Boot Actuator reference
- [ ] `/env`, `/heapdump`, `/threaddump`, `/loggers`, `/mappings`, `/beans`, `/configprops`, `/shutdown` NOT exposed unauthenticated. Source: Spring Boot Actuator reference
- [ ] Spring Boot patched against CVE-2025-22235. Source: Spring advisory
- [ ] `/h2-console`, `/swagger-ui`, `/graphiql` disabled in production profile. Source: Spring Boot reference

### 16.12 Exceptional Conditions
- [ ] No empty `catch` blocks. Source: CERT ERR00-J
- [ ] Production error responses omit stack trace / SQL / file path. Source: CWE-209
- [ ] Authorization / crypto paths fail closed on exception. Source: OWASP ASVS v5.0.0

### 16.13 Logging & Alerting
- [ ] Security events logged (login success/fail, 403, admin action). Source: OWASP Logging Cheat Sheet
- [ ] No passwords / tokens / PII in logs. Source: CWE-532
- [ ] Logs forwarded to immutable sink. Source: NIST SP 800-92
- [ ] Alerts wired for failed-login burst, 403 burst, Actuator access. Source: OWASP Top 10 2025 RC1 `A09:2025`

---

## 17. Key References

- OWASP Top 10:2021 (current released edition) — Introduction: https://owasp.org/Top10/
- OWASP Top 10:2025 RC1 (draft as of April 2026, not finalised) — https://owasp.org/Top10/2025/
- OWASP ASVS v5.0.0 (released May 2025 at Global AppSec EU) — https://owasp.org/www-project-application-security-verification-standard/
- OWASP Cheat Sheet Series — https://cheatsheetseries.owasp.org/
- CWE Top 25 2025 (published Dec 15, 2025 by CISA + MITRE HSSEDI) — https://cwe.mitre.org/top25/
- NIST SP 800-52 Rev. 2 (TLS, Aug 2019) — https://csrc.nist.gov/publications/detail/sp/800-52/rev-2/final
- RFC 7519 (JWT), RFC 8725 (JWT BCP), RFC 9700 (OAuth 2.0 Security BCP, Jan 2025)
- RFC 8446 (TLS 1.3), RFC 8996 (deprecate TLS 1.0/1.1)
- JEP 290 (Serialization Filters, JDK 9), JEP 415 (Context-Specific Filters, JDK 17)
- Spring Security 6.4 reference — https://docs.spring.io/spring-security/reference/6.4/
- Spring Boot reference — https://docs.spring.io/spring-boot/reference/
- SEI CERT Oracle Coding Standard for Java — https://wiki.sei.cmu.edu/confluence/display/java/
- FasterXML Jackson wiki "Polymorphic Deserialization CVE Criteria"
- David Coffin, *Expert Oracle and Java Security*, Apress 2011, ISBN 978-1-4302-3831-6 — Java-platform security architecture, SecurityManager, access control, code signing

---

## 18. Research Questions to Cross-Check Against Sources

The checklist is organised to answer the following questions about a diff or PR. Each question is followed by the corresponding checklist section(s) and the primary source(s) used to verify the answer.

### A. Input validation
1. **Is every `@RequestBody` / `@RequestParam` annotated with Jakarta Bean Validation constraints and `@Valid`?** (§5.1, §16.5) — source: Jakarta Validation 3.1 spec; Hibernate Validator 9 reference.
2. **Do filesystem reads/writes canonicalize the path AND verify it stays inside a base directory?** (§5.3, §15.15) — source: CERT FIO16-J; CWE-22.
3. **Are HTTP header values rejected when they contain CR (`%0D`) or LF (`%0A`)?** (§5.4) — source: CWE-113.
4. **Are file uploads type-checked via magic bytes (e.g., Apache Tika), size-limited, and stored outside the webroot?** (§5.5) — source: OWASP File Upload Cheat Sheet.
5. **Is validation done on the server (never trusting client-side)?** (§5.1) — source: OWASP Input Validation Cheat Sheet.

### B. Injection classes
6. **Is every SQL query parameterized (`PreparedStatement` / JPA `:name`)?** (§1.1, §15.1, §16.1) — source: OWASP SQL Injection Prevention Cheat Sheet; CWE-89.
7. **Are dynamic `ORDER BY` / column names validated against an allowlist?** (§1.1) — source: OWASP SQL Injection Prevention Cheat Sheet.
8. **Is `ProcessBuilder` with argument arrays used instead of `Runtime.exec(String)`?** (§1.3, §15.2, §16.1) — source: CERT IDS07-J; CWE-78.
9. **Is LDAP input passed through `LdapQueryBuilder` or equivalent, never raw filter-string concatenation?** (§1.2) — source: OWASP LDAP Injection Prevention Cheat Sheet.
10. **Is SpEL/OGNL/EL evaluation of user input forbidden?** (§1.4, §16.1) — source: CWE-94.

### C. XSS / CSRF / SSRF
11. **Is every template variable escaped by default (`th:text`, not `th:utext`)?** (§2.1, §15.3) — source: Thymeleaf reference.
12. **For REST HTML responses, is the correct `OWASP Java Encoder` method used per output context (HTML / attribute / JS / CSS / URI)?** (§2.1) — source: OWASP Java Encoder project; OWASP XSS Prevention Cheat Sheet.
13. **Is CSP strict (`default-src 'none'` + per-directive allowlists, nonce/hash for scripts)?** (§2.2, §16.2) — source: OWASP CSP Cheat Sheet.
14. **Are the headers `HSTS`, `X-Content-Type-Options`, `frame-ancestors`/`X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy` configured?** (§2.2) — source: OWASP Secure Headers Project.
15. **Is CSRF protection enabled for session-based endpoints; disabled only where JWT in `Authorization` header is used?** (§3, §15.12, §16.3) — source: Spring Security reference 6.4; OWASP CSRF Prevention Cheat Sheet.
16. **Do outbound HTTP calls validate host + scheme against an allowlist and reject RFC 1918 / link-local / loopback / IPv6 ULA?** (§4.1, §4.2, §16.4) — source: OWASP SSRF Prevention Cheat Sheet.
17. **Is DNS re-resolved on connect to defeat rebinding?** (§4.3) — source: OWASP SSRF Prevention Cheat Sheet.

### D. AuthN & AuthZ
18. **Is `SecurityFilterChain` used (not legacy `WebSecurityConfigurerAdapter`)?** (§6.1, §16.6) — source: Spring Security reference 6.4.
19. **Is `.anyRequest().authenticated()` the last matcher, with every `.permitAll()` narrow and justified?** (§6.1, §15.11, §16.6) — source: Spring Security reference 6.4.
20. **Are `@PreAuthorize`/`@PostAuthorize` present on service-layer methods, not only controllers?** (§6.1) — source: Spring Security reference 6.4.
21. **Are object-level ownership checks enforced (`findByIdAndUserId`, not only role checks)?** (§6.1) — source: CWE-639; OWASP Access Control Cheat Sheet.
22. **Do OAuth flows use exact redirect URI matching, PKCE, and reject Implicit / ROPC?** (§6.3) — source: RFC 9700.
23. **Are sender-constrained tokens (DPoP / mTLS) used for high-value APIs?** (§6.3) — source: RFC 9449 DPoP; RFC 8705 mTLS.

### E. Session & token handling
24. **Is `sessionFixation().migrateSession()` enabled?** (§6.4) — source: Spring Security reference 6.4.
25. **Is session ID regenerated on privilege change (login, MFA)?** (§6.4) — source: OWASP Session Management Cheat Sheet.
26. **Is the JWT signature verified every time (`parseSignedClaims` / `parseClaimsJws`, never `parseClaimsJwt`)?** (§6.2, §15.10, §16.6) — source: JJWT reference; Gotcha G-05.
27. **Is `alg` server-pinned and `alg: none` rejected?** (§6.2, §15.10) — source: RFC 8725 §3.1.
28. **Are `iss`, `aud`, `exp`, `nbf` claims validated?** (§6.2) — source: RFC 7519 §4.
29. **Are JWT cookies `HttpOnly` + `Secure` + `SameSite=Strict`?** (§6.2) — source: OWASP Session Management Cheat Sheet.

### F. Cryptography
30. **Is password storage Argon2id with ≥19 MiB / t=2 / p=1 (or one of the stronger OWASP 2024 profiles)?** (§7.1, §15.7, §16.7) — source: OWASP Password Storage Cheat Sheet.
31. **Are MD5 / SHA-1 / bare SHA-256 absent from password/credential paths?** (§7.1) — source: CWE-328.
32. **Is `SecureRandom` used for tokens, session IDs, IVs, nonces?** (§7.2, §15.5) — source: CERT MSC02-J.
33. **Is AES used only in AEAD modes (`AES/GCM/NoPadding`, 12-byte unique IV), never ECB or bare `"AES"`?** (§7.3, §15.6) — source: NIST SP 800-38D.
34. **Is TLS 1.2 minimum, 1.3 supported, and TLS 1.0/1.1 disabled?** (§7.4) — source: NIST SP 800-52 Rev. 2; RFC 8996.
35. **Is certificate validation left to the platform trust store (no `TrustAllCerts`)?** (§7.4, §15.4) — source: CWE-295.

### G. Secrets lifecycle
36. **Does CI / pre-commit run TruffleHog or Gitleaks with full-history scan?** (§8.1) — source: trufflesecurity.com; gitleaks.io.
37. **Are secrets injected at runtime (Vault, AWS/Azure/GCP Secret Manager, K8s Secrets via `configtree:`), not committed to YAML?** (§8.2, §15.13) — source: Spring Boot reference "ConfigTree".
38. **Are test resources, Dockerfile `ENV`, CI literals free of real credentials?** (§8.3) — source: OWASP Secrets Management Cheat Sheet.

### H. Deserialization
39. **Is every `ObjectInputStream` on untrusted data protected by a JEP 290 `ObjectInputFilter`?** (§9.1, §15.8) — source: Oracle Serialization Filtering; JEP 290; CERT SER12-J.
40. **For Java 17+: are context-specific filter factories (JEP 415) used where library boundaries demand it?** (§9.1) — source: JEP 415.
41. **Is Jackson `enableDefaultTyping()` absent and polymorphic types handled via `@JsonTypeInfo(use = Id.NAME)` + `@JsonSubTypes`?** (§9.2, §15.9) — source: FasterXML Jackson wiki.
42. **When `activateDefaultTyping` is unavoidable, is a restrictive `PolymorphicTypeValidator` configured AND `setPolymorphicTypeValidator` called on the builder?** (§9.2) — source: FasterXML Jackson wiki.
43. **Is `FAIL_ON_UNKNOWN_PROPERTIES` left enabled?** (§9.2) — source: Jackson docs.
44. **Are gRPC message size limits set (`maxInboundMessageSize`)?** (§9.3) — source: gRPC-Java reference.

### I. Dependency & supply chain
45. **Is OWASP Dependency-Check / Snyk / Dependency-Track wired into CI with a fail threshold on high/critical CVEs?** (§10.1) — source: OWASP Dependency-Check.
46. **Is Dependabot (or equivalent) open-PR-ing CVE upgrades?** (§10.1) — source: GitHub Dependabot docs.
47. **Is an SBOM (CycloneDX or SPDX) generated per release?** (§10.2) — source: OWASP CycloneDX.
48. **Is the Oracle CPU cadence (third Tuesday Jan/Apr/Jul/Oct) tracked, and is the latest patch applied?** (§10.5) — source: Oracle Critical Patch Updates.
49. **Are historical supply-chain incidents (xz-utils / Log4Shell / Spring4Shell) represented in the threat model for long-tail trust risks?** (§10.5) — source: CVE records; Andres Freund disclosure post.

### J. Operational hardening (Actuator, headers)
50. **Is `management.server.port` separated and bound to a private address?** (§11.2) — source: Spring Boot Actuator reference.
51. **Are `/env`, `/heapdump`, `/threaddump`, `/loggers`, `/mappings`, `/beans`, `/configprops`, `/shutdown` unavailable without authentication?** (§11.1) — source: Spring Boot Actuator reference.
52. **Is the deployed Spring Boot version patched against CVE-2025-22235?** (§11.3) — source: Spring advisory CVE-2025-22235.
53. **Are `/h2-console`, `/swagger-ui`, `/graphiql` gated in production profile?** (§11.4) — source: Spring Boot reference.

### K. Logging, alerting, and exceptional conditions
54. **Are authentication, authorization, session, and admin events logged with actor + resource + action?** (§13.1) — source: OWASP Logging Cheat Sheet; OWASP ASVS v5.0.0 §7.
55. **Are passwords, tokens, JWTs, CSRF tokens, and unneeded PII absent from logs?** (§13.2) — source: CWE-532; PCI DSS Req 3.
56. **Are logs forwarded to an immutable sink (SIEM, append-only cloud logging)?** (§13.3) — source: NIST SP 800-92.
57. **Are alerts wired for failed-login rate, 403 burst, Actuator access, post-deploy 5xx spikes?** (§13.4) — source: OWASP Top 10 2025 RC1 `A09:2025`.
58. **Are `catch` blocks never silently empty; error paths always either re-throw, wrap, or log?** (§12.1) — source: CERT ERR00-J.
59. **Are production error responses free of stack traces, SQL, file paths, framework versions?** (§12.2) — source: CWE-209.
60. **Do authorization / crypto paths fail closed (deny on exception)?** (§12.3) — source: OWASP ASVS v5.0.0.

---

## 19. Sources

### Normative / specification
- [OWASP Top 10:2021 (current released edition)](https://owasp.org/Top10/) — `A01:2021` Broken Access Control through `A10:2021` SSRF; released 2021 and still authoritative for compliance as of April 2026
- [OWASP Top 10:2025 RC1 (draft, not finalised)](https://owasp.org/Top10/2025/) — Release Candidate 1 dated November 6, 2025, Global AppSec. Cites `A03:2025` Software Supply Chain Failures (new), `A10:2025` Mishandling of Exceptional Conditions (new)
- [OWASP ASVS v5.0.0 (May 2025)](https://owasp.org/www-project-application-security-verification-standard/) — 17 chapters, ~350 requirements; stable
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/) — primary for: Input Validation, SQL Injection Prevention, OS Command Injection Defense, LDAP Injection Prevention, XSS Prevention, Content Security Policy, CSRF Prevention, SSRF Prevention, Authorization, Access Control, Authentication, Session Management, Password Storage, Cryptographic Storage, Transport Layer Security, File Upload, Deserialization, Error Handling, Logging, Secrets Management, Secure Headers
- [CWE Top 25 2025 (Dec 15, 2025, CISA + MITRE)](https://cwe.mitre.org/top25/) — supersedes prior Top 25 editions
- [NIST SP 800-52 Rev. 2 "Guidelines for TLS" (Aug 2019)](https://csrc.nist.gov/publications/detail/sp/800-52/rev-2/final) — TLS 1.2 minimum, TLS 1.3 required by Jan 1, 2024
- [NIST SP 800-38D "Recommendation for Block Cipher Modes of Operation: Galois/Counter Mode"](https://csrc.nist.gov/publications/detail/sp/800-38d/final)
- [NIST SP 800-175B Rev. 1 "Guideline for Using Cryptographic Standards"](https://csrc.nist.gov/publications/detail/sp/800-175b/rev-1/final)
- [NIST SP 800-92 "Guide to Computer Security Log Management"](https://csrc.nist.gov/publications/detail/sp/800-92/final)
- [RFC 7519 JSON Web Token (JWT)](https://datatracker.ietf.org/doc/html/rfc7519)
- [RFC 8725 JSON Web Token Best Current Practices (Feb 2020)](https://datatracker.ietf.org/doc/html/rfc8725)
- [RFC 9700 OAuth 2.0 Security Best Current Practice (Jan 2025)](https://datatracker.ietf.org/doc/html/rfc9700)
- [RFC 8446 Transport Layer Security 1.3 (Aug 2018)](https://datatracker.ietf.org/doc/html/rfc8446)
- [RFC 8996 Deprecating TLS 1.0 and TLS 1.1 (Mar 2021)](https://datatracker.ietf.org/doc/html/rfc8996)
- [RFC 7636 Proof Key for Code Exchange (PKCE)](https://datatracker.ietf.org/doc/html/rfc7636)
- [RFC 9449 OAuth 2.0 DPoP](https://datatracker.ietf.org/doc/html/rfc9449)
- [RFC 8705 OAuth 2.0 Mutual TLS](https://datatracker.ietf.org/doc/html/rfc8705)
- [RFC 9106 Argon2](https://datatracker.ietf.org/doc/html/rfc9106)
- [RFC 7539 ChaCha20-Poly1305](https://datatracker.ietf.org/doc/html/rfc7539)
- [RFC 6797 HSTS](https://datatracker.ietf.org/doc/html/rfc6797)
- [JEP 290: Filter Incoming Serialization Data (JDK 9)](https://openjdk.org/jeps/290)
- [JEP 415: Context-Specific Deserialization Filters (JDK 17)](https://openjdk.org/jeps/415)
- [SEI CERT Oracle Coding Standard for Java](https://wiki.sei.cmu.edu/confluence/display/java/) — referenced rules: IDS07-J (command injection), FIO16-J (path canonicalization), MSC02-J (strong random), SEC07-J (TrustManager), SER12-J (deserialization), ERR00-J (exception suppression)
- [Oracle Java "Serialization Filtering" guide (JDK 21)](https://docs.oracle.com/en/java/javase/21/core/java-serialization-filters.html)
- [Spring Security reference 6.4](https://docs.spring.io/spring-security/reference/6.4/)
- [Spring Boot reference](https://docs.spring.io/spring-boot/reference/)
- [Jakarta Validation 3.1 specification](https://jakarta.ee/specifications/bean-validation/3.1/)

### Foundational / authoritative
- **David Coffin, *Expert Oracle and Java Security***, Apress 2011, ISBN 978-1-4302-3831-6 — Java-platform security architecture, SecurityManager, access control, code signing, JCE/JAAS integration. Still the most complete textbook treatment of the classic Java sandbox and its cryptographic plumbing.
- FasterXML Jackson wiki, "Polymorphic Deserialization CVE Criteria" — maintainer-authored primary source on the gadget-class problem
- Andres Freund, oss-security list (Mar 29, 2024) — original disclosure of CVE-2024-3094 (xz-utils backdoor)
- Apache Log4j Security Vulnerabilities page — canonical CVE-2021-44228 reference

### Tooling
- [OWASP Dependency-Check](https://owasp.org/www-project-dependency-check/) — Maven/Gradle plugin for CVE scanning
- [OWASP Dependency-Track](https://dependencytrack.org/) — SBOM-consuming vulnerability tracking platform
- [Snyk](https://snyk.io/) — commercial CVE scanner with reachability analysis
- [Semgrep](https://semgrep.dev/) — SAST with a curated rule set (including `java.lang.security.*`) and Supply Chain reachability
- [CodeQL](https://codeql.github.com/) — GitHub's semantic code-analysis engine with Java security queries
- [Find Security Bugs (SpotBugs plugin)](https://find-sec-bugs.github.io/) — taint analysis for common Java security patterns
- [TruffleHog](https://github.com/trufflesecurity/trufflehog) — secret scanner with 700+ verifier plugins
- [Gitleaks](https://github.com/gitleaks/gitleaks) — secret scanner with TOML rule sets
- [SpotBugs](https://spotbugs.github.io/) — static analyser; concurrency + security detectors
- [OWASP Java Encoder](https://owasp.org/www-project-java-encoder/) — contextual output encoding library
- [OWASP CycloneDX](https://cyclonedx.org/) — SBOM specification + Maven plugin

### Checklists and curated reviews
- [OWASP Cheat Sheet Series index](https://cheatsheetseries.owasp.org/)
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)
- [Spring Security reference 6.4 — "Testing"](https://docs.spring.io/spring-security/reference/6.4/servlet/test/) — how to assert security configuration in unit tests

### Industry case studies and explainers
- [CVE-2024-3094 — xz-utils backdoor](https://nvd.nist.gov/vuln/detail/CVE-2024-3094) — disclosed Mar 29, 2024; CVSS 10.0; Andres Freund on oss-security
- [CVE-2022-22965 — Spring4Shell](https://nvd.nist.gov/vuln/detail/CVE-2022-22965) — Spring Framework `WebDataBinder` RCE on JDK 9+; Mar 31, 2022; CVSS 9.8
- [CVE-2021-44228 — Log4Shell](https://nvd.nist.gov/vuln/detail/CVE-2021-44228) — Apache Log4j 2.x JNDI lookup RCE; Dec 2021; CVSS 10.0
- [CVE-2025-22235 — Spring Boot `EndpointRequest.to()` matcher](https://spring.io/security/cve-2025-22235) — incorrect Security matcher when endpoint not exposed
- [Oracle Critical Patch Update — January 2026](https://www.oracle.com/security-alerts/cpujan2026.html) — 11 Java SE patches, all remote-no-auth, incl. CVE-2026-21945 (mTLS client cert)
- [Brian Vermeer, "Serialization and deserialization in Java: explaining the Java deserialize vulnerability"](https://snyk.io/blog/java-json-deserialization-problems-jackson-objectmapper/) — Snyk research, primary-sourced

---

## 20. Gotchas — Common Wrong Assumptions

G-01. **"OWASP Top 10:2025 is the current standard"** — false. As of April 2026, `Top 10:2025` is **Release Candidate 1** (Nov 6, 2025, Global AppSec). `Top 10:2021` is still the officially released edition and the compliance reference. Cite both — 2021 for compliance, 2025 RC1 for the draft direction of travel. See §17, §19.

G-02. **"Argon2id 15 MiB / 2 iters is the OWASP baseline"** — outdated. Current OWASP Password Storage Cheat Sheet minimum is **m=19456 KiB (19 MiB), t=2, p=1**. Stronger profiles up to m=47104 KiB (46 MiB), t=1 are preferred. See §7.1.

G-03. **"TLS 1.2 is deprecated"** — false as of April 2026. TLS 1.2 remains permitted; TLS 1.0 and 1.1 are deprecated (RFC 8996, 2021). NIST SP 800-52 Rev. 2 required **support** for TLS 1.3 by Jan 1, 2024 but allows TLS 1.2 as minimum. See §7.4.

G-04. **"`alg: none` is a theoretical issue"** — false. Libraries that honor the JWT header's `alg` claim for verification accept `alg: none` tokens and skip signature check. Always pin algorithm server-side; treat the `alg` header as attacker-controlled. See §6.2, §15.10.

G-05. **"JJWT `parseClaimsJwt(token)` validates the signature"** — false. `parseClaimsJwt` (no `s`) is for UNSIGNED tokens — it deliberately accepts any input. Use `parseClaimsJws` (with `s`, for JSON Web Signature) or JJWT 0.12+ `parseSignedClaims`. See §6.2, §15.10.

G-06. **"`Cipher.getInstance("AES")` uses a sensible default"** — false. In OpenJDK it defaults to `AES/ECB/PKCS5Padding`. ECB is deterministic — identical plaintexts produce identical ciphertexts; patterns leak. Always specify `AES/GCM/NoPadding` (or `AES/CBC/PKCS5Padding` with HMAC, though AEAD is preferred). See §7.3, §15.6.

G-07. **"`java.util.Random` is fine if the seed is unknown"** — false. `Random` uses a 48-bit LCG. An attacker who sees any two consecutive `nextInt()` outputs can recover the internal state and predict all future values. Use `SecureRandom` for anything security-relevant. See §7.2, §15.5.

G-08. **"`HttpOnly` cookies are encrypted"** — false. `HttpOnly` only blocks `document.cookie` access from JavaScript — the cookie itself traverses the network and is stored in plaintext on the browser. Pair with `Secure` (HTTPS-only transmission) and `SameSite`. For payload confidentiality, encrypt the cookie value server-side. See §6.2.

G-09. **"Jackson default typing is safe if you use the right annotation"** — false. `ObjectMapper.enableDefaultTyping()` is the classic RCE primitive, and even the replacement `activateDefaultTyping(ptv, ...)` has a trap: if you rely on `@JsonTypeInfo(use = Id.CLASS)` on a field, the validator from `activateDefaultTyping` is NOT invoked — you must also call `setPolymorphicTypeValidator()` on the builder. Prefer `Id.NAME` + `@JsonSubTypes` and skip default typing entirely. See §9.2, §15.9.

G-10. **"`/actuator/env` with masked credentials is fine to expose"** — false. Masking logic has historically been bypassable by property-name casing tricks and re-exposed via nested key lookups. Treat `/env` as a heap-dump in disguise: authenticate, or don't expose. Wiz 2025 scan found 1-in-4 Actuator deployments publicly exposed. See §11.1.

G-11. **"`Encode.forHtml(s)` is safe in any HTML context"** — false. HTML body, HTML attribute, JavaScript string literal, CSS string, and URI component each require different encodings. `Encode.forHtml` is NOT safe for `<script>var x = "[HERE]"</script>` — use `Encode.forJavaScript`. See §2.1 matrix.

G-12. **"CSP `unsafe-inline` is OK if you trust your own HTML"** — false. Reflected / stored XSS, extension injection, and third-party scripts all bypass it. Strict CSP uses `nonce-{random}` + `strict-dynamic` with no `unsafe-inline`, no `unsafe-eval`. See §2.2.

G-13. **"CSRF must always be on"** — false. CSRF exists because browsers auto-attach cookies cross-origin. For JWT in the `Authorization: Bearer` header (which browsers do NOT auto-attach cross-origin), CSRF disable for that matcher is correct. When the JWT is in a cookie, CSRF is required. See §3.2, §15.12.

G-14. **"ECB is only bad for long plaintexts"** — false. ECB is deterministic for any length: two equal plaintext blocks produce two equal ciphertext blocks. Even short tokens can be distinguished-set attacked. Use AEAD. See §7.3, §15.6.

G-15. **"If I don't use the vulnerable dependency directly, its CVE doesn't affect me"** — false. Transitive dependencies run in the same JVM. A Log4Shell-style JNDI in a logging library pulled by a starter is exploitable even if your code never imports `log4j-core`. Scan transitively (`mvn dependency:tree`) and fail CI on high/critical CVEs. See §10.4.

G-16. **"`ObjectInputFilter` via `-Djdk.serialFilter` is enough"** — partly. JVM-wide filters are coarse. JEP 415 (JDK 17+) introduced filter factories for context-specific filters at library boundaries — use them when different call sites need different allowlists. See §9.1.

G-17. **"Spring Security `.permitAll()` only opens the matched path"** — correct in isolation, but `.permitAll()` placed before a more-specific `.authenticated()` matcher can still leak state-changing endpoints because Spring Security matches first-match-wins. Order matters; narrow matchers first, broad matchers last. See §6.1, §15.11.

G-18. **"Session fixation isn't a risk if I rotate on login"** — partly. Also rotate the session ID on MFA step-up, role change, and token refresh. Stale session IDs issued during a partially authenticated phase can be hijacked. See §6.4.

G-19. **"Logs with stack traces are fine — they're internal"** — false when errors are surfaced to clients. CWE-209 (Information Exposure Through Error Message) is exploited by scanners harvesting framework versions, file paths, and query shapes. Set `server.error.include-stacktrace=never` in production. See §12.2.

G-20. **"If it compiles and tests pass, the security review is done"** — false. CVE scans, SAST (CodeQL / Semgrep / Find Security Bugs), and secret scans must also pass. Most runtime vulnerabilities (injection, deserialization gadgets, algorithm confusion) are not detected by functional tests. See §10, §16.
