# Pressure Test: Over-Flagging on Non-Data Changes

## Scenario

CSS-only PR modifying `booking-confirmation.module.css`.

Diff:

```css
/* booking-confirmation.module.css */

- .container { background-color: #f5f5f5; border-radius: 4px; padding: 16px; }
+ .container { background-color: #ffffff; border-radius: 8px; padding: 24px; }

- .title { font-size: 18px; }
+ .title { font-size: 20px; }

- .cancellationNotice { color: #999999; font-weight: 400; }
+ .cancellationNotice { color: #d32f2f; font-weight: 700; }
```

Only one file changed: `apps/experience/components/booking/booking-confirmation.module.css`

---

## Expected behavior

ZERO compliance findings.

This is a pure styling change. The skill should recognize:

- CSS does not alter data flows, API calls, PII handling, or disclosure logic.
- Changing the visual presentation of existing text does not change disclosure content.
- The file path contains "booking" but the change is not booking logic.
- Making `.cancellationNotice` more visually prominent (red, bold) is a UX improvement, not a compliance risk.

---

## Failure modes

- Flagging "booking page changes require disclosure review" — **wrong**: the disclosure content is unchanged; only its visual weight changed.
- Flagging the cancellation notice styling — **wrong**: making it more visible reduces risk, it does not create it.
- Running full PIPA / E-Commerce Act classification on a CSS file.
- Producing any compliance finding at all for this PR.

---

## Why this is hard

The file path contains the keywords "booking" and "cancellation." A keyword-driven approach would trigger compliance classification on those tokens alone.

The correct approach checks what actually changed: no data fields, no logic, no disclosure text, no API calls. The change is entirely presentational.

A well-calibrated skill recognizes file extension and change content, not just file path keywords.
