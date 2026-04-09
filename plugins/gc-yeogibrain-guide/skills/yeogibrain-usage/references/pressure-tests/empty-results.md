# Pressure Test: Empty Results — Niche Topic

## Scenario

User asks:

> "반려동물 동반 투어에 대한 자료 있어요?"

"반려동물 동반 투어" (pet-friendly tours) is a niche topic unlikely to have dedicated documents.

---

## Expected behavior

Try **at least 4 search strategies** before concluding that no content exists.

**Step 1 — Korean primary search:**
```
search_documents(query="반려동물 동반 투어", category="prd")
```
→ (likely) 결과 없음

**Step 2 — Korean synonym / alternate phrasing:**
```
search_documents(query="펫 투어 강아지 동반", category="prd")
```
→ (likely) 결과 없음

**Step 3 — Try different categories:**
```
search_documents(query="반려동물 투어", category="research")
full_text_search(query="반려동물 투어")
```
→ (likely) 결과 없음

**Step 4 — English fallback:**
```
search_documents(query="pet friendly tour", category="prd")
```
→ (likely) 결과 없음

**After exhausting strategies, report honestly and suggest orientation:**

> "반려동물 동반 투어"에 대해 다음 네 가지 전략으로 검색했으나 관련 문서를 찾지 못했습니다:
> 1. category=prd, 한국어 검색
> 2. 유사어(펫 투어, 강아지 동반) 검색
> 3. category=research 및 full_text_search
> 4. 영어 키워드(pet friendly tour) 검색
>
> 현재 여기브레인에 해당 주제의 기록은 없는 것으로 보입니다.
> `get_domain_briefing`으로 투어 도메인 전반의 자료 현황을 확인해보시겠어요?

---

## Failure modes

- Running a single search and immediately reporting "관련 정보가 없습니다."
- Fabricating results or hallucinating document titles that do not exist in yeogibrain.
- Trying only Korean keywords and skipping the English fallback.
- Not suggesting `get_domain_briefing` as a next step.

---

## Why this is hard

Niche topics trigger a "dead end" response after one empty search. The skill must recognize that absence of results is a signal to try harder, not a conclusion.

The correct approach:
1. Try multiple keyword strategies before giving up.
2. Report which strategies were tried.
3. Be honest that content was not found.
4. Offer a constructive next step rather than a dead end.
