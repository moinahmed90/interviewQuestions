##### **Resilience and Fault Tolerance**



Resilience and Fault Tolerance in microservices means designing services so they can continue operating, degrade gracefully, recover automatically, and avoid cascading failures when parts of the system fail.



**Resilience** is a system's ability to recover gracefully from a failure, adapt to changing conditions, and maintain an acceptable level of service. It assumes that things will fail, focusing heavily on healing, isolation, and degrading gracefully rather than preventing failure altogether.



**Fault Tolerance** is a system's ability to continue operating without interruption or data loss even if one or more components completely crash. It relies heavily on strict hardware or data redundancy to instantly mask the failure, ensuring the end-user never notices a momentary drop in performance.



###### **2.2 Key resilience patterns**



1\. Timeout- A timeout limits how long your service waits for a dependency.

Why it matters: - prevents request threads from being blocked too long - avoids resource exhaustion - helps the system fail fast

In Java/Spring this often appears in: - RestTemplate timeouts - WebClient timeouts - Resilience4j TimeLimiter



2\. Retry- Retry means automatically trying the request again after a failure.

Useful for: - temporary network glitches - transient 5xx responses - short-lived downstream restarts

Danger: - retries can increase load on an already failing service - retries are unsafe for non-idempotent operations unless designed carefully



3\. Circuit Breaker - A circuit breaker stops calling a failing service after repeated failures.

States typically include: - Closed: normal operation, requests go through - Open: calls are blocked immediately - Half-open: limited test requests allowed to check recovery

Benefits: - avoids wasting resources on known failures - helps prevent cascading failure - improves recovery behavior



4\. Fallback - Fallback provides an alternative response when the primary operation fails.

Examples: - return cached product information - return “recommendations unavailable” instead of failing checkout - queue request for later processing



5\. Bulkhead - Bulkhead isolates resources so one failing dependency does not consume everything.

Examples: - separate thread pools per integration - semaphore limits for concurrent calls - isolated connection pools



6\. Rate Limiting - Rate limiting controls how many requests are allowed in a period.

Benefits: - protects the service from overload - prevents noisy neighbors - improves stability under traffic spikes



7\. Idempotency - Idempotency means repeated requests produce the same effect.

Important for retries: - GET requests are usually idempotent - POST payment capture often is not unless protected by an idempotency key



8\. Graceful Degradation - The system remains usable with reduced functionality.

Example: - user can still place an order even if reviews are unavailable

\-------------------------------------------------------------------

In Spring Boot, you often apply resilience through annotations:These annotations typically wrap a method that calls an external dependency.



* @Retry(name = "inventoryService", fallbackMethod = "inventoryFallback")

&#x09;Use retry when: - failures are temporary - the operation is idempotent or safely repeatable - a short retry delay can help the downstream recover

&#x09;Do not retry blindly on validation failures, authentication failures, or non-idempotent financial operations without safeguards.

resilience4j:

&#x20; retry:

&#x20;   instances:

&#x20;     paymentService:

&#x20;       maxAttempts: 3

&#x20;       waitDuration: 1s

&#x20;       retryExceptions:

&#x20;         - java.lang.RuntimeException



* @CircuitBreaker(name = "inventoryService", fallbackMethod = "inventoryFallback")

&#x09;Use a circuit breaker when: - a dependency is unstable or overloaded - repeated failures would waste threads and connections - you want fast failure 	and graceful degradation

resilience4j:

&#x20; circuitbreaker:

&#x20;   instances:

&#x20;     inventoryService:

&#x20;       slidingWindowSize: 5

&#x20;       minimumNumberOfCalls: 5

&#x20;       failureRateThreshold: 50

&#x20;       waitDurationInOpenState: 10s

&#x20;       permittedNumberOfCallsInHalfOpenState: 2



* @TimeLimiter(name = "inventoryService")
* @Bulkhead(name = "inventoryService")

&#x09;@Bulkhead(name = "shippingService", type = Bulkhead.Type.SEMAPHORE, fallbackMethod = "shippingFallback")

resilience4j:

&#x20; bulkhead:

&#x20;   instances:

&#x20;     shippingService:

&#x20;       maxConcurrentCalls: 2

&#x20;       maxWaitDuration: 0

Use bulkheads when: - a dependency can become slow under load - you want to protect your request-handling threads - one integration should not consume all capacity

\----------------------------------------------------------------------------------------------





&#x20;   private final WebClient webClient;



&#x20;   public CatalogServiceClient(WebClient.Builder builder) {

&#x20;       this.webClient = builder.baseUrl("http://localhost:8082").build();

&#x20;   }



&#x20;   @Retry(name = "catalogService", fallbackMethod = "catalogFallback")

&#x20;   @CircuitBreaker(name = "catalogService", fallbackMethod = "catalogFallback")

&#x20;   @TimeLimiter(name = "catalogService")

&#x20;   public CompletableFuture<String> getCatalogData(String productId) {

&#x20;       return webClient.get()

&#x20;               .uri("/catalog/{id}", productId)

&#x20;               .retrieve()

&#x20;               .bodyToMono(String.class)

&#x20;               .toFuture();

&#x20;   }



&#x20;   public CompletableFuture<String> catalogFallback(String productId, Exception ex) {

&#x20;       return CompletableFuture.completedFuture(

&#x20;               "Fallback catalog data for product " + productId

&#x20;       );

&#x20;   }



This example stacks several resilience patterns:



* TimeLimiter: aborts if the remote service takes too long
* Retry: retries transient failure
* CircuitBreaker: prevents repeated calls to a failing service
* Fallback: returns substitute data

Why CompletableFuture? - @TimeLimiter typically works with asynchronous return types - this is common with non-blocking or async integration styles

\-----------------------------------------------------------------------------------

Best practices

When using Resilience4j in Spring Boot, follow these guidelines:



1\. Use retries selectively

Do not retry everything. Retry only transient failures such as: - connection resets - temporary timeouts - 5xx upstream errors in some cases



Avoid retries for: - validation errors - authentication failures - business rule violations



2\. Always define meaningful fallbacks

A fallback should: - degrade gracefully - preserve user experience where possible - avoid hiding serious issues silently



Bad fallback:



return "error";

Better fallback:



return "Payment service temporarily unavailable. Please try again later.";

3\. Tune circuit breaker thresholds carefully

Default values may not match your production traffic.



Pay attention to: - request volume - acceptable failure rate - recovery time - startup traffic patterns



4\. Prefer TimeLimiter for slow dependencies

A dependency does not need to fail completely to hurt your system. Slow services can still exhaust threads and degrade latency.



5\. Use Bulkhead for expensive downstream calls

Bulkheads are especially useful when: - integrating with third-party APIs - one dependency can become slow - you need isolation between subsystems



6\. Monitor health and metrics

Integrate with: - Spring Boot Actuator - Micrometer - Prometheus - Grafana



Track: - retry counts - circuit breaker states - failure rates - slow call rates - bulkhead rejections



7\. Match patterns to failure type

Examples: - Retry for transient errors - CircuitBreaker for repeated failures - RateLimiter for controlling outgoing request rate - Bulkhead for isolation - TimeLimiter for latency control



8\. Keep fallback methods simple

Fallbacks should not become another fragile dependency chain. Prefer: - cached values - defaults - user-friendly messages - queueing for later processing when appropriate



9\. Test failure scenarios explicitly

Do not test only the happy path.



Test: - timeout behavior - fallback activation - breaker opening and closing - retry exhaustion - concurrency saturation with bulkhead

\-----------------------------------------------------------------------------------------

Q1

Correct

10 / 10 pts

Which statement best distinguishes resilience from fault tolerance?



Your answer: Resilience focuses on system's ability to recover or adapt after faults to maintain acceptable service; fault tolerance focuses on continuing correct operation during faults, often via redundancy.



Expected: Resilience focuses on system's ability to recover or adapt after faults to maintain acceptable service; fault tolerance focuses on continuing correct operation during faults, often via redundancy.



Q2

Correct

10 / 10 pts

Graceful degradation means that a system under stress will intentionally reduce or disable non-essential features while preserving core functionality to avoid a total outage.



Your answer: true



Expected: true



Q3

Correct

10 / 10 pts

Match each fault-tolerance technique to its primary purpose.



Column A

A. Replication

B. Failover

C. Checkpointing

D. Load shedding

Column B

1\. Automatic switching to a standby component when the primary fails.

2\. Dropping lower-priority requests to keep core services responsive during overload.

3\. Maintaining multiple synchronized copies of state across nodes to tolerate node or disk failures.

4\. Periodic saving of execution state so processes can restart from a recent point after a crash.

Your answer: A-3, B-1, C-4, D-2



Expected: A-3, B-1, C-4, D-2



Q4

Correct

10 / 10 pts

Scenario: A distributed database's primary node crashed irrecoverably. A fully-synchronized secondary replica exists and can accept writes after promotion. The application is experiencing write failures. Which immediate fault-tolerance action most directly restores write availability with minimal data loss?



Your answer: Promote the synchronized secondary replica to primary (manual or automated failover).



Expected: Promote the synchronized secondary replica to primary (manual or automated failover).



Q5

Incorrect

0 / 10 pts

Which design decision most improves fault tolerance for a service that currently stores session data in memory and must withstand node failures?



Your answer: Increase in-memory cache size on each node to tolerate more concurrent sessions.



Expected: Refactor the service to be stateless and store session data in a dedicated external store.



Q6

Correct

10 / 10 pts

Scenario: A payment processing service calls an external fraud-check API that intermittently returns 503. The service uses a circuit breaker with a failure threshold of 5 failures per minute, an open timeout of 10 seconds, and a retry policy of 3 retries per request. During an incident, many requests queued up waiting for retries, causing thread exhaustion and cascading failures. Which single configuration change would most directly reduce the risk of cascading failures while still allowing recovery when the external API becomes healthy?



Your answer: Decrease the per-request retry attempts from 3 to 0 to avoid queued retries piling up.



Expected: Decrease the per-request retry attempts from 3 to 0 to avoid queued retries piling up.



Q7

Incorrect

0 / 10 pts

True or False: Active-active redundancy always provides faster failover than active-passive redundancy because all nodes are already serving traffic.



Your answer: true



Expected: false



Q8

Correct

10 / 10 pts

Match each recovery objective or reliability metric to its correct definition.



Column A

A. RTO

B. RPO

C. MTTR

D. MTBF

Column B

1\. Average elapsed time between the start of one failure and the start of the next failure across a population of systems (indicator of reliability).

2\. Maximum acceptable time to restore service after an outage.

3\. Maximum acceptable amount of data loss expressed as a time window (how far back data can be recovered).

4\. Average time required to repair or recover a system after a failure.

Your answer: A-2, B-3, C-4, D-1



Expected: A-2, B-3, C-4, D-1

