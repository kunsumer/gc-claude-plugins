# Worked Example: False Negative — Logging Refactor That Exposes PII

<!-- Last reviewed: 2026-04-09 -->

This example documents a tricky change that a shallow compliance review would pass as clean
but that actually introduces a serious PII-in-logs violation. Use this example to calibrate
the skill's data-flow tracing and to train reviewers on the failure modes that produce
false negatives.

---

## Why this example exists

Most PII-in-logs findings are straightforward: a developer calls `log.info("{}", entity)` and
the entity contains personal data. This example is harder because:

- The controller code itself looks unchanged and clean.
- The PR description frames the change as a pure infrastructure refactor with no data-handling
  implications.
- The violation is introduced in shared middleware that sits outside the domain layer.
- The risk is not visible without tracing what data flows through the logged request body.

This is exactly the class of change most likely to slip through a keyword-only or
surface-level review.

---

## Input

**Artifact type:** Code review
**PR title:** "Refactor: extract shared request logging middleware"
**PR description:**
> Moves request/response logging out of individual controllers into a shared
> `RequestLoggingFilter`. No functional changes — this is infrastructure cleanup only.
> Removes duplicate log statements from BookingController, CancellationController,
> and VoucherController.

### Before (controller-level logging — in scope for 3 controllers)

```java
// BookingController.java — BEFORE
@PostMapping
public ResponseEntity<BookingResponse> createBooking(
        @RequestBody @Valid BookingRequest request,
        @AuthenticationPrincipal TravelerPrincipal principal) {

    log.info("Booking request received — productId={}", request.productId());

    return ResponseEntity.status(HttpStatus.CREATED)
            .body(bookingService.createBooking(principal.getTravelerId(), request));
}
```

The before state logs only `productId`, a non-personal identifier. No PII is present in any
of the three original controller log statements.

### After (shared filter — replaces all three controller log statements)

```java
// RequestLoggingFilter.java — AFTER
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class RequestLoggingFilter extends OncePerRequestFilter {

    private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        ContentCachingRequestWrapper wrapped =
                new ContentCachingRequestWrapper(request);

        filterChain.doFilter(wrapped, response);

        String body = new String(wrapped.getContentAsByteArray(),   // line 27
                StandardCharsets.UTF_8);

        log.info("Inbound request — method={} uri={} body={}",      // line 30
                request.getMethod(), request.getRequestURI(), body);
    }
}
```

---

## What makes this tricky

1. **Framing mismatch.** The PR description says "no functional changes — infrastructure
   cleanup only." A reviewer who trusts the description may not trace what `body` contains
   at line 30.

2. **The controller code is now clean.** Removing the controller-level log statements looks
   like an improvement. The new code in `RequestLoggingFilter` is in a different file and a
   different layer, so a reviewer focused on the controllers may not read it carefully.

3. **The filter applies to all endpoints globally.** Unlike the original controller log which
   only applied to `POST /bookings`, the filter captures the body of every POST, PUT, and PATCH
   request across the entire service. This includes booking, cancellation, and voucher creation
   — all of which carry high-risk PII.

4. **`wrapped.getContentAsByteArray()` is opaque at a glance.** It does not contain the word
   "PII," "name," "phone," or "passport." A reviewer who does not trace what bytes are in the
   buffer will not catch the violation.

---

## How the skill should catch it

The correct review sequence traces the data flow from the request body to the log sink.

**Step 1 — Identify what the filter logs.**
`wrapped.getContentAsByteArray()` returns the raw bytes of the HTTP request body. For
`POST /api/v1/bookings`, this body is the JSON serialisation of `BookingRequest`.

**Step 2 — Identify what `BookingRequest` contains.**
Cross-reference the existing `BookingRequest` record (visible from the code-review example or
the repo):
`travelerName`, `travelerPhone`, `travelerEmail`, `passportNumber`, `nationality`, `birthDate`,
`productId`, `optionId`, `tourDate`, `quantity`.

High-risk PII fields present: `travelerName`, `travelerPhone`, `travelerEmail`,
`passportNumber` (고유식별정보), `birthDate`.

**Step 3 — Confirm the filter is applied globally.**
`@Component` + `OncePerRequestFilter` with `@Order(Ordered.HIGHEST_PRECEDENCE)` means this
filter runs for every request handled by the servlet. There is no path allowlist or exclusion
pattern. The filter therefore logs PII from booking, cancellation, and voucher endpoints.

**Step 4 — Apply the rule.**
PIPA-001: Personal data must not be logged in plain text. This filter writes the full request
body — including passport number and contact details — to the application log on every
booking-related request. This is a BLOCK finding regardless of the PR's stated intent.

---

## Expected finding

```text
Severity : BLOCK
Rule     : PIPA-001 — Raw PII in application logs via shared request filter
Location : RequestLoggingFilter.java, lines 27–30
```

**What the code does:** `wrapped.getContentAsByteArray()` captures the full serialised JSON
request body. For booking, cancellation, and voucher endpoints, this body contains high-risk
personal data including traveler name, phone, email, passport number, and birth date.
The complete body is written to the log at line 30 with no masking or field-level redaction.
Because the filter has no path restriction, it applies globally to all POST/PUT/PATCH requests.

**왜 문제인가 (Why this matters):**
개인정보보호법 제29조는 개인정보가 분실·도난·유출·위조·변조 또는 훼손되지 않도록
안전성 확보에 필요한 기술적·관리적 보호조치를 취하도록 규정합니다.
애플리케이션 로그는 일반적으로 개발자, 운영자, 로그 수집·분석 플랫폼(예: ELK, Splunk)에
넓게 공유됩니다. 여권번호(고유식별정보)가 평문으로 로그에 기록될 경우,
접근 권한이 있는 모든 인원에게 불필요하게 노출됩니다.

이 위반이 실제 유출로 이어지지 않더라도, ISMS-P 인증 심사에서 중결함으로 분류되고
개인정보보호위원회 현장 점검 시 시정명령 및 최대 전년도 매출액 3% 과징금의 근거가 됩니다.
"리팩터링이었기 때문에 의도하지 않은 변경이었다"는 항변은 위반 사실을 면제하지 않습니다.

**Fix options:**

1. **Path allowlist — log only non-sensitive endpoints:**
   ```java
   private static final Set<String> SAFE_LOG_PATHS = Set.of(
           "/api/v1/products", "/api/v1/options", "/actuator/health");

   // In doFilterInternal, before logging:
   if (SAFE_LOG_PATHS.stream().noneMatch(request.getRequestURI()::startsWith)) {
       log.info("Inbound request — method={} uri={}", // no body
               request.getMethod(), request.getRequestURI());
       return;
   }
   ```

2. **Field-level redaction — mask known PII keys in the JSON body:**
   ```java
   String safeBody = MaskUtils.maskJsonFields(body,
           "travelerName", "travelerPhone", "travelerEmail",
           "passportNumber", "nationality", "birthDate",
           "bankAccountNumber", "accountHolderName");
   log.info("Inbound request — method={} uri={} body={}",
           request.getMethod(), request.getRequestURI(), safeBody);
   ```

3. **Log only structural metadata, never body content:**
   ```java
   log.info("Inbound request — method={} uri={} contentType={} contentLength={}",
           request.getMethod(), request.getRequestURI(),
           request.getContentType(), request.getContentLength());
   ```
   This is the safest option and is sufficient for most operational debugging purposes.

---

## Common failure mode

A reviewer who produces a **false negative** on this PR typically does one of the following:

- Reads the PR description ("no functional changes") and performs only a spot-check on the
  changed files, not a full data-flow trace.
- Notes that the controller code is now cleaner (PII-specific log lines removed) and marks
  that as an improvement without reading `RequestLoggingFilter` carefully.
- Sees `getContentAsByteArray()` but does not ask "what is in those bytes for this endpoint?"
- Applies a keyword scan (`passportNumber`, `travelerPhone`, etc.) to the diff, finds no
  matches in variable names, and concludes there is no PII exposure — missing that the
  exposure is indirect via the serialised body.

The skill must not rely on keyword matching alone. When a log statement writes a variable
whose value is derived from an HTTP request body, the review must trace what that body
contains for the endpoints the code handles.
