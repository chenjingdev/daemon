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

- **프로바이더 개조 금지.** Claude Code/Codex는 정책 준수 위해 **터미널 노드로 감싸기만** 함.
- **실시간 노드 생성 안 함.** 자율성은 "새 노드를 만드는 것"이 아니라 **"주어진 노드를 AI가 주체적으로 조합"**하는 데서 나옴.
- **MCP delegate 툴로 서브에이전트 호출.** 프롬프트 오염 없이, 공식 채널로.
- **유저 메모리와 AI 자아 메모리 분리.** 혼초 peer 개념 확장.
- **관찰 가능성 최우선.** 터미널 실시간 가시화, 노드별 I/O 트레이스, 생각 히스토리.

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

## 안전 / 주의

- **과잉 행동 방지** — Attention Trigger threshold, Cooldown 필수.
- **자아 드리프트 모니터링** — 자아 원본 불변, 현재 상태 별도. diff 추적.
- **Approval 노드** — 자율성 높은 Action 앞에 필수.
- **Dry-run + Audit log** — 모든 실행 기록.
- **"자아"는 은유, 구조적 프롬프트일 뿐** — 디버깅 시 혼동 금지.

---

## 기록

- 2026-04-23 seed. `C 경로` 하이브리드 MVP 확정.
