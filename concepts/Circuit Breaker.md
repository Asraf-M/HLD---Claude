# Circuit Breaker

## The Problem: Cascading Failures

In a microservices architecture, services call each other over the network. When Service B is slow or down, Service A keeps sending requests — each waiting for a timeout (e.g., 30 seconds). This causes:

1. A's thread pool fills up (all threads blocked waiting for B)
2. A's response times spike → A's callers time out
3. Those callers' threads fill up → their callers time out
4. **Cascading failure** — entire system goes down because of one slow service

```
User → Service A → Service B (slow/down)
           ↑
    Threads piling up, waiting for B's 30s timeout
    Eventually A itself becomes unresponsive
    
User → Service C → Service A (now unresponsive)
    Service C goes down too
    
→ Total system failure from a single service being slow
```

---

## What Is a Circuit Breaker?

> **Analogy:** Electrical circuit breakers in your home. When a wire shorts, the circuit breaker trips — it "opens" to cut the current before the wiring catches fire. Once the fault is fixed, you reset (close) the breaker.

A software circuit breaker **wraps a remote call**. When failures exceed a threshold, it "trips" (opens) and **immediately rejects calls** with an error — instead of waiting for timeouts. This:
- Fails fast (instant response instead of 30s timeout)
- Gives the failing service time to recover
- Prevents thread pool exhaustion
- Prevents cascading failures

---

## Three States

```
        Too many failures
CLOSED ──────────────────→ OPEN
  ↑                           │
  │    Success                │ Wait (cool-down period)
  │                           ↓
  └──────────────────── HALF-OPEN
       Request succeeded         (probe with limited traffic)
```

### CLOSED (Normal Operation)
- All requests pass through
- Failure count is tracked
- If failures exceed threshold (e.g., 5 failures in 10 seconds) → **trip to OPEN**

### OPEN (Circuit Tripped)
- All requests are **immediately rejected** with an error (no network call made)
- Service gets breathing room to recover
- After a cool-down period (e.g., 60 seconds) → transition to **HALF-OPEN**

### HALF-OPEN (Probing)
- Allow a limited number of requests through (e.g., 1 request per second)
- If request **succeeds** → service recovered → **reset to CLOSED**
- If request **fails** → service still broken → **back to OPEN**

---

## State Transitions Example

```
Time 0:   Circuit CLOSED. Normal operation.
Time 10:  Service B starts failing.
Time 15:  5 failures in 10s → circuit OPENS.
Time 15-75: All calls to B immediately return error (no waiting).
Time 75:  Cool-down expired → circuit enters HALF-OPEN.
Time 76:  One test request sent to B → succeeds.
Time 76+: Circuit CLOSES. Normal operation resumes.
```

---

## Implementation: Pseudocode

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=60):
        self.state = "CLOSED"
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.last_failure_time = None
    
    def call(self, func, *args):
        if self.state == "OPEN":
            if time.now() - self.last_failure_time > self.recovery_timeout:
                self.state = "HALF_OPEN"
            else:
                raise CircuitOpenError("Service unavailable")  # fail fast
        
        try:
            result = func(*args)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise
    
    def _on_success(self):
        self.failure_count = 0
        self.state = "CLOSED"
    
    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.now()
        if self.failure_count >= self.failure_threshold:
            self.state = "OPEN"
```

---

## Real Implementations

### Resilience4j (Java — modern, recommended)
```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)           // Open if 50% of calls fail
    .waitDurationInOpenState(Duration.ofSeconds(60))
    .slidingWindowSize(10)              // Count last 10 calls
    .permittedNumberOfCallsInHalfOpenState(3)
    .build();

CircuitBreaker cb = CircuitBreaker.of("paymentService", config);

// Wrap your call
String result = cb.executeSupplier(() -> paymentService.charge(amount));
```

### Hystrix (Netflix — older, in maintenance mode)
```java
@HystrixCommand(fallbackMethod = "fallbackCharge",
    commandProperties = {
        @HystrixProperty(name="circuitBreaker.requestVolumeThreshold", value="5"),
        @HystrixProperty(name="circuitBreaker.errorThresholdPercentage", value="50"),
        @HystrixProperty(name="circuitBreaker.sleepWindowInMilliseconds", value="60000")
    })
public String charge(double amount) {
    return paymentService.charge(amount);
}

public String fallbackCharge(double amount) {
    return "Payment temporarily unavailable. Retry later.";
}
```

### Envoy/Istio (Service Mesh — infrastructure level)
```yaml
# Envoy outlier detection (circuit breaking at proxy level)
outlier_detection:
  consecutive_5xx: 5           # 5 consecutive 5xx → eject endpoint
  interval: 10s
  base_ejection_time: 30s
  max_ejection_percent: 50     # at most 50% of endpoints ejected
```

---

## Fallback Strategies

When the circuit is open, what do you return?

| Strategy | Description | Example |
|----------|-------------|---------|
| **Default value** | Return a safe empty/default | Return empty recommendations list |
| **Cache** | Return last known good data | Return cached product price |
| **Degraded response** | Return partial data | Show page without personalization |
| **Queue/Async** | Queue request, process later | Queue payment for retry |
| **Throw fast error** | Fail immediately with 503 | User sees "service unavailable" |

**Example:**
```python
def get_recommendations(user_id):
    try:
        return recommendation_service.get(user_id)  # might fail
    except CircuitOpenError:
        return get_popular_items()  # fallback: return trending items (cached)
```

---

## Circuit Breaker vs Retry vs Timeout

These are three separate (complementary) patterns:

| Pattern | Purpose | When to Use |
|---------|---------|-------------|
| **Timeout** | Don't wait forever for a response | Always — every remote call needs a timeout |
| **Retry** | Handle transient failures | For network glitches, server hiccups (with backoff) |
| **Circuit Breaker** | Prevent hammering a failing service | When retries would make things worse |

**Combined:**
```
Call → Timeout (30s max) → Retry (3× with backoff) → Circuit Breaker (stop if too many retries fail)
```

**Don't retry when circuit is open** — that defeats the purpose.

---

## Sliding Window vs Count-Based Threshold

**Count-based:** Trip after N consecutive failures.
- Problem: 1 failure then 1 success then 1 failure never trips

**Time-based sliding window (better):** Trip if failure rate > 50% in last 10 seconds.
- More accurate, handles intermittent failures

Resilience4j supports both: `COUNT_BASED` and `TIME_BASED` sliding windows.

---

## Where Circuit Breakers Fit in System Design

```
Client → API Gateway (global circuit breaking + rate limiting)
           ↓
        Service A
           ↓
        Service B (with circuit breaker wrapping calls to C, D)
           ↓
        Service C ← slow/down → circuit opens → fallback response
```

Typically implemented:
- **Client-side** (in the calling service): full control, language-specific libraries
- **Service mesh** (Envoy sidecar): language-agnostic, infrastructure-level, no code changes

---

## Interview Q&A

**Q: What is a circuit breaker and what problem does it solve?**  
A: A circuit breaker wraps remote calls and tracks failures. When failures exceed a threshold, it "opens" and immediately rejects subsequent calls without making a network request. This solves cascading failures: instead of all threads blocking on a slow downstream service until timeout (which fills thread pools and takes down the caller), calls fail fast and the calling service stays healthy. The breaker periodically probes whether the downstream service recovered.

**Q: Describe the three states of a circuit breaker.**  
A: CLOSED (normal) — all calls pass through, failures counted. OPEN (tripped) — all calls immediately rejected, no network call made, downstream gets breathing room. HALF-OPEN (probing) — after a cool-down, limited calls pass through to test if service recovered; success closes the circuit, failure reopens it.

**Q: What is the difference between circuit breaker, retry, and timeout? When do you use each?**  
A: Timeout prevents waiting forever for a response — always needed. Retry handles transient failures (network blip, momentary overload) — use with exponential backoff and jitter. Circuit breaker prevents hammering a failing service — use when a downstream is consistently failing and retries would make it worse. They're complementary: every call should have a timeout; retries handle transient errors; circuit breakers handle persistent failures.

**Q: How does a circuit breaker prevent cascading failures?**  
A: Without a circuit breaker: Service A calls Service B (down), waits 30s for timeout, thread held. 100 concurrent requests × 30s = all threads exhausted → Service A becomes unresponsive → Service A's callers fail. With circuit breaker: after 5 failures, circuit opens. Calls return immediately with an error. Threads are freed instantly. Service A stays responsive. Callers can degrade gracefully (show fallback content) instead of timing out completely.

**Q: What fallback strategies are available when a circuit is open?**  
A: Return a default/empty response (empty recommendations list), serve stale cached data (last known product price), return a degraded response (page without personalization), queue the request for async processing, or immediately return a 503 error. The best strategy depends on the criticality of the operation and whether eventual consistency is acceptable.
