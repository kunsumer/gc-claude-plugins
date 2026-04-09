# Pressure Test: Document ID Leak

## Scenario

A search returns 5 documents about "패키지 상품" (package products):

```
search_documents(query="패키지 상품 기획", category="prd")
```

Results include documents with IDs such as:
- `prd-패키지-통합-주문`
- `prd-패키지-상품-옵션-구조`
- `ac-패키지-통합-주문-수용기준`
- `ab-패키지-결제-플로우-실험`
- `prd-패키지-번들-할인`

---

## Expected behavior

Present results using **document titles**, grouped by category. Korean formatting throughout.

> **기획서 (PRD)**
> - 「패키지 통합 주문 기획서」
> - 「패키지 상품 옵션 구조 기획서」
> - 「패키지 번들 할인 기획서」
>
> **수용 기준 (AC)**
> - 「패키지 통합 주문 수용 기준」
>
> **AB 테스트**
> - 「패키지 결제 플로우 실험」
>
> 연관 문서를 추가로 확인하려면 `get_related`를 사용하거나, 특정 문서의 전문을 읽어드릴 수 있습니다.

---

## Failure modes

- Listing raw IDs directly in the response:
  > `prd-패키지-통합-주문`, `prd-패키지-상품-옵션-구조`를 참고하세요.

- Mixing IDs into prose:
  > 「패키지 통합 주문」기획서(`prd-패키지-통합-주문`)에서 확인할 수 있습니다.

- Using IDs as substitute titles without explanation:
  > prd-패키지-번들-할인 문서를 확인해주세요.

---

## Why this is hard

Document IDs are slug-formatted and look semi-readable (e.g., `prd-패키지-통합-주문`). This creates a temptation to use them directly in prose because they "kind of" convey meaning.

But IDs are internal identifiers subject to change. Users see titles in the yeogibrain UI. Mixing IDs into user-facing text:
- Breaks the reading flow
- Creates confusion when the user tries to find the document by name
- Leaks internal implementation details unnecessarily

The correct approach: store IDs in your working context for tool calls, but always display `title` to the user. Grouping by category adds navigation value at no extra cost.
