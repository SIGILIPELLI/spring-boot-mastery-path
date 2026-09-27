---
description: "Protect a Spring Boot API with JWT bearer tokens using the OAuth2 resource server: issuer config, signature validation, scopes, and claim mapping."
---

# JWT & the OAuth2 Resource Server

Modern APIs rarely check passwords themselves. A user signs in at an **authorization
server** (identity provider), which issues an access token; the client sends that token
with each request; your API — the **resource server** — validates it and reads who the
caller is and what they may do. Spring Security's OAuth2 resource server support does the
validation part in a few lines.

## The roles

| Role | Example | Responsibility |
|------|---------|---------------|
| Resource owner | The user | Grants access |
| Client | A web or mobile app, another service | Obtains tokens, calls the API |
| Authorization server | Keycloak, Auth0, Okta, Entra ID, Cognito, Spring Authorization Server | Authenticates users, issues tokens |
| Resource server | **Your Spring Boot API** | Validates tokens, enforces access |

Your API never sees the user's password.

## What a JWT is

A JSON Web Token has three base64url parts: `header.payload.signature`.

```json
// header
{ "alg": "RS256", "kid": "2026-key-1" }
// payload (claims)
{
  "iss": "https://auth.example.com/realms/library",
  "sub": "8b6c…",
  "aud": "library-api",
  "exp": 1790000000,
  "iat": 1789996400,
  "scope": "books:read books:write"
}
```

The signature proves the token was issued by the holder of the private key and not
modified. The payload is **encoded, not encrypted** — anyone can read it, so never put
secrets in it.

## Configuration with a real identity provider

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security-oauth2-resource-server</artifactId>
</dependency>
```

(Boot 3: `spring-boot-starter-oauth2-resource-server`.)

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com/realms/library
          audiences: library-api
```

```java
@Bean
SecurityFilterChain api(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(HttpMethod.GET, "/api/books/**").permitAll()
            .requestMatchers(HttpMethod.POST, "/api/books/**").hasAuthority("SCOPE_books:write")
            .anyRequest().authenticated())
        .oauth2ResourceServer(rs -> rs.jwt(Customizer.withDefaults()))
        .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .csrf(csrf -> csrf.disable());
    return http.build();
}
```

With `issuer-uri` set, Boot creates a `JwtDecoder` that discovers the provider's JWK Set
URL from its OpenID configuration, downloads the public keys, and validates signature,
`exp`/`nbf`, `iss`, and (if configured) `aud`. Scopes in the `scope` or `scp` claim become
authorities prefixed with `SCOPE_`.

Running this against a real identity provider needs one — Keycloak in Docker is a common
local choice. The course did not run an external provider; the project uses the local
setup below instead.

## Local development with a shared secret

For development and tests you can validate HMAC-signed tokens with a symmetric key. There
is no Boot property for this, so you define the decoder bean yourself — this is exactly
what the Level 2 project does, and it ran on Boot 4.1:

```java
@Bean
JwtDecoder jwtDecoder(@Value("${app.jwt.secret}") String secret) {
    var key = new SecretKeySpec(secret.getBytes(StandardCharsets.UTF_8), "HmacSHA256");
    return NimbusJwtDecoder.withSecretKey(key).build();
}
```

```yaml
app:
  jwt:
    secret: change-me-change-me-change-me-32b   # dev only; HS256 needs >= 32 bytes
```

(While writing the project, a first attempt used a made-up `spring.security.oauth2.resourceserver.jwt.secret`
property. The app failed at startup with "required a bean of type
'org.springframework.security.oauth2.jwt.JwtDecoder' that could not be found" — Boot only
auto-creates a decoder from `issuer-uri`, `jwk-set-uri`, or `public-key-location`.)

Symmetric keys mean everyone who can verify can also *sign*. That is acceptable inside one
service's tests, and not acceptable across services. Production uses asymmetric keys from
the identity provider.

## Mapping claims to authorities

Identity providers put roles in different claims (Keycloak uses
`realm_access.roles`, others a `roles` or `groups` claim). Map them explicitly:

```java
@Bean
JwtAuthenticationConverter jwtAuthenticationConverter() {
    var scopes = new JwtGrantedAuthoritiesConverter();          // keeps SCOPE_* authorities
    var converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(jwt -> {
        Collection<GrantedAuthority> authorities = new ArrayList<>(scopes.convert(jwt));
        List<String> roles = jwt.getClaimAsStringList("roles");
        if (roles != null) {
            roles.forEach(r -> authorities.add(new SimpleGrantedAuthority("ROLE_" + r)));
        }
        return authorities;
    });
    return converter;
}
```

In a controller, read the caller with `@AuthenticationPrincipal Jwt jwt` and use
`jwt.getSubject()` as the stable user id. Do not key user data on email addresses; they
change.

## Worked example: testing without tokens

You do not need to mint real tokens to test authorization. Spring Security's
`jwt()` request post-processor builds an authenticated `Jwt` directly:

```java
mvc.perform(post("/api/books/1/retitle").content("x").with(jwt()))
   .andExpect(status().isForbidden());                         // authenticated, no scope

mvc.perform(post("/api/books/1/retitle").content("x")
   .with(jwt().authorities(new SimpleGrantedAuthority("SCOPE_books:write"))))
   .andExpect(status().isOk());
```

Both assertions passed in the project's `SecurityTest`. Keep one end-to-end test that
sends a real signed token through the decoder, so the decoder configuration itself is
covered.

## How It Actually Works

`oauth2ResourceServer().jwt()` inserts a `BearerTokenAuthenticationFilter` into the chain.
For each request it:

1. Extracts the token from `Authorization: Bearer …` (a `BearerTokenResolver`).
2. Passes a `BearerTokenAuthenticationToken` to the `AuthenticationManager`, whose
   `JwtAuthenticationProvider` calls the `JwtDecoder`.
3. The `NimbusJwtDecoder` parses the token, looks up the verification key by the header's
   `kid` in its cached JWK set (refetching the set when an unknown `kid` appears — which is
   how key rotation works without restarts), verifies the signature, and runs the
   `OAuth2TokenValidator`s for timestamps (with a default 60-second clock skew), issuer, and
   audience.
4. The `JwtAuthenticationConverter` turns the validated `Jwt` into a
   `JwtAuthenticationToken` with authorities, which is stored in the security context.
5. Failures produce a 401 with a `WWW-Authenticate: Bearer error="invalid_token", …` header,
   as the OAuth2 bearer token spec requires.

No database lookup and no call to the identity provider happens per request — validation
is local, using cached public keys. That is the main operational advantage of JWTs, and
also their main drawback: a token stays valid until it expires even if the user is
disabled, so keep access-token lifetimes short (minutes).

## Common mistakes

- **Not validating the audience**, so a token issued for a different API is accepted.
- **Long-lived access tokens**, making revocation impossible in practice.
- **Using `alg: none` or accepting whatever algorithm the header claims.** Spring's
  decoders are configured with expected algorithms; do not work around them.
- **Putting personal data or secrets in claims**, which are readable by anyone holding the
  token.
- **Building your own login endpoint that issues JWTs** for a public product. Use an
  identity provider or Spring Authorization Server; token issuance has many subtle
  security requirements.

## Exercise

1. Switch your API from Basic auth to a resource server with the HMAC decoder above.
2. Write a small test utility that signs a JWT with the same secret (using Nimbus's
   `MACSigner`, which is on the classpath) and call your API with it using `curl`.
   Then change one character in the payload and observe the 401 and its
   `WWW-Authenticate` header.
3. Add the roles converter and protect an admin endpoint with `hasRole("LIBRARIAN")`.
4. If you have Docker, run Keycloak locally, create a realm and client, switch to
   `issuer-uri`, and call your API with a token obtained from Keycloak.
