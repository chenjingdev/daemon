# Daemon

> **Designable Autonomy Platform**
>
> 결정론과 자율성 사이에서, **자율성의 수준을 설계 가능하게** 만드는 AI 워크플로우 플랫폼.

그리스어 `daimonion` = 내면의 안내 정령.
Unix `daemon` = 백그라운드 런타임.
두 뜻이 정확히 겹친다.

---

## 왜 만드는가 — 해결할 네 가지 불편

1. **설정 카오스** — 플러그인·스킬·에이전트가 전역 설치 vs 지역 설치 사이에서 길을 잃음. 만든 에이전트 재활용이 어려움.
2. **하네스 강제의 직관성 부족** — 훅/플러그인으로 "강제"를 구현하면 흐름이 안 보임. 시각적으로 고정된 파이프라인이 필요.
3. **기존 하네스들의 블랙박스·파편화** — 시장에 나오는 하네스들은 각각 매력 포인트가 다른데, 전부 쓸 수도 없고 내부 수정도 어려움. 좋은 기능을 **부품처럼 끼워붙이는** 방식이 필요.
4. **Memory + Dynamic Prompting 구현 불가** — 기존 하네스 내부 컨텍스트 관리 로직에 손을 못 대서, 원하는 메모리·프롬프팅 전략을 구현할 수 없음.

---

## 핵심 아키텍처

```
┌─────────────────────────────────────────────────┐
│ AI 메모리 (자아: 성격·가치관·목표·최근 경험)       │
└─────────────────────────────────────────────────┘
                     ↑ 참조
Sense Nodes ─────┐
  • Webhook      │
  • File         │   감각 누적
  • Slack        │
  • Mail         ├──→ Attention Trigger
  • Time         │    (누적량/강도/패턴)
  • Custom       │            ↓
                 │      Thought (사유)
                 │            ↓
                 │      Action Selector
                 │            ↓
                 │      Tool Nodes
                 │      (Claude Code, Codex, bash, HTTP, ...)
                 │            ↓
                 └──(결과도 새 감각으로 순환)
```

### 주요 노드 계열

- **Sense** — 외부 자극을 감각 객체로 수신. 이벤트 스트림 출력 타입.
- **Sensory Buffer** — TTL + decay 있는 감각 저장소.
- **Attention Trigger** — 누적량·강도·패턴·시간창·복합 조건.
- **Self Identity** — AI 자아 블록 로드 (유저 메모리와 분리).
- **Memory Retrieval** — 다이나믹 프롬프팅 (top K 조각 선별).
- **Thought** — 자아 + 감각 + 기억 → 자연어 사유. 액션 결정.
- **Action Selector** — 가용 도구 중 AI가 동적 선택.
- **Terminal / Tool Nodes** — Claude Code, Codex, bash 등 **개조 없이 박스로** 포함.
- **Raw Model** — Anthropic SDK 직접 호출. 완전한 프롬프트 통제.
- **If** — 수동 조건 또는 AI 판정 게이트.
- **Approval** — 자율성 높은 액션 전 사람 승인.

---

## 레이어 프레임

Daemon은 기존 하네스를 **감싸는** 게 아니라 **다른 층위**에 존재한다.

- **Claude Code / Codex = 몸** — 손·반사·도구·샌드박스·파일시스템
- **Raw Model (SDK 직접) = 뇌** — 판단·분류·추론. 몸의 짐(툴/권한/정책) 없음
- **Daemon = 자아** — 기억·감각·의도·조율

세 레이어 공존. 한쪽이 다른 쪽을 대체하지 않고, **노드마다 작업 성격에 맞는 엔진**을 쓴다.

### 엔진 선택 룰

| 작업 성격 | 엔진 | 이유 |
|---|---|---|
| 코드 편집·실행·디버깅 | Claude Code / Codex | 이미 성숙품. 재발명 무의미 |
| 파일시스템·셸 조작 | Claude Code | 권한·샌드박스 이미 안전 |
| 사유·판단·결정 (Thought) | Raw Model | Claude Code의 짐 불필요 |
| 분류·태깅·요약 | Raw Model (Haiku) | 싸고 빠름 |
| 감각 수신·웹훅·HTTP | Daemon 네이티브 | 외부 얇은 인터페이스 |
| 다이나믹 프롬프팅 조립 | Daemon 네이티브 | 핵심 가치 |

---

## 자율성 스펙트럼

같은 캔버스에서 세 모드가 자연스럽게 표현된다.

| 모드 | 특징 | 쓰임 |
|---|---|---|
| **Rigid** | 모든 분기 수동, Thought 없음 | n8n 대체재 |
| **Hybrid** | 결정론적 뼈대 + Thought 한두 개 | Sweet spot |
| **Autonomous** | Sense→Thought→Action Selector 풀루프 | 24시간 자율 동작 |

"길들이기의 강도"를 워크플로우 단위로 다이얼처럼 조절.

---

## 기존 플랫폼과의 경계

| | Daemon | n8n / Zapier | Langflow / Flowise | AutoGPT |
|---|---|---|---|---|
| 결정론적 워크플로우 | ✓ | ✓ | 부분 | ✗ |
| AI Thought 1급 노드 | ✓ | ✗ | 부분 | ✓ |
| 자아·감각·경험 메모리 | ✓ | ✗ | ✗ | ✗ |
| Action Selector | ✓ | ✗ | ✗ | ✓ |
| 다이나믹 프롬프팅 | ✓ | ✗ | ✗ | ✗ |
| 기존 하네스 박스 포함 | ✓ | ✗ (SSH만) | ✗ | ✗ |

**n8n에 SSH 노드 있는데 비슷한 거 되지 않나?**
70%는 된다. 나머지 30%가 다음 네 가지이고, 이건 n8n을 플랫폼 레벨에서 리팩토링해야 가능하다:
Thought 1급, 기억 계층, Action Selector, 실행 직전 프롬프트 재조립 훅.

---

## 핵심 설계 원칙

- **Additive Only.** Daemon은 기존 하네스에 덧붙이기만 하고, 빼거나 바꾸지 않는다. 공식 창구 네 개만 사용:
  - `CLAUDE.md` / 프로바이더별 rules 파일
  - `--append-system-prompt` 같은 설정값
  - MCP 리소스/툴
  - Hook 출력
  - 기존 하네스와 싸우고 싶어지는 순간이 오면 = **그 기능을 Raw Model 노드로 빼라는 신호**.
- **프로바이더 개조 금지.** Claude Code/Codex는 정책 준수 위해 **터미널 노드로 감싸기만** 함.
- **실시간 노드 생성 안 함.** 자율성은 "새 노드를 만드는 것"이 아니라 **"주어진 노드를 AI가 주체적으로 조합"**하는 데서 나옴.
- **MCP delegate 툴로 서브에이전트 호출.** 프롬프트 오염 없이, 공식 채널로.
- **유저 메모리와 AI 자아 메모리 분리.** 혼초 peer 개념 확장.
- **관찰 가능성 최우선.** 터미널 실시간 가시화, 노드별 I/O 트레이스, 생각 히스토리.
- **완전 자체 하네스로 빠지지 않기.** "SDK만 꼽아 Claude Code 전부 재구현"은 함정. 차별점은 위 층위에 있음.

---

## MVP 로드맵 (C 경로 — 하이브리드, 7주)

| 주 | 산출물 | 포트폴리오 |
|---|---|---|
| 1 | React Flow + DAG 실행기 + Terminal 노드 1종 | — |
| 2 | Manual/Cron Trigger + If 노드(수동) + 로그 뷰어 | "비주얼 CLI 오케스트레이터" 영상 |
| 3 | Raw Model 노드(Anthropic SDK) + AI If 노드 | "Claude Code + Codex 체인 + AI 검증" 영상 |
| 4 | Webhook Sense + Sensory Buffer(SQLite) | — |
| 5 | Attention Trigger + Cooldown | — |
| 6 | Thought 노드 + AI 자아 파일 + 최소 다이나믹 프롬프팅 | **"자극→사유→행동" 자율 에이전트 킬러 영상** |
| 7 | 버그·UX·데모 시나리오 마감 | 종합 데모 |

### 초기부터 박아둘 것

- **공통 I/O 스키마** — `data` / `event_stream` / `reasoning` 세 타입 미리 예약. 나중에 Sense 추가 시 re-plumbing 방지.
- **트리거 노드 vs 일반 노드 구분** — Manual Trigger 하나만 지원하더라도 구분은 지금부터.
- **Stateful 노드 프로토콜** — `state: { read(), write() }` 창구 예약.

---

## 스택 잠정

- 프론트: React + React Flow
- 백: Bun + node-pty (터미널) + WebSocket (실시간 스트림)
- 실행 엔진: DAG 토폴로지 + 이벤트 트리거 v2
- 메모리: 혼초 자가호스팅 연동 + SQLite (감각 버퍼)
- 모델: Anthropic SDK (Raw Model 노드), Claude Code (Terminal 노드)
- 배포 형태: 로컬 데스크톱 우선 (Tauri 또는 Electron 검토)

---

## 센스노드 1호 — Webhook

MVP 첫 Sense는 **Webhook**. 이유:

- 구현 난이도 낮음 (HTTP 수신기)
- Slack/GitHub/Linear/Zapier 모두 웹훅으로 밀어줄 수 있어 범용
- "Slack 10개 쌓이면 AI가 반응"을 **가짜 Slack 연동 없이** 시연 가능

---

## Event Model

Daemon은 이벤트를 **Observation(관찰) + Control(제어)** 두 레이어로 다룬다.

### Observation Layer — 패시브, 범용

- 대상: `on_session_start`, `on_turn_start`, `on_turn_end`, `on_message`, `on_tool_use_post`
- 구현: Terminal Node의 stdout/stderr를 **Daemon이 직접 스트림 파싱**. 프로바이더 네이티브 훅 안 씀.
- 예: Claude Code의 `stop_reason === "end_turn"`을 stream-json에서 직접 캐치 → 네이티브 Stop hook과 **동일 시점** 감지
- 장점: **어떤 CLI든 커버** (Claude Code, Codex, bash, curl, 자체 스크립트). 플러그인 생태계의 씨앗

### Control Layer — 액티브, 프로바이더별

- 대상: `on_tool_use_pre` (차단·수정 같은 **개입성 이벤트**)
- 관찰만으로는 불가능 (이미 실행된 후라서)
- 구현: 네이티브 훅 설정을 자동 생성해서 **Daemon 라우터 엔드포인트로 포워드**하게 주입
- Claude Code처럼 1급 훅 있는 프로바이더에만 적용. 다른 프로바이더는 `capability` 플래그로 미지원 명시

### Provider Adapter

프로바이더마다 얇은 어댑터 하나:

```ts
interface ProviderAdapter {
  spawn(prompt, opts): ChildProcess
  parseStream(chunk): ProviderEvent[]
  isTurnEnd(event): boolean
  injectHook?(config): HookConfig   // Control Layer 지원 시
}
```

어댑터 하나 추가 = 새 프로바이더/CLI가 Daemon 생태계에 편입. **부품 교체 방식의 핵심 구현 지점**.

### 네이티브 end 훅 발동 원리 (참고)

- Anthropic API가 훅을 부르는 게 아님. **Claude Code CLI가 로컬에서** 발동시킴
- API 응답의 `stop_reason === "end_turn"` 감지 → CLI가 settings.json의 Stop hook 커맨드 spawn
- 그래서 Daemon도 **같은 신호**(`stop_reason`)를 스트림에서 직접 보면 된다. 네이티브 훅 안 써도 동일 시점 판정 가능

---

## Delegation Routing

Claude Code/Codex 등이 sub-agent를 호출하려 할 때, Daemon이 가로채 **사전 정의된 Terminal Node로 라우팅**한다.

### 작동 방식 (채택안)

1. Daemon MCP 서버가 `daemon__delegate(agent_id, prompt)` 노출
2. 부모 세션의 `--append-system-prompt`에 "Task 대신 daemon__delegate 사용 + 가용 agent 목록" 주입
3. `--disallowed-tools Task`로 네이티브 Task 비활성화
4. Claude가 daemon__delegate 호출 → Daemon이 Agent Card에 매핑된 Terminal Node로 라우팅
5. 자식 Node의 turn end 대기 → 결과를 툴 리턴으로 부모에게 전달

### Agent Card

재사용 단위. 카드 하나 = **이름 + 시스템 프롬프트 + 허용 도구 + 모델/프로바이더 + MCP 설정**.

- UI에서 검색/드래그로 워크플로우에 꽂음
- pre-spawn (빠름·리소스 드는 대신) 또는 on-demand spawn (절약·콜드 스타트)
- 여러 카드가 풀에서 병렬 실행 가능
- "설정 카오스" 페인의 해결책 1순위

### 이 패턴이 여는 것

- **모델 혼합**: 부모 Opus, 자식 Haiku (비용 절감)
- **프로바이더 혼합**: 부모 Claude Code, 자식 Codex (특화)
- **페르소나/도구/권한 각자 설정**
- Daemon UI에서 모든 delegate 흐름 관찰 가능

### 주의

- 컨텍스트 격리 완전 (보안 ↑, 연속성 ↓). 필요 시 Daemon이 부모 요약을 `daemon__delegate` 파라미터에 첨부
- 반환 포맷 정규화 필요 (텍스트/JSON)
- 병렬 delegate 동시성 한계 + rate limit 고려
- 자식 실패는 부모에게 tool error로 전달 → 부모가 자연스럽게 대안 모색

---

## 안전 / 주의

- **과잉 행동 방지** — Attention Trigger threshold, Cooldown 필수.
- **자아 드리프트 모니터링** — 자아 원본 불변, 현재 상태 별도. diff 추적.
- **Approval 노드** — 자율성 높은 Action 앞에 필수.
- **Dry-run + Audit log** — 모든 실행 기록.
- **"자아"는 은유, 구조적 프롬프트일 뿐** — 디버깅 시 혼동 금지.

---

## 기록

- 2026-04-23 seed. `C 경로` 하이브리드 MVP 확정.
- 2026-04-23 확장. 레이어 프레임(몸/뇌/자아) + 엔진 선택 룰, Additive Only 원칙, Event Model(Observation/Control), Provider Adapter, Delegation Routing + Agent Card 섹션 추가.
