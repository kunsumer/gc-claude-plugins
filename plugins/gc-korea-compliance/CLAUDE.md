# GC Korea Compliance Plugin

## Philosophy
A good compliance review catches real issues without crying wolf. Precision
matters more than recall — one false BLOCK erodes team trust faster than one
missed INFO builds it. When in doubt, use REVIEW severity to escalate rather
than BLOCK to mandate.

## Top 3 mistakes to avoid
1. Over-applying rules whose scope gates aren't satisfied (check "Applies when")
2. Presenting internal baselines (AES-256, KMS rotation) as statutory mandates
3. Producing shallow findings without specific file references, statute citations,
   or actionable remediation

## When NOT to use this skill
- Pure frontend styling/layout changes with no data or disclosure implications
- Infrastructure changes (CI/CD, build config) that don't touch data flows
- Documentation-only changes

## What this plugin covers
- PIPA, ISMS-P, E-Commerce Act, EFTA, LIA, Tourism Promotion Act frameworks
- Detection heuristics for code review
- Checklists for PRD, UX, architecture, and code reviews
- Severity classification (BLOCK / REVIEW / WARN / INFO)
- Reviewer guardrails to prevent over-application of rules

## What this plugin does NOT cover
- Project-specific data classes and compliance triggers
- Project-specific boundary maps and ownership rules

Projects should add their own repo-local overlay to extend this plugin with
domain-specific context.

## Cross-plugin handoffs
- If you find a schema migration with PII implications, suggest running the
  `safe-migration` skill
