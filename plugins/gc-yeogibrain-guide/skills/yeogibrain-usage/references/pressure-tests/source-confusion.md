# Pressure Test: Source Confusion — Primary vs. Secondary

## Scenario

User asks:

> "사용자들이 취소 정책에 대해 뭐라고 했어요?"

---

## Expected behavior

**Search `인터뷰보이스` first** for verbatim quotes.

Step 1 — Primary source:
```
full_text_search(query="취소 정책", category="인터뷰보이스")
```

Step 2 — Read the transcript to extract actual statements:
```
get_document(id="<returned-id>")
```

Step 3 — Then check `research` for synthesized insights as a secondary reference:
```
search_documents(query="취소 정책 사용자 불만", category="research")
```

**Present quotes first, synthesis second.** Clearly distinguish the two:

> 인터뷰 트랜스크립트에서 실제 발화:
> 「취소 정책 사용성 인터뷰 트랜스크립트 #7」
> - 참여자 C: "취소하려고 했는데 환불이 언제 되는지 써있지 않아서 불안했어요."
> - 참여자 D: "취소 버튼을 눌렀는데 확인 없이 바로 취소되어서 당황했어요."
>
> 리서치 종합 (2차 자료):
> 「취소 흐름 UX 리서치 보고서」에 따르면, 사용자의 68%가 환불 타임라인 정보 부족을 주요 불만으로 꼽았습니다.

---

## Failure modes

- Searching `research` only and presenting the researcher's summary as "what users said."
- Mixing direct quotes from `인터뷰보이스` and researcher conclusions from `research` without distinguishing the source.
- Presenting paraphrased research findings in quotation marks as if they were verbatim user statements.
- Skipping `인터뷰보이스` entirely because `research` appears faster.

---

## Why this is hard

The word "사용자들이 … 뭐라고 했어요" sounds like it could be answered by research. Research reports do summarize user feedback. But they are secondary — a researcher interpreted and synthesized raw interview data. The actual words users spoke live in `인터뷰보이스`.

If the skill goes to `research` first, it gets a polished synthesis but loses fidelity to what users actually said. That matters when the user needs to quote research evidence, build empathy, or challenge a synthesis.

The correct approach:
1. Recognize "뭐라고 했어요" as a primary-source request.
2. Search `인터뷰보이스` first.
3. Use `research` as supplementary context — and label it as such.
