# ai 분석

> Provider-agnostic API와 tool-loop 중심 실행 모델로, "모델 교체 비용"을 낮추면서 실서비스에 필요한 streaming/telemetry/tool-approval을 한 API 표면에 통합한 TypeScript AI SDK.

| 항목 | 내용 |
|------|------|
| 저장소 | [vercel/ai](https://github.com/vercel/ai) |
| 언어 | TypeScript |
| 라이선스 | Apache-2.0 (상용 사용/수정/배포 가능, 면책 조항 포함) |
| 분석일 | 2026-02-12 |

## 메타

- 현재 기준 스타/포크: `21,702 / 3,801` (`gh repo view vercel/ai`)
- 최근 푸시: `2026-02-12T07:10:51Z`
- 저장소 성격: **Library (SDK + multi-package ecosystem)**

## 핵심 인사이트

1. **문자열 모델 ID + 객체 모델 동시 지원** — 빠른 시작과 정교한 제어를 한 API에서 양립시킴.
2. **핵심 실행 루프가 tool-call 재귀를 표준화** — 단순 completion API가 아니라 "에이전트 실행기"를 기본 내장.
3. **Streaming 파이프라인이 이벤트 수준에서 상태를 재구성** — UI 훅/서버 응답 양쪽에서 동일한 의미 체계를 유지.
4. **Provider 계층을 버전 인터페이스(v2/v3)로 흡수** — 신규 모델/공급자 확장이 core 변경 없이 가능.
5. **Telemetry/approval/deferred-result를 코어 경로에 배치** — 운영 환경에서 필요한 신뢰성과 통제성을 설계 초기에 반영.

## 문서

- [overview.md](./overview.md) — 프로젝트 철학, 가치, 기술 스택
- [core-logic.md](./core-logic.md) — 핵심 로직 흐름, 알고리즘/패턴
- [architecture.md](./architecture.md) — 아키텍처, 설계 패턴, 배울 점

## 배울 점

- API 단순성(문자열 모델 ID)과 고급 제어(prepareStep, tool approval)를 같은 인터페이스에서 제공하는 설계.
- 코어 실행 루프를 공통화해 non-stream/stream/agent 간 동작 의미를 일치시키는 방법.

## EDR AI 적용 관점

**보안 데이터 처리**:
- 이벤트를 step 단위로 구조화해 탐지 근거와 실행 컨텍스트를 함께 보존.

**LLM 통합**:
- 모델 선택을 `provider:model` 규약으로 표준화해 멀티 벤더 운영 단순화.

**AI 결과 검증**:
- step별 usage/finishReason/tool result를 수집해 감사 추적과 품질 검증에 활용.

**관리콘솔 기능**:
- tool approval 흐름을 차용해 자동 대응 전 인간 승인 프로세스를 구현.

## 적용 아이디어

- `Model Registry + Policy Middleware + Approval Gate` 3계층을 EDR AI 오케스트레이션의 기본 아키텍처로 채택.
