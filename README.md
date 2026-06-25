![AI Skills for Everyone](author/wildmental-bjpark.png)

# ga4-vibe
> Skill for Cursor, Claude, Codex agents

**Language / 언어:** [한국어](README.md) · [English](README.en.md)

**앱이 이미 가진 이벤트 로깅·분석 대시보드 체계를 Google Analytics 4(+BigQuery)에 연결하는 4-산출물 파이프라인을, 어느 프로젝트에서도 같은 절차로 재현하는 Skill입니다. `/ga4-vibe`로 호출. Cursor, Claude Code, Codex 모두 지원합니다.**

핵심 원칙은 **추측 금지** — 코드를 먼저 스캔해 사실에 고정하고, 사실이 부족하면 묻고, 문서·매핑표는 코드에서 도출합니다. GA4 이벤트명·파라미터·전송 경로가 **실제 코드가 보내는 것과 1:1**로 맞아 첫 패스부터 정확합니다.

---

## 이 스킬이 해결하는 것

### 1. 4개 트랙을 한 절차로 산출한다

| 코드 | 트랙 | 산출물 | 한 줄 |
|---|---|---|---|
| **T-APP** | 앱 실제 연동 | 클라이언트 태그 + 서버 전송 코드 + env + 테스트 | 진짜 이벤트가 GA4로 흐른다 |
| **T1** | 더미 전송기 | 결정적(시드 기반) 합성 사용자 퍼널 스크립트(기본 dry-run) | 분석 환경을 데이터로 검증한다 |
| **T2** | 설정 가이드 | GA4 연동·대시보드 교육형 문서(공식 URL 인용) | 따라만 하면 연결된다 |
| **T3** | 창고 분석 | GA4→BigQuery 연결 가이드 + EDA/통계 SQL | 적재된 데이터로 패턴을 찾는다 |

진행: **Phase A 스캔/게이트 → B 결정 잠금 → C 구현(독립 트랙 병렬) → D aztks 평가·보완 → E 검증·인도.**

### 2. Phase A — 컨텍스트 자동 스캔 + 충분성 게이트 (추측 금지)

구현 전에 앱의 **이벤트 카탈로그·수집 경로·전송 훅·대시보드·env 스키마·루트 레이아웃·툴체인**을 grep/glob로 직접 스캔해 변수에 고정합니다. 충분성 게이트를 통과해야 착수하고, 부족하면 **멈추고** ① 자료를 요청하거나 ② 객관식 구현 결정 질문(`AskUserQuestion`, 권장안 1번)을 던집니다.

### 3. 안전한 잠금 기본값

- **집계 경로 분리(이중 집계 방지)** — 클라이언트 태그 = 자동 `page_view`, 서버 Measurement Protocol = 인증 후 행동 이벤트. 같은 이벤트를 두 경로로 보내지 않습니다.
- **env 게이팅 / no-op** — 측정 ID·서버 secret이 둘 다 설정될 때만 활성, 미설정이면 전체 no-op(빌드·런타임 안전), 한쪽만이면 throw(미스컨피그 가시화).
- **비밀값 분리** — 측정 ID(`G-XXXX`)는 공개, MP `api_secret`·서비스계정 키는 서버 전용 비밀(로그·URL·테스트·문서 비노출).
- **비차단** — 모든 GA4 전송은 본 흐름(적재 ack·로그인)을 절대 깨지 않습니다.

### 4. 정확도 가드레일을 착수 전 작업지시로 (처음부터 맞히기)

과거 AZTKS 리뷰에서 감점을 유발한 실패(문서 매핑표가 코드와 어긋남·이중 카운트·비밀 노출·비가역 콘솔 설정 누락 등)를 **G1–G10 규칙**으로 못박았습니다. 특히 GA4 콘솔의 **비가역 설정**(BigQuery 링크 백필 없음·데이터 보존·맞춤 측정기준·트래픽 필터)은 **트래픽 시작 전**에 처리하도록 가이드 맨 앞 콜아웃으로 강조합니다.

---

## 왜 skill 이 필요한가

| 흔한 시도 | 기대 | 실제 |
|---|---|---|
| 스캔 없이 GA4 연동 착수 | 빠른 연결 | 잘못된 경로·이벤트명에 구축 → 재작업 |
| 매핑표를 기억/추측으로 작성 | 문서 완성 | 코드가 `login_succeeded`인데 문서는 `login` → DebugView 검증 깨짐 |
| 같은 이벤트 클라+서버 둘 다 전송 | 빠짐없이 수집 | GA4 수치 부풀림(이중 카운트) |
| 트래픽부터 흘리고 콘솔 설정 나중에 | 천천히 설정 | BigQuery 백필 없음·보존 만료 → 과거 영구 손실 |
| 라이브 전송 자동 실행 | 바로 검증 | 실 property 오염·자격증명 유입 |

이 스킬은 코드 스캔으로 사실에 고정하고, 매핑을 코드에서 도출하며, 비가역 설정을 앞세우고, dry-run을 기본으로 둬 위 함정을 사전 차단합니다.

---

## 빠른 시작

### 사전 요구사항

- Cursor, Claude Code, 또는 Codex
- 연동 대상이 될, **이미 이벤트 로깅·분석 대시보드를 가진** 앱 코드베이스
- (선택) Phase D 평가 루프에 [`aztks-agent`](https://github.com/wild-mental/aztks-skill) 서브에이전트 — 없으면 동등한 셀프 AZTKS 점검으로 대체

### 스킬 설치

스킬은 **개인** 또는 **프로젝트** 범위 중 하나를 골라 설치합니다. 두 범위 모두 `curl`로 `SKILL.md`만 받습니다. **이 저장소 전체를 작업 repo에 clone하지 마세요.**

| 도구 | 개인 경로 | 프로젝트 경로 |
|------|-----------|---------------|
| Cursor | `~/.cursor/skills/ga4-vibe/` | `.cursor/skills/ga4-vibe/` |
| Claude Code | `~/.claude/skills/ga4-vibe/` | `.claude/skills/ga4-vibe/` |
| Codex | `~/.agents/skills/ga4-vibe/` | `.agents/skills/ga4-vibe/` |

#### 개인 스킬 (권장 — Git repo 무변경)

```bash
# Cursor
mkdir -p ~/.cursor/skills/ga4-vibe
curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.cursor/skills/ga4-vibe/SKILL.md \
  -o ~/.cursor/skills/ga4-vibe/SKILL.md

# Claude Code
mkdir -p ~/.claude/skills/ga4-vibe
curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.claude/skills/ga4-vibe/SKILL.md \
  -o ~/.claude/skills/ga4-vibe/SKILL.md

# Codex
mkdir -p ~/.agents/skills/ga4-vibe
curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.agents/skills/ga4-vibe/SKILL.md \
  -o ~/.agents/skills/ga4-vibe/SKILL.md
```

#### 프로젝트 스킬 (프로젝트 경로에 설치)

```bash
# Cursor
mkdir -p .cursor/skills/ga4-vibe
curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.cursor/skills/ga4-vibe/SKILL.md \
  -o .cursor/skills/ga4-vibe/SKILL.md

# Claude Code
mkdir -p .claude/skills/ga4-vibe
curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.claude/skills/ga4-vibe/SKILL.md \
  -o .claude/skills/ga4-vibe/SKILL.md

# Codex
mkdir -p .agents/skills/ga4-vibe
curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.agents/skills/ga4-vibe/SKILL.md \
  -o .agents/skills/ga4-vibe/SKILL.md
```

#### 설치 후

- **Cursor**: **Reload Window** 한 번
- **Claude Code**: 스킬 수정은 세션 중 live 반영; 세션 시작 후 새 top-level `.claude/skills/`는 재시작 필요할 수 있음
- **Codex**: 스킬이 안 보이면 Codex 재시작

### 사용 방법

연동할 앱 프로젝트에서 호출하면 Agent가 Phase A 스캔부터 시작합니다.

| 도구 | 수동 호출 |
|------|-----------|
| Cursor | `/ga4-vibe` |
| Claude Code | `/ga4-vibe` |
| Codex | `/skills` 또는 `$ga4-vibe` |

**적용되는 요청 예시:**

- "우리 로그/대시보드 체계를 GA4에 연결해줘"
- "GA로 더미 이벤트(합성 퍼널) 보내는 스크립트 만들어줘"
- "운영자가 따라할 GA4 설정·대시보드 가이드 써줘"
- "GA4→BigQuery 연결하고 EDA/리텐션 SQL 만들어줘"

> **Phase D 평가 루프:** 각 트랙 산출 후 `aztks-agent`(MODE: EVALUATE)로 GO/NO-GO를 받아 보완·재평가합니다. `aztks-agent`([aztks-skill](https://github.com/wild-mental/aztks-skill))가 설치돼 있으면 그것을, 없으면 동등한 셀프 AZTKS 5차원 점검을 수행합니다.

---

## 스킬 구성

```
.cursor/skills/ga4-vibe/SKILL.md   # Cursor용
.claude/skills/ga4-vibe/SKILL.md   # Claude Code용
.agents/skills/ga4-vibe/SKILL.md   # Codex용
```

| 섹션(SKILL.md) | 내용 |
|------|------|
| Phase A | 컨텍스트 자동 스캔(10개 변수) + 충분성 게이트 + 자료요청/객관식 결정질문 |
| Phase B | 잠근 기본값(경로 분리·env no-op·비밀 분리·non-blocking·의존성 최소·스키마 불변) |
| Phase C | 4 트랙 구현(T-APP critical path, 독립 트랙 disjoint write scope 병렬) |
| 정확도 가드레일 | G1–G10 + §4.1 비가역 콘솔 설정(트래픽 전 선처리) |
| Phase D | aztks-agent EVALUATE → NO-GO 보완·재평가(미설치 시 셀프 점검) |
| Phase E | 종료 게이트(typecheck/test/lint/build exit 0)·산출물 존재·draft PR |
| 대표 산출물 설명 | 무엇/왜/어떻게 친절 안내 |
| Anti-Pattern Block | 스캔 생략·추측 매핑·이중 전송·secret 노출·라이브 자동 실행 차단 |

---

## 이런 분께 추천합니다

- 이미 자체 로깅·대시보드를 가진 앱을 **표준 분석 플랫폼(GA4/BigQuery)**에 얹고 싶은 팀
- GA4 이벤트명·파라미터가 코드와 어긋나 **DebugView 검증이 깨진** 경험이 있는 분
- 실 트래픽 전에 **합성 데이터로 분석 환경을 검증·시연**하고 싶은 분
- BigQuery 백필 없음 같은 **비가역 설정 함정**을 피하고 싶은 분

---

## 참고

- GA4 Measurement Protocol, BigQuery Export, 데이터 보존·맞춤 측정기준·데이터 필터 — 공식 문서는 SKILL.md의 각 단계 URL 인용을 따른다
- 동반 평가: [aztks-skill](https://github.com/wild-mental/aztks-skill) (`aztks-agent` MODE: EVALUATE)
- Cursor Agent Skills: [Creating Skills](https://docs.cursor.com)
- Claude Code Skills: [Extend Claude with skills](https://docs.anthropic.com/en/docs/claude-code/skills)
- Codex Agent Skills: [Agent Skills](https://developers.openai.com/codex/skills/)

---

## For LLMs

<!-- install-only; operational rules live in SKILL.md — load skill after install -->

```
skill=ga4-vibe
repo=https://github.com/wild-mental/ga4-vibe-skill
install.method=curl-only  # never clone this repo into user's project

flow:
  1. ask scope: "개인 스킬(~/, 모든 프로젝트) vs 프로젝트 스킬(./, 현재 repo만) 중 어디에 설치할까요?"
  2. ask tools: "Cursor / Claude Code / Codex 중 어떤 도구에 설치할까요?" (복수 선택 가능)
  3. run matching curl commands below; do not install both scopes unless user asks
  4. post_install steps; then load SKILL.md — do not infer rules from README

scope.user.paths:
  cursor=~/.cursor/skills/ga4-vibe/SKILL.md
  claude=~/.claude/skills/ga4-vibe/SKILL.md
  codex=~/.agents/skills/ga4-vibe/SKILL.md

scope.project.paths:
  cursor=.cursor/skills/ga4-vibe/SKILL.md
  claude=.claude/skills/ga4-vibe/SKILL.md
  codex=.agents/skills/ga4-vibe/SKILL.md

install.user.cursor=mkdir -p ~/.cursor/skills/ga4-vibe && curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.cursor/skills/ga4-vibe/SKILL.md -o ~/.cursor/skills/ga4-vibe/SKILL.md
install.user.claude=mkdir -p ~/.claude/skills/ga4-vibe && curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.claude/skills/ga4-vibe/SKILL.md -o ~/.claude/skills/ga4-vibe/SKILL.md
install.user.codex=mkdir -p ~/.agents/skills/ga4-vibe && curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.agents/skills/ga4-vibe/SKILL.md -o ~/.agents/skills/ga4-vibe/SKILL.md

install.project.cursor=mkdir -p .cursor/skills/ga4-vibe && curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.cursor/skills/ga4-vibe/SKILL.md -o .cursor/skills/ga4-vibe/SKILL.md
install.project.claude=mkdir -p .claude/skills/ga4-vibe && curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.claude/skills/ga4-vibe/SKILL.md -o .claude/skills/ga4-vibe/SKILL.md
install.project.codex=mkdir -p .agents/skills/ga4-vibe && curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.agents/skills/ga4-vibe/SKILL.md -o .agents/skills/ga4-vibe/SKILL.md

invoke.cursor=/ga4-vibe
invoke.claude=/ga4-vibe
invoke.codex=/skills|$ga4-vibe

post_install.cursor=Reload Window
post_install.claude=live reload; restart if new top-level .claude/skills/ after session start
post_install.codex=restart if skill not detected

optional_companion=aztks-agent (from wild-mental/aztks-skill) for the Phase D EVALUATE loop; degrades to self-review if absent

contract:
  deliverables=[T-APP app integration code, T1 deterministic dummy traffic generator, T2 GA4 setup guide, T3 GA4->BigQuery analysis kit]
  phases=[A scan+sufficiency-gate (no guessing; request materials or AskUserQuestion), B lock decisions, C implement (T-APP critical path; independent tracks parallel/disjoint write scope), D aztks-agent EVALUATE loop, E exit gates + delivery]
  no_guessing=scan code first, pin to facts, ask when insufficient; derive docs/tables from the mapper code (not memory)
  path_split=client gtag=auto page_view; server Measurement Protocol=authenticated behavioral events; one event = exactly one path (no dual count)
  env_gating=server send active only when measurement-id AND server secret both set; both unset=full no-op; one-side-only=throw
  secrets=measurement id (G-XXXX, NEXT_PUBLIC_*) is public; MP api_secret + service-account key are server-only secrets (never logged/printed/tested/documented)
  non_blocking=all GA4 sends never break the host flow (ingest ack / login); failures = one error log line
  accuracy_guardrails=G1..G10 (mapping table derived from mapper code; exact event names/params/transport; no dual count; env getter unit-tested; measure base gate first; secret non-exposure; UI steps verified by official URLs; dry-run default; irreversible console settings before traffic)
  irreversible_before_traffic=[BigQuery link (no backfill), data retention (raise to 14mo), custom dimensions (no retro), data filters (internal/dev traffic)]
  exit_gate=typecheck && test && lint && build all exit 0; artifact existence check; draft PR (no main merge / no prod auto-deploy by default)
```

---

## 라이선스

[MIT License](LICENSE)
