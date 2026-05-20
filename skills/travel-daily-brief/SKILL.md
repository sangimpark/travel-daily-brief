---
name: travel-daily-brief
description: 사용자가 "여행 사이트 리포트", "오늘의 여행 행사", "여행 사이트 점검", "오늘 여행 추천", "/여행리포트", "/travelbrief" 등을 언급할 때 발동하여, 등록된 여행 사이트 7곳(트립닷컴, 마이리얼트립, 트래블로카, 클룩, KKday, 여기어때, 3hoursahead)의 현재 진행 중인 행사·할인·캠페인 정보를 1장 표로 요약한다.
---

# Travel Daily Brief 🌏

여행 사이트 7곳을 자동 점검 → **1장 표로 행사·할인·캠페인 요약**.
일일이 사이트 들어가지 않아도 한눈에 비교 가능. 사용자 입력 거의 0 — 발동하면 끝.

---

## When to use this skill

자동 발동 트리거:
- "여행 사이트 리포트", "오늘의 여행 행사", "여행 사이트 점검"
- "오늘 여행 추천", "여행 행사 정리"
- "/여행리포트", "/travelbrief"

발동하지 말 것:
- 일반적인 여행 후기·일정 짜기 (이건 다른 스킬)
- 특정 1곳만 점검 (등록된 7곳 종합 점검 스킬)
- 단순 잡담이나 무관한 "여행" 키워드 (예: "여행 가고 싶다…")

---

## How this skill works

이 스킬은 2단계로 흐른다 (총 1~2분, 사용자 입력 0).

### Step 1 · 여행 사이트 7곳 자동 점검

**핵심 동작** (sources.json 있을 때):
1. `data/sources.json` 읽기.
2. **두 묶음으로 처리**:

   **A. `sources` 배열 (WebFetch)** — auto_fetch:true 항목
   - 각 URL을 WebFetch로 병렬 점검
   - `fetch_prompt_hint` 필드를 prompt로 사용해 사이트별 맞춤 추출
   - 등록: 트립닷컴 / 마이리얼트립 / 트래블로카 (해외 호텔·항공·패키지)

   **B. `chrome_fetch` 배열 (Claude in Chrome MCP)** — 봇 차단·로그인 SPA 우회용
   - **사전 셋업** (한 사이클에 1회만):
     1. `mcp__Claude_in_Chrome__list_connected_browsers` → 연결된 브라우저 확인
     2. `mcp__Claude_in_Chrome__select_browser` → 1대 선택
     3. `mcp__Claude_in_Chrome__tabs_context_mcp` (createIfEmpty: true) → MCP 탭 그룹 생성
   - **점검 (priority 순)**:
     - priority:1 (3hoursahead) 먼저 → priority:2 (클룩/KKday) → priority:3 (여기어때)
     - 각 항목: `browser_batch`로 [navigate, get_page_text] 묶어 호출
     - `requires_login:true`인 항목이 빈 페이지(notifications region만) 반환하면 사용자에게:
       > "{{name}} 로그인 세션이 만료된 것 같아요. 브라우저에서 로그인 후 다시 발동해주시면 자동 점검됩니다. 이번 사이클은 그 사이트 스킵하고 진행할게요."
     - `extract_method` 필드 ("get_page_text" 또는 "read_page") 사용. `fallback_extract` 명시되어 있으면 1차 실패 시 보조 사용
     - 각 사이트의 `watch_for` / `notes` 컨텍스트로 답변 정리에 반영

3. **sources.json 없을 때 (첫 사용)**:
   > "오늘 점검할 여행 사이트 URL 5개 정도 알려주세요. 다음부터는 자동으로 점검합니다."
   - 받은 URL 리스트로 `data/sources.json` 생성
   - WebFetch 한 번 시도 → 성공: `sources`로 / 실패(봇 차단): `chrome_fetch`로 자동 분류
   - 로그인 필요 사이트는 `chrome_fetch` + `requires_login: true`

### Step 2 · 1장 표 출력 + 누적 저장

**처리 순서**:
1. **`_exclusions` 필터 적용** (가장 먼저): Step 1에서 모은 모든 캠페인을 `sources.json`의 `_exclusions` 배열로 필터링. `pattern` 정규식이 `match_field`(`title` 또는 `type_or_title`)에 매칭되면 후보에서 제거. 제거된 항목은 메모로도 노출하지 않음.
2. **`_sort_order` 적용**: `sources.json`의 `_sort_order` 룰로 정렬.
   - `type_priority`에 따라 종류별 그룹 정렬 (현재: **단독 상품 > 할인**).
   - 같은 그룹 내에서는 `tiebreaker` 순 (마감 임박 → 단가 큼 → 시즌성).
3. 정렬 1순위 캠페인을 "오늘의 추천" 1개로 표시.
4. 아래 Output format에 맞춰 마크다운 1장 출력 (전체 리스트 + 추천 표시).
5. **누적 저장**: 같은 마크다운 내용을 `~/.claude/skills/travel-daily-brief/data/briefs/{YYYY-MM-DD}.md` 파일로 동시 저장.
   - 같은 날짜 파일이 이미 있으면 덮어쓰기 (하루에 한 번 발동 가정).
   - 파일 헤더에 frontmatter (YAML)로 메타데이터 추가:
     ```yaml
     ---
     date: 2026-05-10
     selected: "#1 KKday 싱가포르 디즈니 크루즈"
     total_candidates: 8
     ---
     ```
   - 누적된 파일들은 시간 흐름에 따른 행사 트렌드 비교에 활용 가능.

---

## Harness rules

### 규칙 1 · sources.json 미등록 차단

**언제 발동**: Step 1에서 `data/sources.json` 파일이 없음 (첫 사용 또는 사용자가 삭제).

**무엇을 한다**:
> "data/sources.json이 없습니다. 점검할 여행 사이트 URL 5~7개 알려주시면 자동 등록하고 이번 점검부터 시작합니다."

### 규칙 2 · Chrome MCP 미연결 차단

**언제 발동**: Step 1의 chrome_fetch 처리 단계에서 `list_connected_browsers` 결과가 빈 배열 (Chrome 익스텐션 미연결 또는 미설치).

**무엇을 한다**:
> "Claude in Chrome 브라우저가 연결 안 되어 있어요. 익스텐션 설치 + Chrome 실행 후 다시 발동해주시면 chrome_fetch 4사도 자동 점검됩니다. 이번 사이클은 sources (WebFetch 3사)만 점검하고 진행할게요."

### 규칙 3 · 로그인 세션 만료 차단

**언제 발동**: chrome_fetch 중 `requires_login:true` 사이트(3hoursahead)가 빈 페이지(notifications region만) 반환.

**무엇을 한다**:
> "{{사이트명}} 로그인 세션이 만료된 것 같아요. 브라우저에서 로그인 후 다시 발동해주시면 자동 점검됩니다. 이번 사이클은 그 사이트만 스킵하고 나머지 진행할게요."

---

## Output format

1장 마크다운 — 정확히 다음 양식으로 출력:

```
# 📅 오늘의 여행 사이트 리포트 — {{YYYY-MM-DD}}

## 🔥 진행 중 프로모션 ({{N}}개)
> 자료원: WebFetch 3사 + Chrome MCP 4사. `_exclusions` 자동 적용 후 표시.

| # | 광고주 | 캠페인 | 종류 | 핵심 → 추천 이유 | 마감 | 🔗 |
|:--:|---|---|---|---|---|:--:|
| 1 | {{광고주명}} | {{캠페인명}} | {{할인/단독 상품}} | {{할인율·금액·혜택 1~2단어}}<br>{{⭐⭐⭐ 1순위 — 이유 1줄}} | {{~M/D}} | [🔗]({{광고주 URL}}) |
| 2 | ... | ... | ... | ... | ... | ... |
| ... | | | | | | |

(표 아래) `📌 **3hoursahead 종합 페이지**: [🔗 3hoursahead 프로모션 목록](https://3hoursahead.com/ko/studio/promotions)` 한 줄 추가.

## 🎯 오늘의 추천 1개
→ **#{{N}} {{캠페인명}}**: {{정렬 1순위 이유 1~2줄}}.
단, 사용자 판단으로 표의 다른 항목 선택 가능.
```

**컬럼 순서 고정**: 광고주 > 캠페인 > 종류 > 핵심 및 추천 이유 > 마감 > 🔗

**원칙**:
- `_exclusions` 매칭된 캠페인은 표에서 완전 제거 (메모도 노출 X)
- AI가 미리 거르지 말 것 — 진행 중 캠페인을 가능한 전부 나열 (보통 8~15개)
- 같은 캠페인이 두 자료원(예: 3hoursahead + 광고주 직접)에 동시 등장하면 1행으로 통합
- **광고주 / 캠페인 컬럼 분리**: 광고주명만 별도 컬럼 (검색·필터 쉽게)
- **마감 컬럼**: 형식 `~M/D` (D-N 표기 없음)
- **핵심 + 추천 이유 한 셀 — 줄바꿈으로 분리**: 형식 `{{핵심}}<br>{{추천 표시 + 이유}}`
  - 1줄: 핵심 (할인율·금액·혜택 1~2단어, 예: "최대 87% 특가", "10만원 즉시 할인", "호텔+항공권")
  - 2줄: 추천 표시 + 이유 (⭐⭐⭐ 1순위 / ⭐⭐ 2순위 / ⭐ 보조 / 추천 없으면 빈 줄)
  - 예: `최대 **87% 특가**<br>**⭐⭐ 2순위** — 큰 할인율 + 마감 임박`
  - 추천 없으면 1줄 핵심만 (예: `쿠알라룸푸르/발리`) 또는 `—`
- **🔗 컬럼**: 광고주 URL 링크. `sources.json`의 `url` 또는 `fetch_url` 필드 사용
- 1순위 행은 광고주 컬럼도 **굵게**, 1·2순위는 핵심(할인율·금액)도 **굵게**

---

## Notes for the skill builder

- v0.3 (2026-05-10) — travel-daily-brief 공개판. CPA 블로거 전용 단계(어제 성과·시간/에너지) 제거. 발동하면 자동 점검 + 표 출력만 하는 일반 도구.
- v0.2 (2026-05-10) — Claude in Chrome MCP 통합. 매일 7사 자동 점검 가능.
- v0.1 (2026-05-07) — 초기 (cpa-daily-brief로 출발).
- 첫 사용 시 광고주 URL 리스트 등록(sources.json 생성)이 필수.
- chrome_fetch는 Chrome 브라우저가 켜져 있고 Claude in Chrome 익스텐션이 연결되어 있어야 동작.

### 다음에 다듬을 지점

- [ ] 매일 자동 발동 (사용자가 수동으로 트리거 안 해도 됨) — scheduled-tasks MCP 활용 가능
- [ ] 텔레그램·이메일 등 외부 알림 push
- [ ] 시간 흐름에 따른 프로모션 트렌드 분석 (briefs 누적 데이터 활용)
- [ ] 사용자 카테고리 가중치 (예: "가족여행" 키워드 포함 캠페인 우선) — _sort_order에 user_keywords 필드 추가

---

> 🌏 travel-daily-brief · v0.3 (2026-05-10)
