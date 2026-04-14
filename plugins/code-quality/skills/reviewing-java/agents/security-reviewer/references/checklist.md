# Java Security Review Checklist

Concise reference for finding security vulnerabilities during code review. Focus on OWASP Top 10, input validation, authentication, cryptography, and dependency security.

---

## 1. Injection

### SQL Injection
- **Never** concatenate user input into SQL queries. Use `PreparedStatement` with parameterized queries or JPA named parameters exclusively
- **Ban** `Statement.execute(String)` and `Statement.executeQuery(String)` with dynamic SQL
- **Native queries in JPA**: `@Query(nativeQuery = true, value = "SELECT * FROM users WHERE name = :name")` — safe. `"... WHERE name = '" + name + "'"` — vulnerable
- **Criteria API / QueryDSL**: generally safe. But dynamic `ORDER BY` clauses often bypass parameterization — validate against an allowlist of column names

### LDAP Injection
- Never pass unsanitized input to `DirContext.search()` or LDAP filter strings
- Use Spring LDAP's `LdapQueryBuilder` which auto-escapes filter values

### OS Command Injection
- **Never** use `Runtime.exec(String)` with shell string interpolation
- Use `ProcessBuilder` with argument arrays: `new ProcessBuilder("cmd", arg1, arg2)` — arguments are never interpreted as shell syntax
- If shell execution is unavoidable, validate input against a strict allowlist pattern

### Expression Language Injection
- Never evaluate user input as SpEL, OGNL, or EL expressions
- Spring `@Value("${...}")` is safe (property resolution). `@Value("#{...}")` evaluates SpEL — never use with user input

---

## 2. Cross-Site Scripting (XSS)

- **Templating engines**: use auto-escaping by default. Thymeleaf `th:text` (safe) vs `th:utext` (unsafe — raw HTML). Flag any `th:utext` with user-controlled data
- **REST APIs returning HTML**: encode output with OWASP Java Encoder: `Encode.forHtml()`, `Encode.forJavaScript()`, `Encode.forCssString()`
- **Security headers** (configure in `SecurityFilterChain`):
  - `Content-Security-Policy: default-src 'self'; frame-ancestors 'none'; script-src 'self'`
  - `X-Content-Type-Options: nosniff`
  - `X-Frame-Options: DENY`
- **JSON APIs**: generally safe from reflected XSS, but stored XSS still applies if HTML clients render the data without escaping

---

## 3. CSRF

- Spring Security enables CSRF protection by default for session-based auth
- **SPAs with JWT**: CSRF tokens may be unnecessary if using `Authorization: Bearer` header (not cookies). But if JWT is in a cookie, CSRF protection is required
- Use `CookieCsrfTokenRepository.withHttpOnlyFalse()` for SPAs that need to read the token
- Verify CSRF tokens on **all state-changing endpoints** (POST, PUT, DELETE, PATCH)
- `GET` endpoints must be **safe** (no side effects) — otherwise CSRF attacks via `<img>` tags

---

## 4. Server-Side Request Forgery (SSRF)

- **Allowlist** destination hosts and schemes for any server-side HTTP call
- **Reject private/internal IP ranges**: `10.*`, `172.16-31.*`, `192.168.*`, `169.254.*`, `127.*`, `0.0.0.0`, `::1`, `fd00::/8`
- Resolve DNS **before** checking — attackers use DNS rebinding to bypass IP checks
- Never let user input control full URLs for server-side HTTP calls (`RestTemplate`, `WebClient`, `HttpClient`)
- If user provides a URL (webhooks, callbacks): parse and validate scheme (https only), host (against allowlist), and port

---

## 5. Input Validation

### Where to Validate
- **Controller layer**: structural validation with JSR-380/Jakarta Bean Validation (`@NotNull`, `@NotBlank`, `@Size`, `@Pattern`, `@Email`) + `@Valid` on request bodies/parameters
- **Service layer**: business rule validation (e.g., "order total cannot exceed credit limit")
- **Never** trust client-side validation alone

### Principles
- **Allowlist over denylist**: define what IS permitted (alphanumeric pattern for 90%+ of fields). Reject everything else
- **Path traversal**: never pass user input to filesystem APIs (`new File(userInput)`). If unavoidable:
  ```java
  Path resolved = basePath.resolve(userInput).normalize();
  if (!resolved.startsWith(basePath)) {
      throw new SecurityException("Path traversal attempt");
  }
  ```
  Better: map file references to database IDs, never expose filesystem paths
- **Header injection**: reject HTTP header values containing `\r\n` (CRLF injection)
- **File uploads**: validate content type (magic bytes, not just extension), enforce size limits, store outside webroot with random names

---

## 6. Authentication & Authorization

### Spring Security Configuration
- **Catch-all rule**: `.anyRequest().authenticated()` as the last matcher in `SecurityFilterChain`. More specific matchers must precede general ones
- **Audit open endpoints**: flag any `.permitAll()` or `@PermitAll` without documented justification
- **Method-level security**: use `@PreAuthorize` on service methods, not just controllers
- **Object-level authorization**: enforce ownership checks (`repository.findByIdAndUserId()`), not just role checks. "User A cannot access User B's orders" is not covered by `@RolesAllowed`

### JWT Pitfalls
- **Always verify signature** server-side. Never use `parseClaimsJwt()` (unsigned) — use `parseClaimsJws()` (signed)
- **Enforce expected algorithm**: reject `alg: none`. Pin expected algorithm (e.g., RS256) in validation config
- **Validate claims**: `iss` (issuer), `aud` (audience), `exp` (expiration), `nbf` (not before)
- **Token storage**: `HttpOnly` + `Secure` + `SameSite=Strict` cookies for web. `Authorization: Bearer` header for API clients
- **Token revocation**: maintain a blocklist for logout/compromised tokens, or use short-lived tokens + refresh tokens

### Session Management
- Call `sessionFixation().migrateSession()` to prevent session fixation attacks
- Set session timeout appropriately (e.g., 30 minutes for web apps)
- Configure CORS at the Security filter level, not just Spring MVC level

---

## 7. Cryptography

### Passwords
- **Use**: Argon2id (15 MiB memory, 2 iterations), bcrypt (cost >= 10), or scrypt
- **Ban**: MD5, SHA-1, SHA-256 for password storage — these are fast hash functions, not password hashing algorithms
- Use Spring Security's `DelegatingPasswordEncoder` for forward-compatible hashing with automatic migration

### Randomness
- **`java.security.SecureRandom`**: for tokens, session IDs, nonces, CSRF tokens, password reset links
- **`java.util.Random`**: NEVER for security contexts — predictable seed, predictable output
- `UUID.randomUUID()` uses `SecureRandom` internally — safe for tokens

### Encryption
- **Use**: AES-256-GCM (authenticated encryption). Use `Cipher.getInstance("AES/GCM/NoPadding")`
- **Ban**: DES, 3DES, AES-ECB (deterministic — identical plaintexts produce identical ciphertexts), static IVs
- Never implement your own cryptographic algorithms

### TLS
- Enforce TLS 1.2+ in server configuration
- Disable SSLv3, TLS 1.0, TLS 1.1
- Use the platform trust store for certificate validation — don't disable certificate verification (`TrustAllCerts` is a P0 finding)

---

## 8. Secrets Management

### Detection
- Run **TruffleHog** or **Gitleaks** as a pre-commit hook and in CI. Scan full git history, not just HEAD
- Ban hardcoded strings matching patterns: AWS keys (`AKIA*`), private keys (`-----BEGIN`), `password =` with literal values, JDBC URLs with credentials

### Storage
- **Never** commit: `.env` files, `application-prod.yml` with real credentials, keystores, private keys
- Inject secrets at runtime via HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, or Kubernetes Secrets (mounted as files via `spring.config.import=configtree:`)
- Spring Cloud Config: use encrypted backend, not plaintext properties
- In logs/exceptions: mask or redact any potentially sensitive values

### What to Flag in Review
- `application.yml` / `application.properties` with values that look like real credentials
- Connection strings with embedded username/password
- API keys or tokens in source code (even in test code — use test-specific secrets or mocks)
- `src/test/resources` with production-like credentials

---

## 9. Serialization & Deserialization

### Java Native Serialization
- **Ban `ObjectInputStream.readObject()` on untrusted input** — the #1 source of Java RCE vulnerabilities
- If unavoidable: use `ObjectInputFilter` (JEP 290, Java 9+) with an explicit allowlist of permitted classes
- Set JVM-wide filter: `-Djdk.serialFilter=maxdepth=5;maxarray=100000;!*` (deny-all, then allow specific classes)

### Jackson
- **Never** call `ObjectMapper.enableDefaultTyping()` — enables polymorphic deserialization, the root cause of most Jackson CVEs
- Use `activateDefaultTyping()` with an explicit `PolymorphicTypeValidator` (Jackson 2.10+)
- Enable `DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES` (default in Spring Boot)
- Prefer `@JsonTypeInfo(use = Id.NAME)` (string discriminator) over `Id.CLASS` (class name in JSON — attacker controls which class is instantiated)
- Avoid deserializing to `Object`-typed fields. Always specify concrete target types in `readValue()`
- Keep Jackson updated — maintainers actively block dangerous gadget classes via `SubTypeValidator`

### Protobuf / Avro / gRPC
- Set message size limits (`maxInboundMessageSize` in gRPC) to prevent denial-of-service via huge payloads
- Validate deserialized values — Protobuf doesn't enforce constraints, only structure

---

## 10. Dependency Security

### Scanning
- Run **OWASP Dependency-Check** (`dependency-check-maven`/`dependency-check-gradle`) or **Snyk** in CI on every PR
- Fail the build on **critical** and **high** CVEs
- Enable **GitHub Dependabot** for automated upgrade PRs with CVE annotations
- Prefer tools with **reachability analysis** (Snyk, Semgrep Supply Chain) to prioritize CVEs in code paths you actually invoke

### Suppression Management
- Suppress false positives only with documented justification and expiration date
- Review suppressions quarterly
- Track suppressed CVEs in a file committed to the repo (`dependency-check-suppressions.xml`)

### Transitive Dependencies
- Run `mvn dependency:tree` / `gradle dependencies` regularly — know what you're shipping
- Transitive dependencies inherit their vulnerabilities. A CVE in `commons-text` pulled in by `spring-boot-starter` affects you even if you never import it directly

---

## 11. Spring Boot Actuator & Debug Endpoints

- **Production**: disable all actuator endpoints or restrict to management port behind VPN/auth
- **Never expose**: `/actuator/env` (shows config + potential secrets), `/actuator/heapdump` (heap contents), `/actuator/mappings` (all endpoints), `/actuator/beans`
- **Safe to expose**: `/actuator/health` (with `show-details=never`), `/actuator/prometheus`
- Remove or secure: `/h2-console`, `/swagger-ui`, `/graphiql` in production profiles
- Set `management.server.port` to a different port (e.g., 9090) and restrict network access

---

## 12. Quick Security Review Heuristic

When reviewing Java code for security, scan in this order:

1. **User input flows**: trace every request parameter, header, path variable, request body — where does it go? SQL? File system? HTTP call? Shell? Log?
2. **Authentication bypass**: any new endpoint missing auth? Any `.permitAll()` without justification?
3. **Authorization gaps**: role check present? Object-level ownership enforced?
4. **SQL / LDAP / command injection**: any string concatenation with external input near query/command execution?
5. **Deserialization**: any `readObject()`, `enableDefaultTyping()`, `@JsonTypeInfo(use = CLASS)`?
6. **Secrets in code**: any hardcoded passwords, API keys, tokens? In config files, test resources, comments?
7. **Cryptography**: password hashing with proper algorithm? `SecureRandom` for tokens? No `TrustAllCerts`?
8. **Dependency CVEs**: has the dependency scan passed? Any new dependencies added?
9. **Error responses**: do error messages leak internal details (stack traces, SQL, file paths)?
10. **Security headers**: CSP, HSTS, X-Content-Type-Options, X-Frame-Options configured?

---

Sources:
- [OWASP Top 10:2025](https://owasp.org/Top10/2025/)
- [OWASP Java Security Code Review Patterns](https://www.augmentcode.com/guides/java-security-code-review-owasp-patterns-for-enterprise)
- [OWASP Input Validation Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html)
- [OWASP Dependency-Check](https://owasp.org/www-project-dependency-check/)
- [Detecting Authorization Flaws in Java Spring — NetSPI](https://www.netspi.com/blog/technical-blog/secure-code-review/authorization-flaws-java-spring-via-source-code-review/)
- [Jackson Polymorphic Deserialization CVE Criteria — FasterXML](https://github.com/FasterXML/jackson/wiki/Jackson-Polymorphic-Deserialization-CVE-Criteria)
- [Snyk: Java JSON Deserialization Problems](https://snyk.io/blog/java-json-deserialization-problems-jackson-objectmapper/)
- [Password Hashing Best Practices — Java Code Geeks](https://www.javacodegeeks.com/2025/05/best-practices-for-storing-and-validating-passwords-in-java-bcrypt-argon2-pbkdf2.html)
- [Secret Scanning Tools 2026 — GitGuardian](https://blog.gitguardian.com/secret-scanning-tools/)
- [TruffleHog Deep Dive — Jit](https://www.jit.io/resources/appsec-tools/trufflehog-a-deep-dive-on-secret-management-and-how-to-fix-exposed-secrets)
- [Spring Security JWT — DevGlan](https://www.devglan.com/spring-security/jwt-authentication-spring-security)
