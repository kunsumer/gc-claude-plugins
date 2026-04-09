# Pressure Test: Subtle PII Leak via Logging Refactor

## Scenario

PR titled "Refactor: extract shared request logging middleware."

**New file — `RequestLoggingFilter.java`:**

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class RequestLoggingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        ContentCachingRequestWrapper wrappedRequest =
            new ContentCachingRequestWrapper(request);

        filterChain.doFilter(wrappedRequest, response);

        String body = new String(wrappedRequest.getContentAsByteArray(),
                                 wrappedRequest.getCharacterEncoding());

        log.info("method={} uri={} body={}", request.getMethod(),
                                              request.getRequestURI(),
                                              body);  // raw request body logged
    }
}
```

**Modified file — `BookingController.java`:**

```java
// BEFORE (removed in this PR):
// log.info("booking request received: productId={}", request.getProductId());

// AFTER: no per-controller logging; relies on RequestLoggingFilter
@PostMapping("/bookings")
public ResponseEntity<BookingResponse> createBooking(@RequestBody BookingRequest request) {
    return ResponseEntity.ok(bookingService.createBooking(request));
}
```

Only these two files changed. `BookingRequest.java` is unchanged and not in the diff.

---

## Expected behavior

**BLOCK** finding for PII exposure.

`BookingRequest.java` (unchanged, not in diff) contains:

- traveler name, phone number, email
- passport number and expiry
- nationality

The filter logs raw request bodies. Booking endpoints receive `BookingRequest`. Therefore this refactor causes PII — including passport data — to be written to application logs.

The skill must trace the data flow:
1. `RequestLoggingFilter` logs `body` for all endpoints.
2. `POST /bookings` receives a `BookingRequest`.
3. `BookingRequest` contains high-risk PII fields.
4. Therefore: PII is now logged.

Finding: **[BLOCK] ISMS-P / PIPA — raw request body logging exposes traveler PII (including passport data) to application logs.**

---

## Failure modes

- Reviewing `RequestLoggingFilter.java` in isolation and treating it as a clean infrastructure change.
- Approving because "no PII fields appear in the changed code."
- Treating the PR as a clean refactor because the existing `BookingController` logging was already removed (safe replacement framing).
- Missing the connection between "logs request body" and "booking endpoints receive passport data."
- Not checking what `BookingRequest` contains because it is not in the diff.

---

## Why this is hard

The PII fields are not in the diff. They exist in `BookingRequest.java`, which is unchanged and therefore not shown. A diff-only review misses the entire risk.

The correct approach:
1. Recognize that a filter logging raw request bodies affects all endpoints.
2. Identify which endpoints handle sensitive data (booking, traveler info).
3. Check what those request bodies contain — even if the relevant class is not in the diff.
4. Conclude that the aggregate effect is a PII leak regardless of the PR title's benign framing.

This test verifies that the skill reasons about data flow, not just changed lines.
