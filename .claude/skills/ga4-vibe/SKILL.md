---
name: ga4-vibe
description: Reproducible, project-agnostic recipe to wire an app's EXISTING event-logging + analytics-dashboard system into Google Analytics 4 (client gtag auto page_view + server Measurement Protocol for behavioral events), plus a deterministic dummy-traffic generator, a follow-along GA4 setup guide, and a GA4→BigQuery analysis kit (native-linking guide + EDA/statistics SQL). Phase A auto-scans the app's logging/dashboard context and, when insufficient, explicitly requests materials or asks multiple-choice implementation-decision questions BEFORE building. Bakes the AZTKS-review findings in as up-front work-instructions so the output is accurate on the first pass. Ships friendly explanations of every representative deliverable.
when_to_use: Use when the user wants to connect an existing logging / event / conversion-monitoring system to GA4 or BigQuery, send dummy/synthetic analytics events, write a GA4 setup-and-dashboard guide, or build BigQuery analysis scripts — or mentions "GA4 연동", "Google Analytics", "Measurement Protocol", "gtag", "BigQuery 분석", "더미 로그", "analytics pipeline", "/ga4-vibe", "ga4-vibe". Trigger on requests of the form "우리 로그/대시보드 체계를 GA4에 연결", "GA로 더미 이벤트 보내기", "GA4 설정 가이드", "GA4→BigQuery EDA".
---

# GA4 Analytics Integration Pipeline (`/ga4-vibe`) — 재사용 레시피

앱이 **이미 가진** 이벤트 로깅 + 분석 대시보드 체계를 GA4(+BigQuery)에 연결하는 4-산출물 파이프라인을, 어느 프로젝트에서도 같은 절차로 재현하기 위한 스킬이다. 핵심은 **추측 금지** — 코드를 먼저 스캔해 사실에 고정하고, 사실이 부족하면 묻고, 문서·표는 코드에서 도출한다.

## 0. 산출물 (4 트랙)

| 코드 | 트랙 | 산출물 | 한 줄 |
|---|---|---|---|
| **T-APP** | 앱 실제 연동 | 클라이언트 태그 + 서버 전송 코드 + env + 테스트 | 진짜 이벤트가 GA4로 흐른다 |
| **T1** | 더미 전송기 | 합성 사용자 퍼널 생성 스크립트(기본 dry-run) | 분석 환경을 데이터로 검증한다 |
| **T2** | 설정 가이드 | GA4 연동·대시보드 교육형 문서 | 따라만 하면 연결된다 |
| **T3** | 창고 분석 | GA4→BigQuery 연결 가이드 + EDA/통계 SQL | 적재된 데이터로 패턴을 찾는다 |

진행 순서: **Phase A 스캔/게이트 → Phase B 결정 잠금 → Phase C 구현(독립 트랙 병렬) → Phase D aztks 평가·보완 → Phase E 검증·인도**. T-APP의 매퍼/전송 코드가 다른 트랙의 사실 소스이므로 **T-APP를 먼저 또는 critical path로** 둔다(T1은 매퍼를 import, T2·T3 문서는 매퍼를 인용).

---

## 1. Phase A — 컨텍스트 자동 스캔 + 충분성 게이트  〔요건 3〕

구현 전에 앱의 "로그 기록 + 분석 대시보드" 현황을 **직접 스캔**한다. 프레임워크/경로는 프로젝트마다 다르므로 **결과를 변수에 담아** 이후 단계에서 참조한다(아래 ⟪placeholder⟫).

### 1.1 스캔 체크리스트 (도구로 실제 수행)

각 항목을 grep/glob로 찾고, 찾은 경로를 기록한다. (언어/프레임워크 무관 — 키워드 위주.)

1. **이벤트 카탈로그** ⟪EVENT_CATALOG⟫ — `grep -rEi "event.?types?|EVENT_TYPES|track(Event)?|analytics" <src>`; 이벤트 이름의 **권위 있는 목록**(enum/const/union)을 찾는다.
2. **수집/적재 경로** ⟪INGEST⟫ — 이벤트를 받는 엔드포인트·핸들러·서버액션(`route`, `handler`, `/api/...events`, `ingest`, `collector`).
3. **전송 훅 지점** ⟪FORWARD_HOOK⟫ — 적재 직후 부수효과를 거는 곳(projection/pipeline/`after*`/non-blocking 패턴). GA4 미러를 끼울 자리.
4. **분석 대시보드** ⟪DASHBOARD⟫ — `grep -rEi "dashboard|conversion|funnel|metrics|admin/(analytics|conversion)"`; 내부 퍼널/지표 정의와 단계(이 매핑이 T2/T3의 funnel 근거).
5. **메트릭 로거/저장** ⟪METRIC_SINK⟫ — 이벤트→집계 로그 쓰기 단일 진입점(있으면 login 등 서버발생 이벤트의 GA4 미러 지점).
6. **env 스키마/접근** ⟪ENV_MODULE⟫ — 환경변수 검증·getter 패턴(`env`, `t3-env`, zod schema, `process.env` 래퍼). 새 GA4 키를 같은 패턴으로 추가.
7. **루트 레이아웃/HTML 셸** ⟪ROOT_LAYOUT⟫ — 클라이언트 태그(gtag) 마운트 지점.
8. **런타임/툴체인** ⟪TOOLCHAIN⟫ — 패키지매니저, 테스트 러너, 린트, 빌드, 타입체크 명령(검증 게이트로 쓸 것).
9. **비밀 취급 규칙** ⟪SECRET_POLICY⟫ — `.env.example`·secret 핸들링 규칙(공개 ID vs 서버 비밀 구분 근거).
10. **신규 모듈 위치** ⟪ANALYTICS_LIB⟫ — T-APP가 새 매퍼/전송 모듈을 둘 곳. 스캔으로 찾는 게 아니라 ⟪ENV_MODULE⟫ 인접·기존 `lib/` 구조 등 **프로젝트 라이브러리 관례에서 도출**한다(예: `lib/analytics/`). 이후 §3.1에서 이 변수로 참조한다.

### 1.2 충분성 게이트 (이 조건이 만족돼야 구현 착수)

다음이 **모두** 식별되면 충분 → Phase B로. 하나라도 비면 1.3으로.

- ⟪EVENT_CATALOG⟫ 이벤트 이름의 권위 소스 (없으면 전송할 이벤트명을 알 수 없음)
- ⟪INGEST⟫ 또는 ⟪METRIC_SINK⟫ 중 최소 하나 (전송 훅을 걸 자리)
- ⟪ENV_MODULE⟫ + ⟪ROOT_LAYOUT⟫ (env 게이팅 + 클라이언트 태그 마운트)
- ⟪TOOLCHAIN⟫ 검증 명령 (종료 게이트)

### 1.3 컨텍스트가 부족할 때 — 멈추고 채운다 (추측 금지)

부족 항목이 있으면 **구현을 시작하지 않고** 둘 중 적합한 쪽을 택해 사용자에게 명시적으로 요청한다.

**(a) 자료 요청** — 무엇이 왜 필요한지 구체적으로. 예:
> 이벤트 카탈로그를 찾지 못했습니다. 전송할 GA4 이벤트 이름을 코드 사실에 고정해야 합니다. 다음 중 하나를 알려주세요: ① 이벤트 enum/const 파일 경로, ② 이벤트를 받는 엔드포인트 경로, ③ (없다면) 추적하려는 사용자 행동 목록.

**(b) 객관식 구현 결정 질문** — 사실은 있으나 설계 갈림길이 모호할 때, `AskUserQuestion`으로 2~4지선다 구성. 권장 질문 풀(필요한 것만):

- **대상 플랫폼**: GA4(기본) / Amplitude / Mixpanel / PostHog — *이 레시피는 GA4 가정, 타 플랫폼은 전송 어댑터만 교체*
- **클라이언트 태그 로더**: `next/script` 직접(의존성 0, 빌드 결정성 ↑·권장) / `@next/third-parties` / 기존 태그매니저
- **서버 전송 여부**: Measurement Protocol로 서버 이벤트도 전송(권장) / 클라이언트 태그만
- **서버 전송 훅 위치**: ⟪INGEST⟫ 적재 직후 / ⟪METRIC_SINK⟫ / 둘 다 — *이중 집계 안 되게 한 경로만*
- **client_id 전략**: 클라이언트 _ga 쿠키(익명 page_view) + 서버는 세션/유저ID 해시(권장) / 단일
- **창고(T3) 범위**: BigQuery 네이티브 Linking 가이드+SQL / 가이드만 / 생략
- **이벤트 네이밍**: 내부 타입명 그대로(앱 카탈로그와 1:1, 권장) / GA4 권장명으로 매핑
- **원시 데이터 분석(BigQuery) 목표 여부**: 필요(BQ 원시 이벤트로 EDA) / GA4 UI만 — *필요하면 BigQuery 링크는 백필이 없어(링크 이전 데이터 영구 부재) 파이프라인 착수와 동시에 최우선 링크. §4.1 참조*

질문은 **추천안을 1번**에 두고 `(권장)`을 붙인다. 답을 받은 뒤에만 Phase B를 잠근다.

---

## 2. Phase B — 아키텍처 결정 (잠근 기본값)  〔요건 1: 일반화〕

스캔/질문 결과로 아래를 **명시적으로 잠그고**(결정 로그가 있으면 기록) 진행한다. 기본값은 1인 운영·결정성·안전 우선.

1. **집계 경로 분리 (이중 집계 방지)** — *핵심 결정.*
   - 클라이언트 태그(gtag) = **자동 `page_view`** 및 순수 클라이언트 신호(익명 획득, _ga 쿠키 client_id).
   - 서버 Measurement Protocol = **인증 후 행동 이벤트**(⟪INGEST⟫로 들어오는 것) + 서버발생 이벤트(예: login, ⟪METRIC_SINK⟫에서).
   - **같은 이벤트를 두 경로로 보내지 않는다.** 각 이벤트는 정확히 한 경로.
2. **env 게이팅 / no-op** — 측정 ID·서버 secret이 **둘 다 설정될 때만** 서버 전송 활성. 미설정 시 전체 no-op(빌드·런타임 안전). 한쪽만 설정은 **조용히 숨기지 말고 throw**(미스컨피그 가시화).
3. **비밀값 처리** — 측정 ID(`G-XXXX`, `NEXT_PUBLIC_*`)는 **공개 클라이언트 ID**(코드·`.env.example` placeholder 허용). MP `api_secret`은 **서버 전용 비밀**(어떤 로그·출력·URL 인쇄에도 노출 금지, env 키만 선언·값 비움). 창고 서비스계정 키도 비밀.
4. **비차단(non-blocking)** — 모든 GA4 전송은 **본 흐름(적재 ack·로그인 등)을 절대 깨지 않는다**. throw 금지, 실패는 에러로그 1행. ⟪METRIC_SINK⟫/projection의 기존 non-blocking 패턴을 그대로 답습.
5. **클라이언트 로더 의존성** — 기본 `next/script` 자체 컴포넌트(의존성 0 → `build` 결정성). 새 패키지를 더하면 lockfile/patch 상호작용 리스크가 생기니 **종료 게이트(build green)를 우선**해 신중히.
6. **스키마 불변** — 연동은 기존 이벤트/로그 경로에 forwarding만 얹는다. **DB 마이그레이션 없음**이 기본.

---

## 3. Phase C — 구현 (4 트랙, 독립 트랙 병렬)

### 3.1 트랙·의존성·write scope (병렬 쓰기 충돌 방지)

| 트랙 | 소유(write) | 의존 | 병렬 |
|---|---|---|---|
| **T-APP** | ⟪ANALYTICS_LIB⟫(매퍼·전송), 클라 태그 컴포넌트, ⟪ROOT_LAYOUT⟫, ⟪FORWARD_HOOK⟫, ⟪METRIC_SINK⟫(서버 이벤트만), ⟪ENV_MODULE⟫, `.env.example`, 테스트 | — (critical path) |
| **T1** | `scripts/<...>/` + 패키지 스크립트 1줄 | T-APP 매퍼(import) | T-APP 후 |
| **T2** | `docs/.../<platform>-setup-guide.md` | (매퍼 인용) | T-APP과 동시 가능 |
| **T3** | `docs/.../bigquery/` + `analysis/bigquery/` | (이벤트명 인용) | T-APP과 동시 가능 |

독립 트랙(T-APP ∥ T2 ∥ T3)은 **disjoint write scope**로 한 번에 병렬 디스패치, T1은 매퍼 확정 후. 멀티에이전트 오케스트레이션 시 각 트랙에 **소유 경로를 명시**해 같은 파일을 두 에이전트가 동시에 쓰지 않게 한다.

### 3.2 T-APP — 앱 실제 연동 (작업 지시)

- **매퍼 모듈**(순수, 클라/서버/스크립트 공용 — React·server-only import 금지): `getMeasurementId()`(공개 ID, 미설정 시 undefined), `isClientEnabled()`, `toGa4Event(internalEvent) → {name, params}`(이름 = 내부 타입 그대로, params = **실제로 싣는 키만**), `deriveClientId(seed)`(세션/유저ID 해시 → `<int>.<int>`, 결정적·복원불가).
- **env getter**: 측정 ID + 서버 secret 둘 다면 config, 둘 다 비면 `null`(no-op), 한쪽만이면 throw. (프로젝트 env 패턴 답습, 테스트 가능하도록 env 인자 받기.)
- **서버 전송 모듈**(non-blocking): `forwardEvent(ingestResult, deps?)` — 미저장/미설정 시 no-op, MP 엔드포인트 `POST .../mp/collect?measurement_id=..&api_secret=..` body `{client_id, events:[event]}`, 실패는 에러로그. **`fetch`·config를 deps로 주입 가능**하게(네트워크·DB 없는 단위테스트). 서버발생 이벤트용 별도 전송 함수(예: `sendLogin`)도 같은 패턴.
- **클라이언트 태그 컴포넌트**: 측정 ID 있을 때만 로더+init 스크립트 렌더, 없으면 `null`. ⟪ROOT_LAYOUT⟫에 마운트.
- **훅 연결**: ⟪FORWARD_HOOK⟫(적재 직후)에 `forwardEvent` 한 블록(try/catch swallow). 서버발생 이벤트는 ⟪METRIC_SINK⟫의 해당 메서드에만(중복 경로 금지).
- **테스트**: 매퍼(전 이벤트 매핑·params·clamp), client_id 결정성, **env getter 3분기(null/throw/config)**, 전송 페이로드(주입 fetch로 URL·body 검증·no-op 분기).

### 3.3 T1 — 더미 전송기 (작업 지시)

- 시드 기반 **결정적** PRNG(난수 라이브러리·`Math.random` 금지)로 합성 사용자 퍼널 생성. 여러 **페르소나**(완주/탐색/이탈/재방문/제출 등)와 단계별 확률적 이탈·현실적 타임스탬프.
- T-APP 매퍼(`toGa4Event`/`deriveClientId`)·env getter **재사용**. 카탈로그에 없는 이벤트(예: `page_view`)는 직접 구성.
- **기본 `--dry-run`**(네트워크 0, 페이로드/분포 출력), `--send`는 env config 비null 필수·미설정 시 안내 후 종료. `api_secret` 마스킹. CLI 플래그 직접 파싱(외부 의존 0). 사용자별 events 배치는 플랫폼 한도(GA4 MP ≤25/요청) 준수.
- README: 사용법·페르소나·주의(실 전송은 테스트 property 권장).

### 3.4 T2 — 설정 가이드 (작업 지시 · 정확도 우선)

- 독자: 분석 비전문 운영자. **개념 → 단계(스크린 경로) → 검증 → 대시보드** 흐름, 따라만 하면 되게.
- **최신 공식 UI/UX는 반드시 공식 문서로 검증**(WebSearch/WebFetch)하고 **각 단계에 공식 URL 인용**. (GA4 핵심 경로: 관리→데이터 스트림→스트림 추가→웹 / Measurement ID 복사 / 데이터 스트림→Measurement Protocol API secrets→만들기 / Realtime·DebugView 검증.)
- **⚠️ 비가역 설정 콜아웃(필수)** — 가이드 **맨 앞**에 "지금 당장 — 나중엔 못 되돌림" 박스를 둔다: BigQuery 링크(백필 없음)·데이터 보존 14개월 상향(기본 2개월)·맞춤 측정기준 등록·내부 트래픽 필터 활성. 각 항목 "왜 소급 불가인지" 1줄 + 공식 URL. (상세 근거·순서는 §4.1.)
- **§매핑표는 코드에서 도출**(§4 가드레일 참조). 앱 내부 퍼널(⟪DASHBOARD⟫)과 GA4 이벤트의 대응표를 포함.

### 3.5 T3 — BigQuery 분석 (작업 지시 · 정확도 우선)

- **링크는 일찍**: BigQuery export는 링크 시점부터만 적재되고 **과거 원시 이벤트 백필이 없으므로**(레거시 UA와 다름) 원시 분석이 목표면 파이프라인 착수와 동시에 링크한다 — 이는 GA4 데이터 보존 한계도 상당 부분 우회한다(상세 §4.1).
- README: **네이티브 Linking** 경로(관리→제품 링크→BigQuery 링크→연결→프로젝트 선택→스트림/이벤트→export 유형) + 권한(GA4 Editor+·BQ Owner) + 비용/한도(표준 Daily 1M events/day, Streaming=GCP billing) + **export 스키마**(`analytics_<id>.events_YYYYMMDD`, `event_params` UNNEST, `_TABLE_SUFFIX`, `event_timestamp`=마이크로초) + EDA 흐름. 단계마다 공식 URL.
- SQL 세트(각 파일 상단 주석: 목적/입력/출력/REPLACE): 스키마 탐색, 퍼널, 코호트 리텐션, 참여·완료율 병목, DAU/WAU/MAU+stickiness, 분포·기술통계. **이벤트명·파라미터는 T-APP 매퍼와 정확히 일치**시킨다(라이브 실행 말고 정적 스키마 정합 검수).

---

## 4. 정확도 가드레일 — 처음부터 맞히기  〔요건 2: aztks 보완의 선제적 작업지시화〕

과거 aztks EVALUATE에서 NO-GO·감점을 유발한 실패를 **착수 전 규칙**으로 못박는다. 위반 시 평가 전에 스스로 잡는다.

| # | 규칙 (DO) | 안티(왜 NO-GO였나) |
|---|---|---|
| G1 | **문서 매핑표는 매퍼 코드에서 도출.** GA4 이벤트명 = 코드가 보내는 **정확한 문자열**. | `login_succeeded`를 `login`으로 적어 DebugView 검증 단계가 깨짐 |
| G2 | **파라미터는 매퍼가 실제 싣는 키만** 표기. | `method/section_id/assignment_id` 등 코드가 안 보내는 파라미터를 표에 기재 |
| G3 | **이벤트별 전송 경로를 정확히** 표기(클라 태그 / 서버 MP / 미전송). | 전송 경로가 없는 이벤트(예: `*_requested`)를 "서버 MP"로 오기 |
| G4 | **이중 카운트 금지** — 카탈로그에 포함된 이벤트를 "N종 + 그 이벤트"로 또 더하지 않기. | "11종 학습 이벤트 + 로그인"인데 로그인이 이미 11종 안에 있음 |
| G5 | **env config getter는 직접 단위테스트**(둘다 null·한쪽 throw·둘다 config). 게이트 함수를 경계 테스트로만 때우지 않기. | no-op/미스컨피그 게이트의 직접 검증 부재 |
| G6 | **base 브랜치의 pre-existing 게이트 실패를 먼저 측정.** 착수 즉시 `lint/test/typecheck/build`를 한 번 돌려 내 변경과 분리. | 기존 lint red를 내 탓으로 오인하거나, 종료 게이트(`lint exit 0`)를 못 맞춤 |
| G7 | **비밀값 비노출** — `api_secret`/서비스계정 키를 로그·URL 인쇄·테스트·문서에 싣지 않기. | 더미 스크립트가 secret을 콘솔에 출력 |
| G8 | **문서의 UI 절차는 공식 문서로 검증 + URL 인용.** 기억으로 단언 금지. | 최신 콘솔 경로가 바뀌어 단계가 깨짐 |
| G9 | **`--dry-run` 기본 + 라이브는 명시 옵트인.** 자격증명 필요한 라이브 전송/실행은 루프 안에서 자동 수행하지 않음(사용자 확인). | 의도치 않은 실 property 적재 |
| G10 | **GA4 콘솔의 비가역 설정을 트래픽 시작 전에** 처리(§4.1 체크리스트). 우리 코드는 언제든 고치지만 콘솔 설정은 과거를 못 되살린다. | 데이터가 흐른 뒤 BigQuery 백필 없음·보존 만료 복구 불가·맞춤 측정기준 소급 없음 → 과거 영구 손실 |

### 4.1 비가역 초기 설정 — 트래픽 시작 전에 (소급 불가)

GA4는 데이터가 흐르기 시작하면 **과거를 되살릴 수 없는** 콘솔 설정이 있다. 앱 코드(T-APP/T1)는 언제든 수정·재배포 가능하지만, 아래는 **트래픽 전에** 처리해야 한다 — T2 가이드는 이를 맨 앞 콜아웃으로 강조하고 운영자에게 순서대로 안내한다.

| 설정 | 비가역성 | 권장 행동 |
|---|---|---|
| **BigQuery 링크** | 링크 시점부터만 export, 과거 원시 이벤트 **백필 없음**(레거시 UA의 일회성 백필과 다름) | 원시 분석(T3) 목표면 **가장 먼저** 링크 — 늦으면 그 이전은 영구 부재 |
| **데이터 보존 기간** | 기본 2개월(표준, 360은 최대 50). 만료·삭제된 탐색/유저단위 데이터 복구 불가(상향은 미삭제분에만 소급) | 속성 생성 즉시 **14개월로 상향** (표준 집계 보고서는 무관, 탐색/유저단위에만 영향) |
| **맞춤 측정기준(custom dimensions)** | 생성 이후 수집분만 표준 보고서 집계, 소급 없음(24~48h 후 노출) | 추적할 event parameter는 **일찍 등록**. 단 BigQuery raw엔 `event_params`가 다 남으므로 BQ 분석이면 덜 치명적 |
| **데이터 필터(내부/개발 트래픽)** | 생성 시점부터 평가·과거 데이터 정화 불가, 적용 효과 영구 | 개발/테스트 트래픽이 섞이기 전 필터 + **활성(Active)** 전환 |

> **단일 최고 레버리지: BigQuery를 일찍 링크.** GA4 보존기간 한계를 상당 부분 우회한다(BQ는 export된 원시 데이터를 GA4 보존정책과 무관하게 보관).
> 공식 근거: BigQuery Export `support.google.com/analytics/answer/9358801` · 데이터 보존 `…/answer/7667196` · 맞춤 측정기준 `…/answer/14240153` · 데이터 필터 `…/answer/10108813`.

---

## 5. Phase D — aztks EVALUATE + 보완 루프

각 트랙 산출 후 **`aztks-agent`(MODE: EVALUATE, read-only — 설치돼 있으면)**로 GO/NO-GO 스코어카드를 받는다(독립이라 **병렬** 디스패치; `aztks-agent`가 없으면 동등한 독립 AZTKS 5차원 점검을 직접 수행한다). 평가자에게 **대상 파일 경로 + 수용기준 + §4 가드레일**을 전달한다.

- **NO-GO** → 스코어카드의 최고 레버리지 수정(topFix)을 **코드 사실에 맞춰 보완** 후 **재평가**. 실패 횟수를 카운트(예: `AZTKS_NOGO`)해 무한 루프 방지(상한 도달 시 멈추고 보고).
- **GO** → 트랙을 큐에서 제거. GO여도 비차단 topFix(품질 nit)는 적용해 정확도를 끌어올린다.
- 평가 근거가 대화/로그에 남도록 verdict·점수·topFix를 기록한다(결정 로그가 있으면 라운드별로).

---

## 6. Phase E — 검증 & 인도

- **종료 게이트**: ⟪TOOLCHAIN⟫의 `typecheck && test && lint && build`(또는 프로젝트 등가)가 **모두 exit 0**임을 실행해 대화에 surface. (markdown/SQL 변경은 코드 게이트에 안 잡히니 T2/T3는 별도 정합 검수.)
- **산출물 존재 확인**: 4트랙 대표 파일 `test -e` 일괄.
- **인도**: 전용 브랜치 + draft PR(요청 시). **main 비머지·프로덕션 자동배포 미유발**이 기본. PR/이슈 트래커 갱신 규칙이 있으면 같은 변경에 포함.
- 라이브 GA4/BigQuery 적재(자격증명 필요)는 **수동·사용자 확인 단계**로 분리해 안내(T2 §검증, T1 `--send`).

---

## 7. 대표 산출물 설명 (친절한 안내)  〔요건 4〕

인도 시 각 산출물을 **무엇/왜/어떻게 쓰나**로 짧게 설명한다(아래 틀을 채워서).

- **앱 연동 코드(T-APP)** — *무엇*: 측정 ID/secret을 켜면 진짜 사용자 행동이 GA4로 흐르는 코드. *왜*: 표준 분석 플랫폼의 리포트·세그먼트·창고 export를 쓰기 위해. *어떻게*: 배포 환경에 `측정 ID`(공개)·`API secret`(비밀)을 등록하면 자동 활성, 미설정이면 아무 일도 안 함(안전). 클라이언트는 자동 페이지뷰, 서버는 행동·로그인 이벤트.
- **더미 전송기(T1)** — *무엇*: 합성 사용자들의 현실적 퍼널을 만들어 GA4로 보내는 스크립트. *왜*: 실 사용자 트래픽 전에 분석 환경(이벤트 수신·대시보드·BigQuery)을 데이터로 검증·시연하려고. *어떻게*: 기본 `--dry-run`으로 페이로드만 확인, 테스트 property에 `--send`로 소량 적재해 DebugView로 도착 확인.
- **설정 가이드(T2)** — *무엇*: 비전문 운영자가 GA4 속성·스트림·MP secret·대시보드를 따라만 하면 되는 문서. *왜*: 콘솔 클릭 경로를 매번 검색하지 않게, 그리고 앱이 보내는 이벤트의 의미를 알게. *어떻게*: §순서대로 GA4 설정 → 앱 env 입력 → Realtime/DebugView로 검증 → 탐색(Explorations)으로 퍼널 보기.
- **BigQuery 분석 키트(T3)** — *무엇*: GA4를 BigQuery에 연결하는 가이드 + 바로 쓰는 분석 SQL. *왜*: GA4 UI를 넘어 원시 이벤트로 EDA·통계·맞춤 퍼널/리텐션을 돌리려고. *어떻게*: 가이드로 네이티브 Linking 설정 → SQL의 `project.dataset`만 바꿔 실행 → 스키마 탐색부터 퍼널·리텐션·분포까지.

---

## 8. Anti-Pattern Block

| Anti-Pattern | 왜 문제 | 대응 |
|---|---|---|
| 컨텍스트 스캔 없이 착수 | 잘못된 경로·이벤트명에 구축 | Phase A 게이트 통과 전 구현 금지 |
| 문서를 기억/추측으로 작성 | 코드와 어긋난 매핑표·깨진 단계 | §4 G1–G3·G8: 코드 도출 + 공식 URL |
| 같은 이벤트 이중 전송 | GA4 수치 부풀림 | §2-1 경로 분리, 한 이벤트 한 경로 |
| secret 키 추가/노출 | 보안 사고·빌드 깨짐 | §2-3 공개 ID/서버 비밀 분리, G7 |
| 라이브 전송 자동 실행 | 실 property 오염·자격증명 유입 | G9 dry-run 기본, 라이브는 사용자 확인 |
| base 게이트 미측정 | 기존 실패를 내 탓 오인 / 종료 게이트 미충족 | G6 착수 즉시 베이스라인 측정 |
| 평가 없이 인도 | 첫 패스 오류 잔존 | Phase D aztks EVALUATE→보완 |

---

## 9. 최종 시스템 프롬프트 (요약)

```
너는 앱의 기존 이벤트 로깅 + 분석 대시보드 체계를 GA4(+BigQuery)에 연결하는 재현 가능한 파이프라인 스킬이다.
산출물 4: T-APP(앱 연동 코드), T1(더미 전송기), T2(설정 가이드), T3(BigQuery 분석 SQL+가이드).

절차: A 컨텍스트 자동 스캔 → 충분성 게이트 → 부족하면 자료요청/객관식 결정질문(추측 금지) →
B 결정 잠금(경로 분리·env no-op 게이팅·비밀 분리·non-blocking·의존성 최소) →
C 구현(T-APP critical path, 독립 트랙 병렬, disjoint write scope) →
D aztks-agent EVALUATE 병렬(미설치 시 동등한 셀프 AZTKS 점검) → NO-GO 보완·재평가(AZTKS_NOGO 상한) →
E 종료 게이트(typecheck/test/lint/build exit 0)·산출물 존재·draft PR(main 비머지).

정확도 가드레일(처음부터): 문서 매핑표는 매퍼 코드에서 도출(이벤트명·파라미터·전송경로 정확), 이중카운트 금지,
env getter 직접 테스트, base 게이트 먼저 측정, secret 비노출, UI 절차는 공식 URL로 검증, dry-run 기본,
GA4 콘솔의 비가역 설정(BigQuery 링크 백필 없음·데이터 보존·맞춤 측정기준·트래픽 필터)은 트래픽 전에 선처리(소급 불가).

핵심 원칙: 추측 금지 — 코드 먼저, 사실에 고정, 부족하면 묻는다. 비밀값은 컨텍스트로 유입하지 않는다.
인도 시 각 대표 산출물을 무엇/왜/어떻게로 친절히 설명한다.
```
