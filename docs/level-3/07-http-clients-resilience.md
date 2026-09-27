---
description: "Call other services from Spring Boot with RestClient and HTTP interface clients, set timeouts, and add retries, circuit breakers, and concurrency limits."
---

# HTTP Clients & Resilience

Every remote call can be slow, fail, or fail slowly — the worst case, because it ties up
your threads while you wait. This lesson covers Spring's modern HTTP clients and the
resilience techniques that keep one misbehaving dependency from taking your service down
with it.

## Choosing a client

| Client | Style | Use it for |
|--------|-------|-----------|
| **`RestClient`** (Framework 6.1+) | Synchronous, fluent | The default choice in Spring MVC apps, especially with virtual threads |
| **`WebClient`** | Reactive (`Mono`/`Flux`) | WebFlux apps, or streaming responses |
| **HTTP interfaces** | Declarative Java interface | Most service-to-service calls; backed by either client |
| `RestTemplate` | Synchronous, template methods | Existing code; in maintenance mode, not for new code |

## RestClient

```java
@Configuration
class PaymentClientConfig {
    @Bean
    RestClient paymentRestClient(RestClient.Builder builder,
                                 @Value("${payments.base-url}") String baseUrl) {
        var settings = HttpClientSettings.defaults()   // Boot 4; Boot 3.4+: ClientHttpRequestFactorySettings
                .withConnectTimeout(Duration.ofSeconds(1))
                .withReadTimeout(Duration.ofSeconds(2));
        return builder
                .baseUrl(baseUrl)
                .requestFactory(ClientHttpRequestFactoryBuilder.detect().build(settings))
                .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
                .build();
    }
}

@Component
class PaymentGateway {
    private final RestClient http;

    PaymentGateway(RestClient paymentRestClient) { this.http = paymentRestClient; }

    PaymentResult charge(ChargeRequest request) {
        return http.post()
                .uri("/charges")
                .body(request)
                .retrieve()
                .onStatus(HttpStatusCode::is4xxClientError, (req, res) -> {
                    throw new PaymentRejectedException(res.getStatusCode());
                })
                .body(PaymentResult.class);
    }
}
```

- Inject Boot's auto-configured **`RestClient.Builder`** rather than calling
  `RestClient.create()`: the builder carries Boot's message converters and, when present,
  observation (metrics and tracing) instrumentation.
- **Always set timeouts.** A connect timeout bounds how long to wait for a TCP connection;
  a read timeout bounds silence while waiting for bytes. Defaults vary by underlying HTTP
  library and can be very long or infinite. `HttpClientSettings` and `ClientHttpRequestFactoryBuilder` live in
  `org.springframework.boot.http.client` (Boot 4's `spring-boot-http-client` module).
  Boot can also apply timeouts globally from properties; those property names have
  changed between Boot releases, so check the reference documentation for your version.
- `retrieve()` throws `RestClientResponseException` subtypes for 4xx/5xx unless you handle
  them with `onStatus`.

## HTTP interfaces

Describe the remote API as an interface:

```java
@HttpExchange("/charges")
interface PaymentsApi {
    @PostExchange
    PaymentResult charge(@RequestBody ChargeRequest request);

    @GetExchange("/{id}")
    PaymentResult get(@PathVariable String id);
}
```

and let Spring implement it:

```java
@Bean
PaymentsApi paymentsApi(RestClient paymentRestClient) {
    return HttpServiceProxyFactory
            .builderFor(RestClientAdapter.create(paymentRestClient))
            .build()
            .createClient(PaymentsApi.class);
}
```

Spring Framework 7 adds `@ImportHttpServices` to register groups of such interfaces with
less code, and Boot 4 can configure those groups from properties. The interface form is
easy to fake in tests (it's just an interface) and keeps call sites clean.

## Resilience patterns

| Pattern | Protects against | Idea |
|---------|-----------------|------|
| **Timeout** | Slow dependencies | Give up after a bound |
| **Retry with backoff + jitter** | Transient failures (a dropped connection, a 503) | Try again, waiting longer each time, randomized so clients don't retry in lockstep |
| **Circuit breaker** | A dependency that is down | After many failures, fail fast for a while instead of calling; probe periodically |
| **Bulkhead / concurrency limit** | One dependency consuming all threads | Cap concurrent calls to it |
| **Fallback** | Degradable features | Return a cached or default answer |

**Only retry idempotent operations** — or make them idempotent. Retrying `POST /charges`
after a read timeout may charge the customer twice, because the first request might have
succeeded. Send an idempotency key the server deduplicates on.

### Built into Spring Framework 7

```java
@Configuration
@EnableResilientMethods
class ResilienceConfig { }

@Component
class ExchangeRates {
    @Retryable(maxRetries = 3, delay = 200, multiplier = 2, jitter = 50,
               includes = ResourceAccessException.class)
    @ConcurrencyLimit(10)
    public Rates latest() { ... }
}
```

`@Retryable` and `@ConcurrencyLimit` (package `org.springframework.resilience.annotation`)
cover retries and bulkheads without extra libraries. There is also a programmatic
`RetryTemplate` in `org.springframework.core.retry`.

### Resilience4j for circuit breakers

For circuit breakers, rate limiters, and time limiters, Resilience4j is the standard
choice. It publishes Spring Boot integration modules per Boot generation
(`resilience4j-spring-boot3`, `resilience4j-spring-boot4`) plus the Spring Cloud Circuit
Breaker abstraction:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      payments:
        sliding-window-size: 20
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
```

```java
@CircuitBreaker(name = "payments", fallbackMethod = "chargeLater")
PaymentResult charge(ChargeRequest request) { ... }

PaymentResult chargeLater(ChargeRequest request, CallNotPermittedException ex) {
    return PaymentResult.pending(request.idempotencyKey());   // queue for later
}
```

The annotation-based integration needs AOP on the classpath (`spring-boot-starter-aspectj`
in Boot 4). The Resilience4j examples in this lesson were not run for the course; check
the Resilience4j documentation for the module matching your Boot version.

## Worked example: testing a client against a fake server

`MockRestServiceServer` intercepts a `RestClient` built from the same builder:

```java
@RestClientTest(PaymentGateway.class)
class PaymentGatewayTest {
    @Autowired PaymentGateway gateway;
    @Autowired MockRestServiceServer server;

    @Test
    void rejectedCardBecomesDomainException() {
        server.expect(requestTo(endsWith("/charges")))
              .andRespond(withStatus(HttpStatus.PAYMENT_REQUIRED));

        assertThatThrownBy(() -> gateway.charge(new ChargeRequest("k-1", 1999)))
                .isInstanceOf(PaymentRejectedException.class);
    }
}
```

For timeout behavior, use a real local HTTP server that delays (WireMock is common) —
`MockRestServiceServer` bypasses the network, so it cannot exercise timeouts.

## How It Actually Works

A `RestClient` is a thin fluent layer over a `ClientHttpRequestFactory` — an adapter
for an underlying HTTP library (the JDK `HttpClient`, Apache HttpClient 5, Jetty, or
Reactor Netty). `ClientHttpRequestFactoryBuilder.detect()` picks one based on what is on
the classpath. Requests pass through `ClientHttpRequestInterceptor`s (where observation,
auth headers, and logging hook in), then the factory's connection pool; the response body
is converted by the same `HttpMessageConverter`s your controllers use.

A circuit breaker is a small state machine around calls. **Closed**: calls pass through and
outcomes are recorded in a sliding window (count- or time-based). When the failure rate
(or slow-call rate) exceeds the threshold, it transitions to **Open**: calls fail
immediately with `CallNotPermittedException`, which costs microseconds instead of a
timeout. After the wait duration it moves to **Half-open** and allows a few trial calls;
success closes the circuit, failure reopens it. The annotation versions are implemented as
AOP proxies — with the usual self-invocation caveat.

## Common mistakes

- **No timeouts.** The default is rarely what you want.
- **Retrying non-idempotent calls** without an idempotency key.
- **Retries stacked on retries** (client, gateway, and service each retry 3 times →
  27 calls to a struggling service).
- **Circuit breaker thresholds set without data.** Look at real error rates and latency.
- **Using `RestClient.create()`** and losing Boot's instrumentation.

## Exercise

1. Build a `PaymentsApi` HTTP interface over a `RestClient` with 1 s connect and 2 s read
   timeouts. Point it at a WireMock (or a tiny second Boot app) that sleeps 5 s, and
   confirm the call fails after about 2 s.
2. Add `@Retryable` for `ResourceAccessException` with backoff and jitter; log each attempt.
3. Add an idempotency key header to charges and explain in a comment why retries are now
   safe.
4. Add a Resilience4j circuit breaker with a fallback, force failures, and watch the state
   transitions via Actuator (`/actuator/circuitbreakers` with the Resilience4j module).
