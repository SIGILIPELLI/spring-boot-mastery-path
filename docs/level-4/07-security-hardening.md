---
description: "Harden Spring Boot services: secret management, security headers, CORS, rate limiting, safe error output, dependency scanning, and least-privilege access."
---

# Security Hardening

Level 2 made the API authenticate and authorize requests. Hardening is everything around
that: keeping secrets out of the wrong places, shrinking what an attacker can reach, and
making the vulnerabilities you will inevitably ship less damaging. Work through it as a
checklist; each item closes a real class of incidents.

## 1. Secrets

- Never commit secrets. `application.yml` references them: `password: ${DB_PASSWORD}`.
- In production, inject them from a secret manager (Vault, AWS Secrets Manager, Azure Key
  Vault, GCP Secret Manager) or platform secrets. Spring Cloud Vault and the cloud providers'
  Spring integrations can load them as property sources; Kubernetes can mount secrets as
  files, which Boot reads with `spring.config.import: configtree:/run/secrets/`.
- Prefer **short-lived, automatically rotated credentials** (database credentials issued by
  Vault, cloud workload identity) over long-lived passwords.
- Scan the repository history with a secret scanner in CI (gitleaks, GitHub secret scanning).
- Check that Actuator's `env`/`configprops` show sanitized values (Boot masks values by
  default; `management.endpoint.env.show-values` controls it) — and do not expose them.

## 2. Transport and headers

- TLS everywhere, usually terminated at the load balancer or ingress. Behind it, set
  `server.forward-headers-strategy: framework` (or `native`) so Spring sees the original
  scheme and host — otherwise `Location` headers and redirects say `http://`. Only trust
  forwarded headers from your own proxies.
- Spring Security already sends `X-Content-Type-Options: nosniff`,
  `Cache-Control: no-store` for authenticated responses, `X-Frame-Options: DENY`, and HSTS
  over HTTPS. For HTML-serving apps, add a Content Security Policy:

```java
http.headers(h -> h
        .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'; frame-ancestors 'none'"))
        .referrerPolicy(r -> r.policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.NO_REFERRER)));
```

## 3. CORS

Browsers block cross-origin calls unless the server allows them. Allow exactly the origins
you need:

```java
@Bean
CorsConfigurationSource corsConfigurationSource() {
    var config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.example.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PATCH", "DELETE"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setMaxAge(Duration.ofHours(1));
    var source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}
// and in the filter chain: http.cors(Customizer.withDefaults())
```

`allowedOrigins("*")` combined with credentials is rejected by Spring for good reason. CORS
is not an authorization mechanism — non-browser clients ignore it.

## 4. Input and output

- Validate all input (Level 1). Cap sizes: `spring.servlet.multipart.max-file-size`,
  `server.tomcat.max-swallow-size`, page sizes, string lengths.
- Use parameter binding for every query (Spring Data and `JdbcClient` do); never build SQL
  or JPQL with string concatenation from input.
- Keep error responses generic (Level 1, lesson 08); stack traces and SQL fragments in
  responses help attackers.
- Beware **mass assignment**: binding request JSON directly into entities lets clients set
  fields like `role` or `ownerId`. Request DTOs prevent it.
- Check **object-level authorization** on every access by id — "is this order the caller's?"
  Broken object-level authorization is consistently at the top of the OWASP API Security
  Top 10.

## 5. Rate limiting and abuse

Limit requests per client at the edge (gateway, ingress, CDN) and, for sensitive endpoints
like login or password reset, in the app as well — for example with Bucket4j keyed by user
id or IP. Return 429 with `Retry-After`.

## 6. Dependencies

Most vulnerabilities you ship come from libraries. Stay on a supported Boot line (open-source
support for each minor version is limited; check the Spring support timeline), upgrade Boot
patch releases promptly, and scan in CI (OWASP Dependency-Check, GitHub Dependabot, Snyk,
or `trivy` on the container image). Generate an SBOM (CycloneDX's Maven plugin; Boot can
also expose one through Actuator's `sbom` endpoint) so you can answer "are we affected?"
within minutes of an advisory.

## 7. Least privilege

- The database user the app connects with should not own the schema: run Flyway with a
  migration user and the app with a user limited to DML on its tables.
- Containers run as non-root with a read-only root filesystem where possible.
- Service-to-service calls use their own credentials (OAuth2 client credentials, mTLS), each
  scoped to what the caller needs.
- Actuator on a separate, non-public port.

## Worked example: a hardening review of the library API

Applying the checklist to the Level 2 project finds:

| Finding | Fix |
|---------|-----|
| HMAC JWT secret in `application.yml` | Move to `issuer-uri` with the identity provider's keys; no shared secret in prod |
| `GET /api/books` unbounded | Page it with a max size |
| No CORS config | Allow only the frontend's origin |
| App connects as schema owner | Separate Flyway user and app user |
| Actuator on the main port | `management.server.port: 8081`, not routed publicly |
| No dependency scanning | Dependabot + `trivy` in CI |

## How It Actually Works

**CORS.** For "non-simple" requests (e.g. JSON with an `Authorization` header), the browser
first sends a preflight `OPTIONS` request with `Origin` and `Access-Control-Request-Method`.
Spring's `CorsFilter` (added to the security chain by `http.cors()`, before authentication so
preflights are not rejected with 401) matches the path's `CorsConfiguration`, and either
answers with `Access-Control-Allow-*` headers or rejects it. The browser only sends the real
request if the preflight allowed it.

**Forwarded headers.** With `forward-headers-strategy: framework`, Spring's
`ForwardedHeaderFilter` rewrites the request's scheme, host, port, and remote address from
`Forwarded`/`X-Forwarded-*` headers and removes them. That is why only your proxy should be
able to set them: a client-supplied `X-Forwarded-For` would otherwise spoof its IP for rate
limiting or audit logs.

## Common mistakes

- **Secrets in environment files committed "just for dev"** that are later reused.
- **CORS `*` on authenticated APIs.**
- **Authorization checks only at the URL level**, not per object.
- **Trusting forwarded headers from anywhere.**
- **Running months behind on Boot patch releases.**

## Exercise

1. Run your hardening review on the capstone and fill in a findings table like the one above.
2. Configure CORS for one origin and verify a preflight with
   `curl -i -X OPTIONS -H 'Origin: https://app.example.com' -H 'Access-Control-Request-Method: POST' localhost:8080/api/books`,
   then with a different origin.
3. Add a Bucket4j limit of 5 requests per minute to a sensitive endpoint, returning a 429
   problem detail with `Retry-After`.
4. Add OWASP Dependency-Check or `trivy` to your build and triage the first report.
