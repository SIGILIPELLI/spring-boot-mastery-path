---
description: "Understand Spring Security's filter chain, authentication versus authorization, SecurityFilterChain configuration, password hashing, and method security."
---

# Spring Security Fundamentals

Add `spring-boot-starter-security` to a project and every endpoint suddenly returns 401.
That is Spring Security's secure-by-default stance, and it is the right one. This lesson
explains what just happened, and how to replace the defaults with rules that fit an API.

## What the defaults do

With the starter on the classpath and no configuration of your own, Boot:

- Requires authentication for every request.
- Enables HTTP Basic and form login.
- Creates one in-memory user named `user` with a random password printed in the log at
  startup ("Using generated security password: …").
- Enables CSRF protection, security headers (`X-Content-Type-Options`, cache control,
  `X-Frame-Options`, HSTS on HTTPS), and session fixation protection.

Useful for five minutes. Then you write your own `SecurityFilterChain`.

## Authentication vs authorization

- **Authentication** answers *who are you?* — verifying a password, a session cookie, or a
  token, and producing an `Authentication` object with a principal and granted
  authorities.
- **Authorization** answers *are you allowed to do this?* — checking those authorities
  against rules for the URL or method.

A 401 means authentication failed or was missing; a 403 means you are known but not
allowed.

## Configuring the chain

```java
@Configuration
public class SecurityConfig {

    @Bean
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/books/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .httpBasic(Customizer.withDefaults())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .csrf(csrf -> csrf.disable());
        return http.build();
    }
}
```

- Rules are evaluated **in order**; the first match wins. Put specific rules before general
  ones and end with `anyRequest()`.
- `hasRole("ADMIN")` checks for the authority `ROLE_ADMIN`; `hasAuthority("books:write")`
  checks the exact string.
- A stateless API authenticates every request from its credentials (Basic here, a JWT in
  the next lesson), so no HTTP session is created.
- **CSRF**: disable it only for APIs that do not use cookies for authentication. CSRF
  attacks rely on the browser automatically attaching cookies; a bearer token in an
  `Authorization` header is not attached automatically. If your API uses session cookies
  (a browser app on the same site), keep CSRF protection on.

## Users and passwords

For a small internal tool, users can come from your database through a
`UserDetailsService`:

```java
@Bean
UserDetailsService users(MemberRepository members) {
    return username -> members.findByEmail(username)
            .map(m -> User.withUsername(m.getEmail())
                    .password(m.getPasswordHash())
                    .roles(m.getRoles().toArray(String[]::new))
                    .build())
            .orElseThrow(() -> new UsernameNotFoundException(username));
}

@Bean
PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
```

Never store passwords reversibly. The delegating encoder hashes new passwords with bcrypt
by default and stores them with a prefix (`{bcrypt}$2a$10$…`), so you can migrate to a
stronger algorithm later while old hashes keep working. bcrypt is deliberately slow; that
is the point — it makes offline guessing expensive.

For most real products you should not manage passwords at all: delegate login to an
identity provider (Keycloak, Auth0, Okta, Entra ID, Cognito, Google) and have your API
accept the tokens it issues. That is the next lesson.

## Method security

URL rules are coarse. For "only the member who owns this loan may return it," use method
security:

```java
@Configuration
@EnableMethodSecurity
class MethodSecurityConfig { }

@Service
class LoanService {
    @PreAuthorize("hasRole('LIBRARIAN') or @loanAccess.isBorrower(#loanId, authentication)")
    public void returnLoan(long loanId) { ... }
}
```

`@PreAuthorize` evaluates a SpEL expression before the method runs; `@loanAccess` refers to
a bean you write. It is implemented with the same proxies as `@Transactional` — and has
the same self-invocation limitation.

## Worked example: rules under test

The Level 2 project's rules were verified with Spring Security's test support (this test
passed on Boot 4.1):

```java
@SpringBootTest
@AutoConfigureMockMvc
class SecurityTest {
    @Autowired MockMvc mvc;

    @Test
    void rules() throws Exception {
        mvc.perform(get("/api/books")).andExpect(status().isOk());
        mvc.perform(post("/api/books/1/retitle").content("x")).andExpect(status().isUnauthorized());
        mvc.perform(post("/api/books/1/retitle").content("x").with(jwt()))
           .andExpect(status().isForbidden());
        mvc.perform(post("/api/books/1/retitle").content("x")
           .with(jwt().authorities(new SimpleGrantedAuthority("SCOPE_books:write"))))
           .andExpect(status().isOk());
    }
}
```

Anonymous GET: allowed. Anonymous POST: 401. Authenticated without the right authority:
403. With it: 200. Four assertions pin the whole policy.

## How It Actually Works

Spring Security is a **servlet filter**. Boot registers one filter,
`DelegatingFilterProxy`, named `springSecurityFilterChain`, which delegates to a
`FilterChainProxy` bean. That proxy holds one or more `SecurityFilterChain`s — each with a
request matcher and an ordered list of filters — and uses the **first chain whose matcher
fits** the request.

A typical API chain contains, in order, filters such as: `SecurityContextHolderFilter`
(sets up an empty security context), `HeaderWriterFilter`, `CsrfFilter`, an authentication
filter (`BasicAuthenticationFilter` or `BearerTokenAuthenticationFilter`),
`ExceptionTranslationFilter`, and finally `AuthorizationFilter`.

- An authentication filter extracts credentials and hands them to the
  `AuthenticationManager`, which asks its `AuthenticationProvider`s (for example
  `DaoAuthenticationProvider`, which loads the user and checks the password hash). On
  success the resulting `Authentication` is stored in the `SecurityContextHolder` — a
  `ThreadLocal` — for the rest of the request.
- `AuthorizationFilter` checks the request against your `authorizeHttpRequests` rules
  using the `Authentication` in the context.
- If it throws `AccessDeniedException`, `ExceptionTranslationFilter` (earlier in the chain,
  wrapping everything after it) catches it and responds with 401 via the
  `AuthenticationEntryPoint` if the user is anonymous, or 403 via the
  `AccessDeniedHandler` if not.

Because all of this happens in filters *before* the `DispatcherServlet`, your
`@RestControllerAdvice` never sees these 401s and 403s. To make them return problem
details, configure a custom entry point and access-denied handler on the chain.

## Common mistakes

- **Rule order bugs**: `anyRequest().authenticated()` placed before a `permitAll()` rule
  makes the later rule unreachable (Spring Security fails fast on `anyRequest` not being
  last).
- **Disabling CSRF on a cookie-authenticated browser app.**
- **Confusing roles and authorities** — `hasRole("ADMIN")` vs `hasAuthority("ADMIN")`.
- **Using `@PreAuthorize` without `@EnableMethodSecurity`** — the annotation is ignored.
- **Rolling your own password hashing** or storing tokens in plain text.

## Exercise

1. Add Spring Security to your Level 1 task API. Observe the defaults, then write a
   `SecurityFilterChain` that permits `GET` to everyone and requires authentication for
   writes, using HTTP Basic and an in-memory user with a bcrypt-encoded password.
2. Write a `@WebMvcTest` using `@WithMockUser` and anonymous requests that pins each rule.
3. Add an entry point and access-denied handler that return `application/problem+json`.
4. Add `@PreAuthorize` to a service method and write a test proving it is enforced — then
   call it via self-invocation and show that it is not.
