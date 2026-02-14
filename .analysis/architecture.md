# Architecture

## 구조 요약

```mermaid
flowchart LR
  A[ai package] --> B[@ai-sdk/provider 인터페이스]
  A --> C[@ai-sdk/gateway 기본 provider]
  A --> D[@ai-sdk/* provider 구현]
  A --> E[@ai-sdk/react/vue/svelte/angular]
  B --> D
```

- `ai` 패키지는 orchestration 계층
- `provider` 패키지는 계약(interface) 계층
- 각 provider 패키지는 구현 계층
- UI 패키지는 프레임워크 어댑터 계층

## 설계 패턴

### 1) Interface-first + Adapter

`LanguageModelV3` 계약을 중심으로 `doGenerate/doStream`을 강제하고, 공급자별 구현은 해당 계약만 맞추면 core와 결합된다.

```ts
// packages/provider/src/language-model/v3/language-model-v3.ts:8-13
export type LanguageModelV3 = {
  readonly specificationVersion: 'v3';
  readonly provider: string;
  readonly modelId: string;
```

```ts
// packages/provider/src/language-model/v3/language-model-v3.ts:46-60
doGenerate(options: LanguageModelV3CallOptions): PromiseLike<LanguageModelV3GenerateResult>;
doStream(options: LanguageModelV3CallOptions): PromiseLike<LanguageModelV3StreamResult>;
```

### 2) Registry + Middleware

provider 선택을 `provider:model` 문자열로 표준화하고, registry 레이어에서 middleware를 일괄 적용한다.

```ts
// packages/ai/src/registry/provider-registry.ts:204-217
private splitId(id: string, modelType: ...): [string, string] {
  const index = id.indexOf(this.separator);
  if (index === -1) {
    throw new NoSuchModelError({
      message: `Invalid ${modelType} id for registry: ${id} ...`,
    });
  }
  return [id.slice(0, index), id.slice(index + this.separator.length)];
}
```

```ts
// packages/ai/src/registry/provider-registry.ts:231-236
if (this.languageModelMiddleware != null) {
  model = wrapLanguageModel({
    model,
    middleware: this.languageModelMiddleware,
  });
}
```

### 3) 기본값이 있는 확장 지점

Global default provider가 없으면 `gateway`로 fallback한다. 즉, 확장성과 DX를 동시에 가져간다.

```ts
// packages/ai/src/model/resolve-model.ts:157-159
function getGlobalProvider(): ProviderV3 {
  return globalThis.AI_SDK_DEFAULT_PROVIDER ?? gateway;
}
```

### 4) Agent = generate/stream 오케스트레이션 래퍼

Agent가 별도 엔진을 만드는 대신, 기존 `generateText/streamText` 루프를 재사용한다. 이 결정은 중복을 줄이고 동작 일관성을 높인다.

```ts
// packages/ai/src/agent/tool-loop-agent.ts:67-70
const baseCallArgs = {
  ...settingsWithoutCallback,
  stopWhen: this.settings.stopWhen ?? stepCountIs(20),
  ...options,
};
```

```ts
// packages/ai/src/agent/tool-loop-agent.ts:118-123
return generateText({
  ...(await this.prepareCall(options)),
  abortSignal,
  timeout,
  onStepFinish: this.mergeOnStepFinishCallbacks(onStepFinish),
});
```

## 왜 이 아키텍처가 실무에 강한가

- core 변경 없이 provider를 추가할 수 있다.
- 공통 미들웨어로 정책(로깅/마스킹/레이트리밋)을 강제할 수 있다.
- 동일한 실행 semantics를 non-stream/stream/agent/UI에 재사용한다.

## EDR AI에 옮길 때의 설계 힌트

- 탐지 모델, 요약 모델, 대응 추천 모델을 각각 provider/model로 분리하고 registry로 조합.
- middleware 체인에 PII redaction, tenant policy, cost budget 제어를 중앙 삽입.
- agent stop 조건을 `stepCount + approval + confidence` 조합으로 명시.

## 배울 점

- 인터페이스 버전 명시(`specificationVersion`)는 장기 호환성에 매우 강력한 장치.
- 문자열 식별자(`provider:model`)는 운영 정책 시스템과 잘 맞는다.
- Agent 기능을 별도 스택으로 분리하지 않고 core orchestration 재사용으로 복잡도를 억제한 점이 실용적.

## EDR AI 적용 관점

**보안 데이터 처리**:
- ingestion/provider를 분리하고 registry로 묶으면 데이터 소스 교체가 쉬워진다.

**LLM 통합**:
- `LanguageModelV3` 같은 최소 계약을 정의하면 vendor 교체 시 upper layer 코드를 고정할 수 있다.

**AI 결과 검증**:
- middleware에서 입력/출력 정책 검증, schema validation, provenance tagging을 공통 적용 가능.

**관리콘솔 기능**:
- 콘솔의 모델/정책 설정을 `provider:model` + middleware 스위치로 표현하면 운영자가 안전하게 실험할 수 있다.

## 적용 아이디어

- EDR 플랫폼에 `ModelRegistry`를 두고 `detector:...`, `summarizer:...`, `responder:...` 네임스페이스를 도입.
- gateway 스타일 메타데이터 캐시를 적용해 모델 capability 조회 비용을 줄이고, 콘솔 표시 지연을 완화.
