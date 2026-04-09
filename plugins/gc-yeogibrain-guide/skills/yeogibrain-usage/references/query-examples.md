<!-- Last reviewed: 2026-04-09 | Legal effective dates: N/A -->

# Query Examples — Effective and Ineffective Patterns

## Effective Patterns

### 1. Find user pain points about hotel cancellation

**Goal**: Find verbatim user statements about hotel cancellation frustrations.

**Step 1 — Search primary source first:**
```
full_text_search(query="호텔 취소 불편 힘들다", category="인터뷰보이스")
```

**Step 2 — Read the full transcript:**
```
get_document(id="<returned-id>")
```

**Step 3 — Discover related research synthesis:**
```
get_related(id="<returned-id>")
```

**Korean output example:**
> 「숙소 취소 UX, 인터뷰 트랜스크립트 #14」에서 실제 발화를 확인했습니다.
> 참여자 A: "취소하려고 했는데 버튼이 어디 있는지 몰라서 고객센터에 전화했어요."
> 참여자 B: "환불이 언제 되는지 안 보여서 불안했어요."
> 연관 리서치: 「숙소 취소 흐름 UX 리서치」(research)도 함께 확인하세요.

---

### 2. What did the checkout AB test show?

**Goal**: Find results and lessons learned from checkout A/B experiments.

**Step 1 — Search AB category:**
```
search_documents(query="결제 체크아웃 전환율", category="ab")
```

**Step 2 — Read the full result document:**
```
get_document(id="<returned-id>")
```

**Korean output example:**
> 「결제 버튼 위치 AB 테스트」 결과:
> - 실험 기간: 2025-08 ~ 2025-09
> - 승자: 변형 B (버튼 하단 고정) — 전환율 +8.3%
> - 레슨런: 스크롤 없이 버튼이 보여야 이탈률이 감소함.

---

### 3. How do competitors handle refund policies?

**Goal**: Find competitor and benchmark analysis for refund/cancellation policies.

**Step 1 — Search PRD for benchmarks (primary location):**
```
search_documents(query="환불 정책 벤치마크 경쟁사", category="prd")
```

**Step 2 — If results are thin, try alternate keywords:**
```
search_documents(query="refund policy benchmark competitor", category="prd")
```

**Step 3 — Check research for gap analysis:**
```
search_documents(query="취소 환불 갭분석", category="research")
```

**Korean output example:**
> 「투어 환불 정책 기획서」의 Benchmarks 섹션:
> - Airbnb Experiences: 체험 24시간 전 전액 환불, 이후 50%
> - Klook: 상품별 차등 정책, 최소 72시간 전
> - 여기투어(자사): 현행 48시간 전 전액 환불 정책은 경쟁사 대비 중간 수준.

---

### 4. Full history of a feature

**Goal**: Trace the complete lifecycle of a feature from PRD through implementation specs and AB results.

**Step 1 — Find the originating PRD:**
```
search_documents(query="찜하기 위시리스트 저장", category="prd")
```

**Step 2 — Follow the document chain:**
```
get_related(id="<prd-id>")
```

**Step 3 — Batch-read all connected documents:**
```
get_documents_batch(ids=["<prd-id>", "<ac-id>", "<ab-id>"])
```

**Present chronologically:**
> 「찜하기 기능 기획서」(PRD, 2024-11) → 「찜하기 수용 기준」(AC, 2024-12) → 「찜하기 버튼 위치 AB 테스트」(AB, 2025-02)
> 결론: 기획 의도(전환 기여)는 AB 테스트에서 부분적으로 검증됨. 모바일 전환율 +5%, 데스크탑은 유의미하지 않음.

---

### 5. Is this idea new?

**Goal**: Check whether a proposed feature idea has been considered or attempted before.

**Step 1 — Search PRD for prior specs:**
```
search_documents(query="<feature concept>", category="prd")
```

**Step 2 — Check if AB tests were run:**
```
search_documents(query="<feature concept>", category="ab")
```

**Step 3 — Check if user research covers this:**
```
search_documents(query="<feature concept>", category="인터뷰보이스")
```

**Synthesize with Korean output:**
> 검색 결과를 종합하면:
> - 기획서: 유사한 아이디어가 2024-06 「홈 개인화 추천 기획서」에 포함되었으나 우선순위 낮음으로 보류됨.
> - AB 테스트: 관련 실험 없음.
> - 인터뷰보이스: 사용자 3명이 "맞춤 추천이 있으면 좋겠다"고 언급함.
> 결론: 아이디어는 기존에 제기된 적 있으나 실험 이력 없음. 신규 추진 가능.

---

## Ineffective Patterns

### 1. Wrong source for user quotes

**Scenario**: User asks "사용자들이 환불 관련해서 뭐라고 했어요?"

**Wrong:**
```
search_documents(query="환불 사용자 의견", category="research")
```
The research category contains researcher-synthesized reports, not verbatim user statements.

**Correct:**
```
full_text_search(query="환불", category="인터뷰보이스")
```
Always go to `인터뷰보이스` first when the user asks for what users *said*.

---

### 2. Overly broad query

**Wrong:**
```
search_documents(query="상품")
```
"상품" matches nearly everything. Results will be too scattered to be useful.

**Correct:**
```
search_documents(query="투어 상품 옵션 구조", category="prd")
```
Use category + specific terms. The more precise the query, the more targeted the results.

---

### 3. Wrong category for use case

**Wrong (looking for AB test results):**
```
search_documents(query="투어 목록 정렬 실험", category="prd")
```
PRD is for feature specs, not experiment results.

**Correct:**
```
search_documents(query="투어 목록 정렬 실험", category="ab")
```
Match the category to the type of document you need.

---

### 4. Giving up after one empty search

**Wrong:**
```
search_documents(query="투어 리뷰 작성 UX", category="research")
# → 결과 없음
# "관련 정보가 없습니다."
```

**Correct:**
```
# Step 1: Try the original query
search_documents(query="투어 리뷰 작성 UX", category="research")
# → 결과 없음

# Step 2: Try Korean synonym
search_documents(query="후기 등록 인터페이스", category="research")

# Step 3: Try English terms
search_documents(query="review writing UX", category="research")

# Step 4: Try a different category
search_documents(query="리뷰 작성", category="prd")

# Step 5: Try full text search
full_text_search(query="리뷰 작성")
```
Try at least 3–4 strategies before reporting that content does not exist.

---

### 5. Not following the document chain

**Wrong:**
```
search_documents(query="투어 예약 플로우", category="prd")
get_document(id="<prd-id>")
# → Stop here and present PRD only
```

**Correct:**
```
search_documents(query="투어 예약 플로우", category="prd")
get_document(id="<prd-id>")
get_related(id="<prd-id>")           # Find AC and AB test documents
get_documents_batch(ids=["<ac-id>", "<ab-id>"])   # Read them together
```
A PRD without its acceptance criteria and AB results is an incomplete picture. Always call `get_related` after finding a key document.
