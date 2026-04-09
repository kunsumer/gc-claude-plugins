<!-- Last reviewed: 2026-04-09 | Legal effective dates: N/A -->

# Common Pitfalls

Six failure patterns observed when using the yeogibrain MCP, with impact and avoidance strategies.

---

## 1. Exposing document IDs instead of titles

**Impact**: Confuses users with internal slug identifiers that carry no meaning to them. Makes responses look technical and unpolished.

**How to avoid**: Always use the `title` field when presenting results. Document IDs (e.g., `prd-패키지-통합-주문`) are for tool calls only — never show them in prose.

❌ Bad:
> `prd-투어-예약-플로우`를 확인하세요. `res-호텔-pdp-아이템카드`에서 더 자세한 내용을 볼 수 있습니다.

✅ Good:
> 「투어 예약 플로우 기획서」를 확인하세요. 「호텔 PDP, 아이템카드 리서치」에서 더 자세한 내용을 볼 수 있습니다.

---

## 2. Mixing up 1차/2차 sources

**Impact**: Researcher-summarized conclusions get presented as direct user quotes, distorting what users actually said. This undermines the credibility of research synthesis.

**How to avoid**:
- `인터뷰보이스` = **1차 자료** (primary source). Verbatim user statements from interview transcripts.
- `research` = **2차 자료** (secondary source). Researcher-analyzed insights, summaries, and conclusions.

When the user asks "사용자가 뭐라고 했어요?", always search `인터뷰보이스` first. When the user asks for analyzed insights or patterns across many users, use `research`.

Never paraphrase a researcher's summary as if it were a user quote.

---

## 3. Not batching document reads

**Impact**: Multiple sequential `get_document` calls are slow and consume more context than necessary.

**How to avoid**: When you know you need multiple documents (e.g., after `get_related` returns several IDs), call `get_documents_batch` once instead of calling `get_document` in a loop.

❌ Slow:
```
get_document(id="prd-abc")
get_document(id="ac-abc")
get_document(id="ab-abc")
```

✅ Fast:
```
get_documents_batch(ids=["prd-abc", "ac-abc", "ab-abc"])
```

---

## 4. Searching in English only

**Impact**: The majority of yeogibrain content is in Korean. English-only searches miss most of the knowledge base.

**How to avoid**: Default to Korean search terms. Use English as a fallback, not the primary approach.

Priority order:
1. Korean terms (e.g., "투어 예약 취소", "환불 정책")
2. Korean synonyms or alternate phrasing (e.g., "취소 처리", "결제 취소")
3. English terms as supplementary (e.g., "tour booking cancellation")

When a Korean search returns nothing, try synonyms in Korean before switching to English.

---

## 5. Ignoring orientation tools

**Impact**: Diving straight into `search_documents` when entering an unfamiliar domain leads to poorly targeted queries and missed content.

**How to avoid**: Use the orientation tools before searching in an unfamiliar area:
- `get_domain_briefing` — overview of what knowledge exists in a domain area
- `get_wiki_context` — explains internal terminology you may not recognize
- `get_statistics` — shows coverage (how many documents exist per category)

These tools take seconds and prevent "searching blind." Use `get_domain_briefing` especially when a user asks about a topic you have not searched before in this session.

---

## 6. Treating search results as exhaustive

**Impact**: A search returning 0 or few results gets reported as "no information exists," when the real cause is a sub-optimal query.

**How to avoid**: Note the result count in your reasoning. When results are sparse:
1. Try alternate Korean keywords
2. Try `full_text_search` instead of `search_documents`
3. Try a different category
4. Broaden the query term

Be honest when content genuinely does not exist — but only after trying multiple strategies.

❌ Premature conclusion:
> 검색 결과가 없습니다. 관련 정보가 존재하지 않습니다.

✅ Honest, thorough response:
> "반려동물 동반 투어"에 대해 네 가지 검색 전략(category=prd, ab, 인터뷰보이스, full_text_search)을 시도했으나 결과가 없었습니다. 현재 여기브레인에 해당 주제의 기록은 없는 것으로 보입니다. `get_domain_briefing`으로 관련 도메인을 확인해보시겠어요?
