# 🌏 Travel Daily Brief

> Claude Code 스킬 하나로 매일 아침 **여행 사이트 7곳의 주요 정보를 1장 표로 받는** 도구.
> 트립닷컴, 마이리얼트립, 트래블로카, 클룩, KKday, 여기어때, 3hoursahead까지 — 매일 일일이 들어가지 않아도 됨.

---

## 📊 결과 미리보기

매일 `/오늘추천` 한 번이면 이런 표가 자동 생성됩니다:

| # | 광고주 | 캠페인 | 종류 | 핵심 → 추천 이유 | 마감 | 🔗 |
|:--:|---|---|---|---|---|:--:|
| 1 | **KKday** | 싱가포르 디즈니 크루즈 역대급 | 단독 상품 | 호텔+항공권<br>⭐⭐⭐ 1순위 — 큰 단가 + 가족여행 매칭 | ~6/30 | 🔗 |
| 2 | **여기어때** | 인스파이어 브랜드전 | 할인 | 최대 87% 특가<br>⭐⭐ 2순위 — 마감 임박 | ~5/26 | 🔗 |
| 3 | 트래블로카 | 방콕 항공+호텔 | 할인 | 10만원 즉시 할인 | ~5/31 | 🔗 |
| ... | (총 8~12개, 매일 갱신) | | | | | |

→ 클릭하면 해당 사이트로 바로 이동. 궁금한 행사는 직접 들어가 확인 가능.

---

## 🎯 누구에게 유용한가

- **여행 정보 매니아**: 매일 어디서 뭐가 싸진지 한눈에
- **가족여행 계획 중**: 시즌 행사·할인 한 번에 비교
- **어필리에이트 블로거**: 매일 글감 리서치 시간 절약
- **여행사 마케터**: 경쟁사 행사 트렌드 모니터링

---

## 🚀 동작 원리

| 분류 | 사이트 | 점검 방식 |
|---|---|---|
| **WebFetch** (자동) | 트립닷컴 / 마이리얼트립 / 트래블로카 | Claude Code 내장 WebFetch |
| **Chrome MCP** (자동) | 클룩 / KKday / 여기어때 | 봇 차단 우회용 |
| **Chrome MCP** (로그인 SPA) | 3hoursahead | 사용자 브라우저 로그인 세션 활용 |

→ 매일 수동으로 7곳 들어가는 시간 5~10분 → **2~3분**, 입력 4번 → **0번**.

---

## 🛠 셋업 가이드

### 필수 환경
- [Claude Code](https://claude.com/claude-code) 설치 + 활성화
- Chrome 브라우저 + [Claude in Chrome](https://claude.ai/chrome) 익스텐션 (4개 사이트 자동화에 필요)

### 단계

1. **이 repo clone**:
   ```bash
   git clone https://github.com/sangimpark/travel-daily-brief.git
   ```

2. **스킬을 `~/.claude/skills/`에 복사**:
   ```bash
   cp -r skills/cpa-daily-brief ~/.claude/skills/
   ```

3. **광고주 데이터 파일 준비**:
   ```bash
   cd ~/.claude/skills/cpa-daily-brief/data/
   cp sources.json.example sources.json
   ```
   `sources.json`을 본인이 보고 싶은 광고주 사이트로 수정하거나, 그대로 7곳 사용.

4. **첫 발동**:
   - Claude Code 채팅에서 `/오늘추천` 입력
   - 첫 발동 시 Chrome 익스텐션 페어링 + 3hoursahead 로그인 (한 번만)
   - 그 다음부터는 매일 한 번 발동만 하면 자동

---

## 📅 매일 사용 흐름

매일 아침 (Claude Code 열어둔 상태에서):
1. `/오늘추천` 또는 "오늘 추천 알려줘" 입력
2. 1장 표 받기
3. 끝.

표 안의 🔗 링크로 궁금한 사이트 직접 확인.

---

## 🔧 커스터마이즈

`sources.json` 편집해서:
- 광고주 추가/제거 (예: 호텔스닷컴, 아고다 등)
- 자주 보고 싶은 종류 우선순위 변경 (`_sort_order`)
- 보고 싶지 않은 카테고리 자동 제외 (`_exclusions` — 예: 결제수단 결합형, 커미션 상향 변동)

---

## 📂 폴더 구조

```
travel-daily-brief/
├── README.md
├── .gitignore
└── skills/
    └── cpa-daily-brief/
        ├── SKILL.md                       # 스킬 정의 (트리거, 4단계, Output)
        └── data/
            └── sources.json.example       # 7사 광고주 양식 (본인 sources.json으로 복사)
```

> 📌 본인이 사용 중인 `sources.json` (광고주 추가/수정 후), `briefs/{날짜}.md` (매일 누적 결과)는 `.gitignore`로 제외됨 — 개인 데이터 보호.

---

## ⚠️ 주의

- 광고주 사이트 구조가 바뀌면 `sources.json`의 fetch_url 갱신 필요
- 3hoursahead는 사용자 로그인 세션 살아있어야 자동 동작
- 자동 점검 자체에 비용 발생 안 함 (Claude Code 구독료 외)

---

> 🤖 Claude Code · 처음 만든 사람의 사용 사례: CPA 마케팅 블로거 (`parksangim83`)의 매일 글감 자동 수집. 그 외 여행 정보 트래킹 용도로도 활용 가능.
