---
description: Design or generate a safe schema migration with rollback, data impact, and compliance considerations.
argument-hint: [schema change description]
---

# /gen-migration

Design or generate this migration: $ARGUMENTS

## Goal
Produce a migration artifact that is operationally safe, reversible where
possible, and explicit about data and compliance impact.

## Read first
- project ERD or schema documentation (if available)
- project compliance documentation (if available)
- existing migration style and naming conventions in the repo
- this plugin's `safe-migration` skill

## Steps

### 1. Classify the change
Determine type: additive, backfill, destructive, rename, split/merge, privacy-sensitive.

### 2. Identify blast radius
Map affected entities, indexes, foreign keys, MyBatis mappers, and downstream consumers.

### 3. Check compliance triggers
Determine whether new or changed fields involve PII, payment-adjacent data,
or regulated identifiers. If so, flag for compliance review.

### 4. Generate artifacts
Produce:
- Migration file following repo conventions
- Rollback or recovery script
- Backfill strategy when existing data must change
- Audit/logging implications
- Verification notes

### 5. Flag risks
Call out destructive changes that require backup or staged rollout.
State what happens if the migration fails mid-way.

## Non-negotiables
- Never add plaintext sensitive identifiers prohibited by policy.
- Never ship a destructive migration without a rollback/recovery note.
- Never hide data rewrites inside ambiguous migration names.

## Output
Write or update migration artifacts plus rollout notes.
