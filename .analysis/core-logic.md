# Core Logic

## 실행 흐름 (non-streaming: `generateText`)

1. 입력 표준화 (`system/prompt/messages`)
2. abort/timeout/retry/telemetry 컨텍스트 구성
3. `do ... while` 스텝 루프 진입
4. 각 스텝에서
   - `prepareStep`으로 step별 모델/툴/옵션 조정
   - provider `doGenerate` 호출
   - tool-call 파싱/검증/복구
   - client tool 실행 + provider-executed deferred tool 추적
   - step 결과 누적
5. stop condition 충족 시 종료 + 최종 output 파싱

```ts
// packages/ai/src/generate-text/generate-text.ts:509-538
do {
  const stepInputMessages = [...initialMessages, ...responseMessages];

  const prepareStepResult = await prepareStep?.({
    model,
    steps,
    stepNumber: steps.length,
    messages: stepInputMessages,
    experimental_context,
  });

  const stepModel = resolveLanguageModel(prepareStepResult?.model ?? model);

  const promptMessages = await convertToLanguageModelPrompt({
    prompt: {
      system: prepareStepResult?.system ?? initialPrompt.system,
      messages: prepareStepResult?.messages ?? stepInputMessages,
    },
    supportedUrls: await stepModel.supportedUrls,
  });
```

```ts
// packages/ai/src/generate-text/generate-text.ts:672-687
const stepToolCalls: TypedToolCall<TOOLS>[] = await Promise.all(
  currentModelResponse.content
    .filter((part): part is LanguageModelV3ToolCall => part.type === 'tool-call')
    .map(toolCall =>
      parseToolCall({
        toolCall,
        tools,
        repairToolCall,
        system,
        messages: stepInputMessages,
      }),
    ),
);
```

```ts
// packages/ai/src/generate-text/generate-text.ts:865-874
} while (
  ((clientToolCalls.length > 0 &&
    clientToolOutputs.length === clientToolCalls.length) ||
    pendingDeferredToolCalls.size > 0) &&
  !(await isStopConditionMet({ stopConditions, steps }))
);
```

## tool-call 파싱/복구 전략

이 프로젝트의 중요한 특징은 "오류를 즉시 실패로 끝내지 않고, **복구 가능한 경로를 먼저 시도**"한다는 점이다.

```ts
// packages/ai/src/generate-text/parse-tool-call.ts:39-79
try {
  return await doParseToolCall({ toolCall, tools });
} catch (error) {
  if (
    repairToolCall == null ||
    !(NoSuchToolError.isInstance(error) || InvalidToolInputError.isInstance(error))
  ) {
    throw error;
  }

  const repairedToolCall = await repairToolCall({ toolCall, tools, system, messages, error });
  if (repairedToolCall == null) {
    throw error;
  }

  return await doParseToolCall({ toolCall: repairedToolCall, tools });
}
```

## streaming 경로의 핵심

`streamText`는 단순 bytes forwarding이 아니라, 이벤트를 재구성해 step 단위 상태를 만든다. 그래서 UI 훅과 서버 응답이 같은 의미를 공유한다.

```ts
// packages/ai/src/generate-text/stream-text.ts:754-774
const eventProcessor = new TransformStream({
  async transform(chunk, controller) {
    controller.enqueue(chunk);

    const { part } = chunk;

    if (
      part.type === 'text-delta' ||
      part.type === 'reasoning-delta' ||
      part.type === 'source' ||
      part.type === 'tool-call' ||
      part.type === 'tool-result'
    ) {
      await onChunk?.({ chunk: part });
    }
  },
});
```

```ts
// packages/ai/src/generate-text/stream-text.ts:912-947
if (part.type === 'finish-step') {
  const stepMessages = await toResponseMessages({ content: recordedContent, tools });

  const currentStepResult: StepResult<TOOLS> = new DefaultStepResult({
    content: recordedContent,
    finishReason: part.finishReason,
    usage: part.usage,
    response: { ...part.response, messages: [...recordedResponseMessages, ...stepMessages] },
  });

  await onStepFinish?.(currentStepResult);
  recordedSteps.push(currentStepResult);
  recordedResponseMessages.push(...stepMessages);
  stepFinish.resolve();
}
```

## EDR AI에 특히 유효한 패턴

- stop condition을 함수로 분리(`stepCountIs`, `hasToolCall`)해 루프 종료 정책을 도메인 정책으로 교체 가능.
- provider-executed deferred tool 추적은 "비동기 분석 작업 완료 대기"(예: 샌드박스 detonation)와 매우 유사.

```ts
// packages/ai/src/generate-text/stop-condition.ts:8-10
export function stepCountIs(stepCount: number): StopCondition<any> {
  return ({ steps }) => steps.length === stepCount;
}
```

## 배울 점

- 다단계 agent 실행에서 "step 누적 + 중간 콜백 + 종료조건 함수화" 조합이 유지보수에 강하다.
- tool-call 오류를 복구 가능 경로로 설계하면 실제 운영 실패율을 크게 줄일 수 있다.
- streaming도 상태 머신 관점으로 설계해야 UI/서버 불일치를 줄인다.

## EDR AI 적용 관점

**보안 데이터 처리**:
- `recordedContent`처럼 이벤트를 의미 단위(step)로 재조립하면 원시 로그 스트림을 탐지 근거 단위로 안정적으로 보존할 수 있다.

**LLM 통합**:
- `prepareStep` 패턴을 적용해 단계별 모델 라우팅(저비용 분류 모델 → 고성능 분석 모델)을 구현할 수 있다.

**AI 결과 검증**:
- `onStepFinish`에서 각 단계의 근거/사용량/finishReason을 저장하면 사후 검증과 audit trail 생성이 쉽다.

**관리콘솔 기능**:
- tool approval 흐름을 그대로 차용해 "자동 대응 전 승인" UX를 만들 수 있다.

## 적용 아이디어

- `stopWhen` 유사 정책 엔진을 두고 `max_steps`, `confidence_threshold`, `human_approval_required`를 조합.
- `repairToolCall` 유사 복구기를 도입해 파서 실패 시 schema-guided 재해석을 수행.
