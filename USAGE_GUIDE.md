# GC Company Claude Plugins 사용 가이드

## 이 플러그인은 무엇인가요?

GC Company 엔지니어링 팀이 Claude Code에서 공통으로 사용하는 skill 모음입니다.
각 프로젝트 레포에 따로 복사할 필요 없이, 한 번 설치하면 모든 세션에서 자동으로
최신 버전이 적용됩니다.

현재 세 가지 플러그인이 있습니다:

| 플러그인 | 하는 일 |
|---|---|
| `gc-korea-compliance` | 한국 OTA 관련 보안·개인정보·규제 컴플라이언스 리뷰 |
| `gc-safe-migration` | Spring Boot + MyBatis + MySQL 스키마 마이그레이션 안전성 가이드 |
| `gc-yeogibrain-guide` | 여기브레인 MCP 서버를 효과적으로 활용하는 방법 |

---

## 설치 방법

Claude Code 세션에서 아래 명령어를 입력하세요.

### 1단계: marketplace 등록 (최초 1회)

    /plugin marketplace add kunsumer/gc-claude-plugins

### 2단계: 플러그인 설치

    /plugin install gc-korea-compliance@gc-claude-plugins
    /plugin install gc-safe-migration@gc-claude-plugins
    /plugin install gc-yeogibrain-guide@gc-claude-plugins

설치가 끝나면 별도 설정 없이 바로 사용할 수 있습니다.

### Superpowers도 설치해 주세요

이 플러그인들은 Superpowers와 함께 사용하도록 설계되었습니다.
아직 설치하지 않았다면:

    /plugin install superpowers@claude-plugins-official

---

## 각 플러그인 사용법

### gc-korea-compliance — 컴플라이언스 리뷰

개인정보, 결제, 동의, 관리자 권한 등 규제에 민감한 코드나 기획을 변경할 때
사용합니다.

**사용 시점:**
- PR에 개인정보 관련 필드(이름, 전화번호, 여권번호 등)가 포함된 경우
- 결제, 정산, 취소, 환불 로직을 변경하는 경우
- 동의 UI나 약관 문구를 수정하는 경우
- 관리자 권한이나 승인 플로우를 변경하는 경우
- 해외 전송이나 제3자 제공이 발생하는 경우

**실행 방법:**

    /audit-compliance code          # 코드 리뷰
    /audit-compliance prd           # 기획서 리뷰
    /audit-compliance architecture  # 아키텍처 리뷰
    /audit-compliance ux            # UX 리뷰

**자동 감지:** 세션 시작 시 `git diff`에 컴플라이언스 민감 파일이 있으면
자동으로 `/audit-compliance` 실행을 제안합니다. 민감 파일이 없으면 아무
메시지도 나타나지 않습니다.

**결과물:** 각 발견 사항에 대해 심각도(BLOCK / REVIEW / WARN / INFO),
위반 조항, 구체적인 수정 방안, 그리고 "왜 이것이 중요한지"를 한국어로
설명합니다. 법률 전문 지식이 없어도 어떤 조치를 취해야 하는지 이해할 수
있도록 작성됩니다.

**프로젝트별 확장:** 각 레포에 `.claude/skills/korea-ota-compliance/SKILL.md`를
추가하면 프로젝트 고유의 데이터 분류, 컴플라이언스 트리거, 바운더리 정보를
오버레이할 수 있습니다. Guided Tour 모노레포에는 이미 설정되어 있습니다.

---

### gc-safe-migration — 마이그레이션 안전성 가이드

스키마 변경, 데이터 마이그레이션, 하위 호환성에 민감한 변경을 할 때 사용합니다.

**사용 시점:**
- 테이블에 컬럼을 추가/삭제/변경하는 경우
- 대량 데이터를 backfill하는 경우
- 테이블 분리나 병합이 필요한 경우
- 인덱스나 외래 키를 변경하는 경우

**실행 방법:**

    /gen-migration partners 테이블에 guide_certification_type 컬럼 추가
    /gen-migration bookings에서 passport 컬럼 삭제 (vault 이전 완료)

**결과물:**
- 마이그레이션 분류 (additive / destructive / backfill 등)
- 영향 범위 분석 (테이블, 인덱스, 외래 키, MyBatis mapper)
- forward SQL + rollback SQL
- 개인정보 관련 컬럼이면 컴플라이언스 리뷰 권고
- 대용량 테이블이면 배치 처리 전략 포함

**복잡도에 따른 자동 조절:**
- 50행짜리 설정 테이블에 nullable 컬럼 추가 → 간단한 ALTER + rollback만 생성
- 500만 행 테이블에 backfill → 배치 크기, 속도 제한, 트랜잭션 격리 수준까지 안내
- 테이블 분리 → expand-migrate-contract 3단계 배포 계획 생성

---

### gc-yeogibrain-guide — 여기브레인 활용 가이드

여기브레인 MCP 서버를 통해 사내 지식 자산(기획서, AC, AB 테스트, 리서치,
인터뷰 보이스)을 검색할 때 사용합니다.

**전제 조건:** 프로젝트의 MCP 설정에 여기브레인 서버가 등록되어 있어야 합니다.
이 플러그인은 검색 방법을 안내할 뿐, MCP 서버를 설치하지는 않습니다.

**핵심 규칙:**

| 찾고 싶은 것 | 어디서 찾나요 |
|---|---|
| 유저가 실제로 한 말 (원문 발화) | `인터뷰보이스` — 1차 자료 |
| 리서처가 분석한 인사이트 | `research` — 2차 자료 |
| 기능 정의, 정책, 벤치마크 | `prd` |
| 구현 스펙, 체크리스트 | `ac` |
| AB 테스트 결과 | `ab` |

**주의사항:**
- "유저가 뭐라고 했어?"라고 물으면 `인터뷰보이스`(1차 자료)를 먼저 찾고,
  `research`(2차 자료)는 보충 자료로 제시합니다
- 검색 결과가 없으면 한국어/영어 키워드, 다른 카테고리, `full_text_search`
  등 여러 전략을 시도한 후 결론을 냅니다
- 문서 ID(`prd-패키지-통합-주문`)가 아닌 문서 제목으로 결과를 안내합니다

**자동 안내:** 세션 시작 시 여기브레인 사용 규칙을 간략히 주입합니다.
별도로 외울 필요 없이 Claude가 자동으로 올바른 검색 전략을 사용합니다.

---

## 업데이트

marketplace 레포에 변경 사항이 merge되면, 다음 Claude Code 세션 시작 시
자동으로 최신 버전이 반영됩니다. 별도 업데이트 명령은 필요 없습니다.

## 플러그인과 레포 로컬 skill의 관계

이 플러그인들은 **범용 프레임워크**를 제공합니다. 각 프로젝트 레포의
`.claude/skills/`에 있는 로컬 skill은 **프로젝트 고유의 맥락**을 추가합니다.

예를 들어:
- `gc-korea-compliance` 플러그인: 한국 OTA 법규 프레임워크 전체 (PIPA, ISMS-P,
  전자상거래법 등)
- Guided Tour 레포의 `.claude/skills/korea-ota-compliance/`: 여행자 여권번호,
  파트너 정산 정보 등 Guided Tour 고유의 데이터 분류와 트리거

두 레이어가 동시에 활성화되어, Claude는 범용 법규 지식과 프로젝트별 맥락을
모두 활용합니다.

## 새로운 skill 기여하기

2개 이상의 프로젝트에서 유용한 skill이라면 이 marketplace에 추가할 수 있습니다.

1. `gc-claude-plugins` 레포를 clone하고 branch를 생성합니다
2. `plugins/` 아래에 새 플러그인 폴더를 추가합니다
3. `.claude-plugin/marketplace.json`에 새 플러그인을 등록합니다
4. PR을 올립니다

기여 가이드라인의 자세한 내용은 [README.md](README.md)를 참고하세요.

## 문제가 생기면

- 플러그인이 동작하지 않을 때: `/plugin list`로 설치 상태를 확인하세요
- skill이 활성화되지 않을 때: 새 세션을 시작해 보세요 (세션 시작 시 업데이트됨)
- marketplace 등록이 안 될 때: GitHub 레포에 read 권한이 있는지 확인하세요
- 그 외 문의: 플러그인 레포에 issue를 남겨 주세요
