---
name: yeogibrain-usage
description: >
  Guidance for effectively querying the yeogibrain MCP server. Use when
  searching for PRDs, acceptance criteria, AB test results, UX research,
  user interview transcripts, benchmarks, or internal product knowledge.
user-invocable: true
---

# Yeogibrain Usage Guide

yeogibrain is GC Company's internal product knowledge search MCP server.
It indexes PRDs, acceptance criteria, AB tests, UX research, and user
interview transcripts.

## Categories

| Category | Korean | What it contains |
|---|---|---|
| `prd` | 기획서 | Feature definitions, policies, target customers, benchmarks |
| `ac` | 수용 기준 | Detailed implementation specs and checklists based on PRDs |
| `ab` | AB 테스트 | Experiment design, results, and lessons learned |
| `research` | 리서치 | UX research reports (secondary sources: UT, IDI, Survey, FGI) |
| `인터뷰보이스` | 인터뷰보이스 | Interview transcripts (primary sources: actual user statements) |

## Tool selection by use case

| Purpose | Recommended tool | Category/options |
|---|---|---|
| Find actual user statements/quotes | `full_text_search` → `get_document` | category=인터뷰보이스 |
| Research insights and conclusions | `search_documents` | category=research |
| Feature specs, policies, target users | `search_documents` | category=prd |
| Benchmarks and competitor research | `search_documents` | category=prd |
| Acceptance criteria / implementation specs | `search_documents` | category=ac |
| AB test results and lessons | `search_documents` | category=ab |
| Deep full-text search across all content | `full_text_search` | - |
| Find related documents (PRD↔AC↔AB) | `get_related` | - |
| Browse by specific filters | `list_documents` | - |

## Critical: Primary vs. secondary sources

- **User voice and actual statements** → Always search `인터뷰보이스` (interview transcripts) first. These are primary sources with verbatim user quotes.
- **Research reports** → `research` category contains researcher-curated secondary sources. Use when you need analyzed insights, not raw quotes.
- When the user asks for "user voice" or "what users said", always prioritize 인터뷰보이스 over research.

## Critical: Benchmark searches

- **Benchmarks and competitor analysis** → Search `prd` category first. The PRD 1-pager template includes a standard "Benchmarks" section.
- Research documents with "gap analysis" (갭분석) methodology may also contain competitor comparisons — use as a secondary source.

## Result display rules

- Document IDs are slug-formatted (e.g., `prd-패키지-통합-주문`). These are internal identifiers.
- **Always use document titles** when presenting results to users. Do not expose raw IDs.
- Example: Use "「호텔 PDP, 아이템카드」 리서치를 참고하세요" not "res-호텔-pdp-아이템카드를 참고하세요"

## Search strategy

1. Start with `search_documents` using the most specific category for your use case
2. If results are insufficient, broaden with `full_text_search` across all categories
3. Use `get_related` to discover connected documents (e.g., find the AC for a PRD)
4. Use `get_document` to read the full content of a specific document
5. Use `get_documents_batch` when you need to read multiple documents at once

## Domain briefing integration

Before searching in an unfamiliar domain, orient yourself with the domain tools:

- `get_domain_briefing` — call this when you encounter a topic area you have not searched before in this session. It gives a structured overview of what knowledge exists without guessing at keywords.
- `get_wiki_context` — call this when you encounter internal terminology (product names, feature codenames, internal processes) that you are not familiar with. Do not guess at meaning.
- `get_statistics` — call this when you want to understand coverage (e.g., how many PRDs, how many interview transcripts exist in a given area) before deciding on a search strategy.

Using these tools prevents "searching blind" — the failure mode of running repeated poorly-targeted queries because you do not know what content exists or what internal terms are used.

**Pattern**: Unfamiliar domain → `get_domain_briefing` → targeted search → `get_related` → `get_documents_batch`

## Multi-step research workflows

Three common research patterns with full tool sequences:

### "Is this idea new?"
1. `search_documents(category="prd")` — check if a spec was ever written
2. `search_documents(category="ab")` — check if an experiment was run
3. `search_documents(category="인터뷰보이스")` — check if users have raised this need
4. Synthesize: prior art found / partially explored / no history

### "What do users think about X?"
1. `full_text_search(category="인터뷰보이스")` — get verbatim user statements (primary source)
2. `search_documents(category="research")` — get analyzed insights (secondary source)
3. `get_related` on key documents — discover linked specs or experiments
4. Present verbatim quotes first, then synthesis, labeling the source distinction

### "Full history of feature Y?"
1. `search_documents(category="prd")` — find the originating spec
2. `get_related` — surface connected AC and AB documents
3. `get_documents_batch` — read all connected documents at once
4. Present chronologically: PRD → AC → AB results → current state

See `references/query-examples.md` for full tool call syntax and Korean output examples.
See `references/common-pitfalls.md` for the six most common mistakes and how to avoid them.

## Self-review checklist

Before delivering a yeogibrain-based response, verify:

1. **Titles, not IDs** — every document reference uses the `title` field, not the slug ID.
2. **Source distinction** — `인터뷰보이스` results are labeled as primary / verbatim; `research` results are labeled as secondary / synthesized. Never present a researcher's summary as a user quote.
3. **Document chain followed** — after finding a key document, `get_related` was called to check for connected AC and AB documents.
4. **Korean terms tried first** — if a search returned nothing, Korean synonyms and alternate phrasing were tried before English fallback.
5. **Incomplete results noted** — if search coverage may be incomplete (low result count, niche topic), this is stated explicitly rather than implied to be exhaustive.
6. **Orientation tool used when needed** — if the domain was unfamiliar, `get_domain_briefing` or `get_wiki_context` was called before searching.
