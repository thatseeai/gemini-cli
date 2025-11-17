---
title: "4장. GeminiChat - 채팅 세션 관리"
weight: 04
---

# 4장. GeminiChat - 채팅 세션 관리

## 개요

`GeminiChat`는 개별 채팅 세션을 관리하는 핵심 클래스입니다. 대화 히스토리 유지, API 요청/응답 처리, 스트림 검증, 재시도 로직 등을 담당합니다. 이 클래스는 Google의 GenAI SDK의 Chat 클래스를 기반으로 Gemini CLI의 요구사항에 맞게 커스터마이즈되었습니다.

**파일 위치**: `packages/core/src/core/geminiChat.ts`

## 클래스 구조

### 주요 속성

```typescript
export class GeminiChat {
  private sendPromise: Promise<void> = Promise.resolve();
  private readonly chatRecordingService: ChatRecordingService;
  private lastPromptTokenCount: number;

  constructor(
    private readonly config: Config,
    private readonly generationConfig: GenerateContentConfig = {},
    private history: Content[] = [],
    resumedSessionData?: ResumedSessionData,
  ) {
    validateHistory(history);
    this.chatRecordingService = new ChatRecordingService(config);
    this.chatRecordingService.initialize(resumedSessionData);
    this.lastPromptTokenCount = Math.ceil(
      JSON.stringify(this.history).length / 4,
    );
  }
}
```

### 주요 속성 설명

- **sendPromise**: 메시지 전송의 순차성 보장 (동시 전송 방지)
- **chatRecordingService**: 대화 기록 서비스
- **lastPromptTokenCount**: 마지막 프롬프트의 토큰 수
- **history**: 대화 히스토리 (user와 model의 Content 배열)
- **generationConfig**: API 호출 설정

## 핵심 기능

### 1. 메시지 스트리밍

`sendMessageStream`은 사용자 메시지를 받아 API에 전송하고 응답을 스트리밍합니다.

```typescript
async sendMessageStream(
  model: string,
  params: SendMessageParameters,
  prompt_id: string,
): Promise<AsyncGenerator<StreamEvent>> {
  // 이전 메시지가 완료될 때까지 대기
  await this.sendPromise;

  let streamDoneResolver: () => void;
  const streamDonePromise = new Promise<void>((resolve) => {
    streamDoneResolver = resolve;
  });
  this.sendPromise = streamDonePromise;

  const userContent = createUserContent(params.message);

  // 사용자 입력 기록 (function response 제외)
  if (!isFunctionResponse(userContent)) {
    const userMessage = Array.isArray(params.message)
      ? params.message
      : [params.message];
    const userMessageContent = partListUnionToString(toParts(userMessage));
    this.chatRecordingService.recordMessage({
      model,
      type: 'user',
      content: userMessageContent,
    });
  }

  // 히스토리에 사용자 콘텐츠 추가
  this.history.push(userContent);
  const requestContents = this.getHistory(true); // curated history

  // 재시도 로직을 포함한 제너레이터 반환
  return (async function* () {
    try {
      let lastError: unknown = new Error('Request failed after all retries.');

      // 최대 2회 시도 (초기 + 재시도 1회)
      for (
        let attempt = 0;
        attempt < INVALID_CONTENT_RETRY_OPTIONS.maxAttempts;
        attempt++
      ) {
        try {
          if (attempt > 0) {
            yield { type: StreamEventType.RETRY };
          }

          // 재시도 시 temperature를 1로 설정하여 다른 출력 유도
          const currentParams = { ...params };
          if (attempt > 0) {
            currentParams.config = {
              ...currentParams.config,
              temperature: 1,
            };
          }

          const stream = await self.makeApiCallAndProcessStream(
            model,
            requestContents,
            currentParams,
            prompt_id,
          );

          for await (const chunk of stream) {
            yield { type: StreamEventType.CHUNK, value: chunk };
          }

          lastError = null;
          break; // 성공하면 루프 종료
        } catch (error) {
          lastError = error;
          const isContentError = error instanceof InvalidStreamError;

          if (isContentError) {
            // 재시도 가능한 경우
            if (attempt < INVALID_CONTENT_RETRY_OPTIONS.maxAttempts - 1) {
              logContentRetry(
                self.config,
                new ContentRetryEvent(
                  attempt,
                  (error as InvalidStreamError).type,
                  INVALID_CONTENT_RETRY_OPTIONS.initialDelayMs,
                  model,
                ),
              );
              // 선형 백오프로 대기
              await new Promise((res) =>
                setTimeout(
                  res,
                  INVALID_CONTENT_RETRY_OPTIONS.initialDelayMs * (attempt + 1),
                ),
              );
              continue;
            }
          }
          break;
        }
      }

      if (lastError) {
        if (lastError instanceof InvalidStreamError) {
          logContentRetryFailure(
            self.config,
            new ContentRetryFailureEvent(
              INVALID_CONTENT_RETRY_OPTIONS.maxAttempts,
              (lastError as InvalidStreamError).type,
              model,
            ),
          );
        }
        throw lastError;
      }
    } finally {
      streamDoneResolver!();
    }
  })();
}
```

**주요 특징**:
- **순차성 보장**: `sendPromise`를 사용하여 동시 전송 방지
- **재시도 로직**: Invalid content 에러 시 자동 재시도
- **온도 조정**: 재시도 시 temperature를 1로 설정하여 다른 응답 유도
- **대화 기록**: 사용자 메시지와 모델 응답을 자동으로 기록

### 2. API 호출 및 스트림 처리

```typescript
private async makeApiCallAndProcessStream(
  model: string,
  requestContents: Content[],
  params: SendMessageParameters,
  prompt_id: string,
): Promise<AsyncGenerator<GenerateContentResponse>> {
  const apiCall = () => {
    const modelToUse = getEffectiveModel(
      this.config.isInFallbackMode(),
      model,
    );

    if (
      this.config.getQuotaErrorOccurred() &&
      modelToUse === DEFAULT_GEMINI_FLASH_MODEL
    ) {
      throw new Error(
        'Please submit a new query to continue with the Flash model.',
      );
    }

    return this.config.getContentGenerator().generateContentStream(
      {
        model: modelToUse,
        contents: requestContents,
        config: { ...this.generationConfig, ...params.config },
      },
      prompt_id,
    );
  };

  const onPersistent429Callback = async (
    authType?: string,
    error?: unknown,
  ) => await handleFallback(this.config, model, authType, error);

  // 429 에러 시 폴백 처리를 포함한 재시도
  const streamResponse = await retryWithBackoff(apiCall, {
    onPersistent429: onPersistent429Callback,
    authType: this.config.getContentGeneratorConfig()?.authType,
    retryFetchErrors: this.config.getRetryFetchErrors(),
    signal: params.config?.abortSignal,
  });

  return this.processStreamResponse(model, streamResponse);
}
```

**에러 처리**:
- **429 에러**: 폴백 모델로 자동 전환
- **Fetch 에러**: 설정에 따라 재시도
- **Abort Signal**: 사용자 취소 지원

### 3. 스트림 응답 처리 및 검증

```typescript
private async *processStreamResponse(
  model: string,
  streamResponse: AsyncGenerator<GenerateContentResponse>,
): AsyncGenerator<GenerateContentResponse> {
  const modelResponseParts: Part[] = [];
  let hasToolCall = false;
  let finishReason: FinishReason | undefined;

  // 스트림 청크 처리
  for await (const chunk of streamResponse) {
    const candidateWithReason = chunk?.candidates?.find(
      (candidate) => candidate.finishReason,
    );
    if (candidateWithReason) {
      finishReason = candidateWithReason.finishReason as FinishReason;
    }

    if (isValidResponse(chunk)) {
      const content = chunk.candidates?.[0]?.content;
      if (content?.parts) {
        // Thought 부분 기록
        if (content.parts.some((part) => part.thought)) {
          this.recordThoughtFromContent(content);
        }
        // 함수 호출 감지
        if (content.parts.some((part) => part.functionCall)) {
          hasToolCall = true;
        }

        // Thought를 제외한 파트만 수집
        modelResponseParts.push(
          ...content.parts.filter((part) => !part.thought),
        );
      }
    }

    // 토큰 사용량 기록
    if (chunk.usageMetadata) {
      this.chatRecordingService.recordMessageTokens(chunk.usageMetadata);
      if (chunk.usageMetadata.promptTokenCount !== undefined) {
        this.lastPromptTokenCount = chunk.usageMetadata.promptTokenCount;
      }
    }

    yield chunk; // UI에 즉시 전달
  }

  // Thought 제거 및 텍스트 파트 통합
  const consolidatedParts: Part[] = [];
  for (const part of modelResponseParts) {
    const lastPart = consolidatedParts[consolidatedParts.length - 1];
    if (
      lastPart?.text &&
      isValidNonThoughtTextPart(lastPart) &&
      isValidNonThoughtTextPart(part)
    ) {
      // 연속된 텍스트 파트 병합
      lastPart.text += part.text;
    } else {
      consolidatedParts.push(part);
    }
  }

  const responseText = consolidatedParts
    .filter((part) => part.text)
    .map((part) => part.text)
    .join('')
    .trim();

  // 모델 응답 기록
  if (responseText) {
    this.chatRecordingService.recordMessage({
      model,
      type: 'gemini',
      content: responseText,
    });
  }

  // 스트림 검증
  if (!hasToolCall) {
    if (!finishReason) {
      throw new InvalidStreamError(
        'Model stream ended without a finish reason.',
        'NO_FINISH_REASON',
      );
    }
    if (finishReason === FinishReason.MALFORMED_FUNCTION_CALL) {
      throw new InvalidStreamError(
        'Model stream ended with malformed function call.',
        'MALFORMED_FUNCTION_CALL',
      );
    }
    if (!responseText) {
      throw new InvalidStreamError(
        'Model stream ended with empty response text.',
        'NO_RESPONSE_TEXT',
      );
    }
  }

  // 히스토리에 모델 응답 추가
  this.history.push({ role: 'model', parts: consolidatedParts });
}
```

**스트림 검증 로직**:

스트림이 유효하려면 다음 조건 중 하나를 만족해야 합니다:
1. 도구 호출이 있는 경우
2. 유효한 finish reason과 비어있지 않은 응답 텍스트

검증 실패 시 `InvalidStreamError`를 발생시켜 재시도를 트리거합니다.

### 4. 히스토리 관리

#### 히스토리 조회

```typescript
getHistory(curated: boolean = false): Content[] {
  const history = curated
    ? extractCuratedHistory(this.history)
    : this.history;
  // Deep copy로 외부 변경 방지
  return structuredClone(history);
}
```

**두 가지 히스토리 타입**:
- **Comprehensive History** (포괄적 히스토리): 모든 턴 포함 (유효하지 않은 응답 포함)
- **Curated History** (선별된 히스토리): 유효한 턴만 포함 (API 요청 시 사용)

#### 히스토리 선별 로직

```typescript
function extractCuratedHistory(comprehensiveHistory: Content[]): Content[] {
  if (comprehensiveHistory === undefined || comprehensiveHistory.length === 0) {
    return [];
  }
  const curatedHistory: Content[] = [];
  const length = comprehensiveHistory.length;
  let i = 0;

  while (i < length) {
    if (comprehensiveHistory[i].role === 'user') {
      curatedHistory.push(comprehensiveHistory[i]);
      i++;
    } else {
      const modelOutput: Content[] = [];
      let isValid = true;

      while (i < length && comprehensiveHistory[i].role === 'model') {
        modelOutput.push(comprehensiveHistory[i]);
        if (isValid && !isValidContent(comprehensiveHistory[i])) {
          isValid = false;
        }
        i++;
      }

      // 유효한 모델 출력만 추가
      if (isValid) {
        curatedHistory.push(...modelOutput);
      }
    }
  }

  return curatedHistory;
}
```

**유효성 검증**:

```typescript
function isValidContent(content: Content): boolean {
  if (content.parts === undefined || content.parts.length === 0) {
    return false;
  }
  for (const part of content.parts) {
    if (part === undefined || Object.keys(part).length === 0) {
      return false;
    }
    // Thought가 아닌 빈 텍스트는 무효
    if (!part.thought && part.text !== undefined && part.text === '') {
      return false;
    }
  }
  return true;
}
```

### 5. 도구 호출 기록

```typescript
recordCompletedToolCalls(
  model: string,
  toolCalls: CompletedToolCall[],
): void {
  const toolCallRecords = toolCalls.map((call) => {
    const resultDisplayRaw = call.response?.resultDisplay;
    const resultDisplay =
      typeof resultDisplayRaw === 'string' ? resultDisplayRaw : undefined;

    return {
      id: call.request.callId,
      name: call.request.name,
      args: call.request.args,
      result: call.response?.responseParts || null,
      status: call.status as 'error' | 'success' | 'cancelled',
      timestamp: new Date().toISOString(),
      resultDisplay,
    };
  });

  this.chatRecordingService.recordToolCalls(model, toolCallRecords);
}
```

### 6. Thought 처리

Gemini 2.5 모델은 "thinking" 기능을 지원하여 응답 전에 사고 과정을 표시할 수 있습니다.

```typescript
private recordThoughtFromContent(content: Content): void {
  if (!content.parts || content.parts.length === 0) {
    return;
  }

  const thoughtPart = content.parts[0];
  if (thoughtPart.text) {
    // **주제**와 설명 추출
    const rawText = thoughtPart.text;
    const subjectStringMatches = rawText.match(/\*\*(.*?)\*\*/s);
    const subject = subjectStringMatches
      ? subjectStringMatches[1].trim()
      : '';
    const description = rawText.replace(/\*\*(.*?)\*\*/s, '').trim();

    this.chatRecordingService.recordThought({
      subject,
      description,
    });
  }
}
```

## 에러 처리

### InvalidStreamError

```typescript
export class InvalidStreamError extends Error {
  readonly type:
    | 'NO_FINISH_REASON'
    | 'NO_RESPONSE_TEXT'
    | 'MALFORMED_FUNCTION_CALL';

  constructor(
    message: string,
    type: 'NO_FINISH_REASON' | 'NO_RESPONSE_TEXT' | 'MALFORMED_FUNCTION_CALL',
  ) {
    super(message);
    this.name = 'InvalidStreamError';
    this.type = type;
  }
}
```

**에러 타입**:
- `NO_FINISH_REASON`: 스트림이 finish reason 없이 종료
- `NO_RESPONSE_TEXT`: 응답 텍스트가 비어있음
- `MALFORMED_FUNCTION_CALL`: 함수 호출 형식 오류

## 주요 메서드 요약

| 메서드 | 설명 |
|-------|------|
| `sendMessageStream()` | 메시지 전송 및 응답 스트리밍 |
| `getHistory()` | 히스토리 조회 (curated/comprehensive) |
| `addHistory()` | 히스토리에 콘텐츠 추가 |
| `setHistory()` | 히스토리 설정 |
| `clearHistory()` | 히스토리 초기화 |
| `recordCompletedToolCalls()` | 완료된 도구 호출 기록 |
| `stripThoughtsFromHistory()` | 히스토리에서 Thought 제거 |

## 다음 장에서는

5장에서는 Turn 클래스를 살펴보며, 이벤트 스트림 파싱과 도구 호출 요청 처리를 알아보겠습니다.
