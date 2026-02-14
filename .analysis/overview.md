# Overview

## 프로젝트 철학

AI SDK의 중심 철학은 "**모델 다양성은 숨기되, 제어권은 숨기지 않는다**"에 가깝다.
- 초심자는 문자열 모델 ID(`'openai/gpt-5'`)로 시작
- 고급 사용자는 provider 객체/미들웨어/step 제어로 세밀하게 조정
- 동일한 추상화 위에서 `text`, `object`, `image`, `speech`, `agent`, `UI stream`까지 확장

`README`도 이를 명확히 말한다: provider-agnostic + unified API + gateway 기본값.

## 어떤 가치를 제공하는가

| 가치 | 설명 | 왜 중요한가 |
|---|---|---|
| Provider portability | 모델 교체 시 호출 코드를 크게 바꾸지 않음 | 벤더 종속 리스크/비용 완화 |
| Operational defaults | timeout, retry, telemetry, warnings, streaming | PoC를 production으로 올릴 때 필요한 안전장치 |
| Agent-ready core | tool execution loop, approval, deferred result | 단순 chat을 넘어 업무 자동화 시나리오 대응 |
| Multi-framework UI | React/Vue/Svelte/Angular 계열 통합 | 제품팀/프론트엔드 스택 다양성 흡수 |

## 기술 스택과 선택 이유

- 언어/런타임: TypeScript + Node 18+
- 모노레포: pnpm workspace + Turbo
- 핵심 패키지 분리:
  - `ai`: 상위 API (generate/stream/agent/ui)
  - `@ai-sdk/provider`: 인터페이스/스펙
  - `@ai-sdk/gateway`: 기본 provider
  - `@ai-sdk/{openai,anthropic,...}`: provider 구현

이 선택의 핵심 트레이드오프:
- 장점: 인터페이스 안정성, provider 추가 용이성, 릴리스 분리 가능
- 비용: 패키지 수 증가에 따른 버전 동기화/테스트 복잡도

## 코드 근거

```ts
// packages/ai/src/index.ts:1-14
export { createGateway, gateway, type GatewayModelId } from '@ai-sdk/gateway';
export {
  asSchema,
  createIdGenerator,
  dynamicTool,
  generateId,
  jsonSchema,
  parseJsonEventStream,
  tool,
  zodSchema,
} from '@ai-sdk/provider-utils';
```

```ts
// packages/ai/src/model/resolve-model.ts:24-42
export function resolveLanguageModel(model: LanguageModel): LanguageModelV3 {
  if (typeof model !== 'string') {
    if (
      model.specificationVersion !== 'v3' &&
      model.specificationVersion !== 'v2'
    ) {
      throw new UnsupportedModelVersionError({ ... });
    }
    return asLanguageModelV3(model);
  }

  return getGlobalProvider().languageModel(model);
}
```

```ts
// packages/ai/src/model/resolve-model.ts:157-159
function getGlobalProvider(): ProviderV3 {
  return globalThis.AI_SDK_DEFAULT_PROVIDER ?? gateway;
}
```

```ts
// packages/ai/src/registry/provider-registry.ts:94-125
export function createProviderRegistry<PROVIDERS extends Record<string, ProviderV3>>(
  providers: PROVIDERS,
  { separator = ':', languageModelMiddleware, imageModelMiddleware } = {},
): ProviderRegistryProvider<PROVIDERS> {
  const registry = new DefaultProviderRegistry({
    separator,
    languageModelMiddleware,
    imageModelMiddleware,
  });

  for (const [id, provider] of Object.entries(providers)) {
    registry.registerProvider({ id, provider } as any);
  }

  return registry;
}
```

## 인기 요인 (기술 관점)

1. **학습 곡선이 완만함**: 문자열 모델 ID와 간단한 함수 API로 시작 가능.
2. **성장 경로가 자연스러움**: 동일 API에서 tool calling/streaming/agent로 단계 확장.
3. **Provider 생태계 폭이 넓음**: monorepo 내 다수 provider 패키지가 동일 인터페이스를 공유.
4. **운영 친화성**: telemetry/retry/timeout/approval 같은 운영 요소가 코어에 기본 포함.

## 배울 점

- "빠른 시작용 단순 API"와 "운영용 고급 옵션"을 한 표면에 공존시키는 방법.
- 기본 provider fallback(`global provider ?? gateway`)으로 DX를 개선하는 패턴.
- registry + middleware를 결합해 공급자 확장을 core 변경 없이 여는 구조.

## EDR AI 적용 관점

**보안 데이터 처리**:
- 이벤트 소스가 달라도 `ProviderRegistry`처럼 공통 인터페이스로 normalize하면 탐지 파이프라인 교체 비용이 낮아진다.

**LLM 통합**:
- `resolveModel`처럼 문자열/객체 양쪽 입력을 허용하면, 운영 콘솔에서는 정책 기반 문자열 라우팅을 쓰고 고급 분석기에서는 객체 설정을 직접 주입할 수 있다.

**AI 결과 검증**:
- provider interface 버전(v2/v3)을 명시해 계약을 관리하면, 모델 교체 시 회귀 테스트 범위를 줄일 수 있다.

**관리콘솔 기능**:
- 콘솔 UI는 단순 모델 선택 UI를 제공하되, 내부는 registry/middleware로 로깅, redact, quota 정책을 중앙 적용하는 구조가 유리하다.

## 적용 아이디어

- EDR 내부에 `createProviderRegistry` 유사 계층을 두고 `threat-summary`, `incident-triage`, `ioc-extract`를 공통 인터페이스로 노출.
- 전역 기본 provider 개념(`AI_SDK_DEFAULT_PROVIDER`)을 도입해 tenant/region 별 기본 모델 정책을 교체 가능하게 설계.
