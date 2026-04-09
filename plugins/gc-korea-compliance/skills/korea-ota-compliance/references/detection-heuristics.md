# Detection Heuristics

These are **heuristics only** — review aids, not legal conclusions.
A match means "look closer", not "this is a violation".

## Regex Patterns

### High-risk resident-like identifier
```regex
/\b\d{6}-?\d{7}\b/
```
May indicate RRN or foreign registration number. Review context before concluding.
→ Rules: PIPA-001, PIPA-002

### Hardcoded secret
```regex
/(password|secret|api[_-]?key|token|private[_-]?key)\s*[=:]\s*["'`].{8,}/i
```
→ Rule: ISMS-001

### High-risk field names
```regex
/(residentRegistration|rrn|passport|foreignerRegistration|visaNumber|cardNumber|cvv|cvc|accountNumber)/i
```
→ Rules: PIPA-001, PIPA-002, EFTA-001

### External non-TLS URL
```regex
/http:\/\/(?!localhost|127\.0\.0\.1|0\.0\.0\.0)/
```
→ Rule: ISMS-004

### Geolocation API usage
```regex
/navigator\.geolocation\.(getCurrentPosition|watchPosition)/
```
→ Rule: LIA-001 (check for pre-prompt notice)

### Pre-checked checkbox
```regex
/(defaultChecked|checked)\s*=\s*\{?\s*true/
```
→ Rule: PIPA-004, NET-001 (consent must not be pre-checked)

## Grep Commands for Code Review

```bash
# RRN-like patterns in source
grep -rn '\b[0-9]\{6\}-\?[0-9]\{7\}\b' --include='*.java' --include='*.ts' --include='*.tsx' --include='*.js'

# Hardcoded secrets
grep -rn -i 'password\|secret\|api[_-]\?key\|token\|private[_-]\?key' --include='*.java' --include='*.ts' --include='*.tsx' --include='*.properties' --include='*.yml' --include='*.yaml' --include='*.env' | grep -i '=\|:'

# High-risk field names
grep -rn -i 'residentRegistration\|rrn\|passport\|foreignerRegistration\|visaNumber\|cardNumber\|cvv\|cvc\|accountNumber' --include='*.java' --include='*.ts' --include='*.tsx' --include='*.sql'

# Cleartext HTTP in config
grep -rn 'http://' --include='*.properties' --include='*.yml' --include='*.yaml' --include='*.env' --include='*.tsx' --include='*.ts' | grep -v localhost | grep -v 127.0.0.1

# Geolocation usage
grep -rn 'navigator\.geolocation\|getCurrentPosition\|watchPosition' --include='*.ts' --include='*.tsx' --include='*.js'

# Pre-checked consent
grep -rn 'defaultChecked.*true\|checked.*true' --include='*.tsx' --include='*.jsx'

# Raw card input elements
grep -rn '<input\|<Input' --include='*.tsx' --include='*.jsx' | grep -i 'card\|cvv\|cvc\|pan\|expir'

# Public port exposure in Docker/K8s
grep -rn '0\.0\.0\.0\|ports:' --include='*.yml' --include='*.yaml' --include='Dockerfile' --include='docker-compose*'

# .env files with values (not just placeholders)
find . -name '.env' -o -name '.env.*' | xargs grep -l '=.' 2>/dev/null | grep -v '.example'
```

## What to do with matches

1. **Context check** — Is this test data, a comment, dead code, or production?
2. **Classification** — Does the matched data actually fall under the rule?
3. **Severity assignment** — Use the severity from the matching rule in `rules.md`.
4. **False-positive note** — If the match is benign, note why in the review output.
