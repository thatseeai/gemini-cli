---
title: "5장. Turn - 턴 기반 대화 처리"
weight: 05
---

## 개요

`Turn` 클래스는 에이전트의 단일 턴(대화 라운드)을 관리합니다. API 응답을 파싱하여 다양한 이벤트로 변환하고, 도구 호출 요청을 처리하며, 인용 정보를 수집합니다. 이 클래스는 Gemini CLI의 에이전틱 루프(agentic loop)의 핵심입니다.

**파일 위치**: `packages/core/src/core/turn.ts`

## 클래스 구조

### 주요 속성

```typescript
export class Turn {
  readonly pendingToolCalls: ToolCallRequestInfo[] = [];
  private debugResponses: GenerateContentResponse[] = [];
  private pendingCitations = new Set<string>();
  finishReason: FinishReason | undefined = undefined;

  constructor(
    private readonly chat: GeminiChat,
    private readonly prompt_id: string,
  ) {}
}
```

### 주요 속성 설명

- **pendingToolCalls**: 실행 대기 중인 도구 호출 목록
- **debugResponses**: 디버깅을 위한 응답 기록
- **pendingCitations**: 아직 표시하지 않은 인용 정보
- **finishReason**: 턴 종료 이유
- **prompt_id**: 현재 프롬프트 ID (시퀀스 추적용)

## 핵심 기능

### 1. Turn 실행 (run)

`run` 메서드는 Turn의 핵심으로, API 응답 스트림을 받아 이벤트로 변환합니다.

```typescript
async *run(
  model: string,
  req: PartListUnion,
  signal: AbortSignal,
): AsyncGenerator<ServerGeminiStreamEvent> {
  try {
    const responseStream = await this.chat.sendMessageStream(
      model,
      {
        message: req,
        config: {
          abortSignal: signal,
        },
      },
      this.prompt_id,
    );

    for await (const streamEvent of responseStream) {
      // 사용자 취소 체크
      if (signal?.aborted) {
        yield { type: GeminiEventType.UserCancelled };
        return;
      }

      // RETRY 이벤트 처리
      if (streamEvent.type === 'retry') {
        yield { type: GeminiEventType.Retry };
        continue;
      }

      const resp = streamEvent.value as GenerateContentResponse;
      if (!resp) continue;

      this.debugResponses.push(resp);
      const traceId = resp.responseId;

      // 1. Thought 처리
      const thoughtPart = resp.candidates?.[0]?.content?.parts?.[0];
      if (thoughtPart?.thought) {
        const thought = parseThought(thoughtPart.text ?? '');
        yield {
          type: GeminiEventType.Thought,
          value: thought,
          traceId,
        };
        continue;
      }

      // 2. 텍스트 콘텐츠 처리
      const text = getResponseText(resp);
      if (text) {
        yield { type: GeminiEventType.Content, value: text, traceId };
      }

      // 3. 함수 호출 처리
      const functionCalls = resp.functionCalls ?? [];
      for (const fnCall of functionCalls) {
        const event = this.handlePendingFunctionCall(fnCall);
        if (event) {
          yield event;
        }
      }

      // 4. 인용 정보 수집
      for (const citation of getCitations(resp)) {
        this.pendingCitations.add(citation);
      }

      // 5. 종료 이유 처리
      const finishReason = resp.candidates?.[0]?.finishReason;
      if (finishReason) {
        // 인용 정보 표시
        if (this.pendingCitations.size > 0) {
          yield {
            type: GeminiEventType.Citation,
            value: `Citations:\n${[...this.pendingCitations].sort().join('\n')}`,
          };
          this.pendingCitations.clear();
        }

        this.finishReason = finishReason;
        yield {
          type: GeminiEventType.Finished,
          value: {
            reason: finishReason,
            usageMetadata: resp.usageMetadata,
          },
        };
      }
    }
  } catch (e) {
    // 에러 처리
    if (signal.aborted) {
      yield { type: GeminiEventType.UserCancelled };
      return;
    }

    if (e instanceof InvalidStreamError) {
      yield { type: GeminiEventType.InvalidStream };
      return;
    }

    const error = toFriendlyError(e);
    if (error instanceof UnauthorizedError) {
      throw error;
    }

    const contextForReport = [
      ...this.chat.getHistory(/*curated*/ true),
      createUserContent(req),
    ];
    await reportError(
      error,
      'Error when talking to Gemini API',
      contextForReport,
      'Turn.run-sendMessageStream',
    );

    const status =
      typeof error === 'object' &&
      error !== null &&
      'status' in error &&
      typeof (error as { status: unknown }).status === 'number'
        ? (error as { status: number }).status
        : undefined;

    const structuredError: StructuredError = {
      message: getErrorMessage(error),
      status,
    };

    await this.chat.maybeIncludeSchemaDepthContext(structuredError);
    yield { type: GeminiEventType.Error, value: { error: structuredError } };
    return;
  }
}
```

### 2. 함수 호출 처리

```typescript
private handlePendingFunctionCall(
  fnCall: FunctionCall,
): ServerGeminiStreamEvent | null {
  const callId =
    fnCall.id ??
    `${fnCall.name}-${Date.now()}-${Math.random().toString(16).slice(2)}`;
  const name = fnCall.name || 'undefined_tool_name';
  const args = (fnCall.args || {}) as Record<string, unknown>;

  const toolCallRequest: ToolCallRequestInfo = {
    callId,
    name,
    args,
    isClientInitiated: false,
    prompt_id: this.prompt_id,
  };

  this.pendingToolCalls.push(toolCallRequest);

  return { type: GeminiEventType.ToolCallRequest, value: toolCallRequest };
}
```

**함수 호출 정보 생성**:
1. Call ID 생성 (API 제공 또는 자동 생성)
2. 함수 이름 및 인자 추출
3. `ToolCallRequestInfo` 객체 생성
4. `pendingToolCalls` 목록에 추가
5. `ToolCallRequest` 이벤트 생성

## 이벤트 타입

### GeminiEventType Enum

```typescript
export enum GeminiEventType {
  Content = 'content',                    // 텍스트 콘텐츠
  ToolCallRequest = 'tool_call_request',  // 도구 호출 요청
  ToolCallResponse = 'tool_call_response', // 도구 실행 결과
  ToolCallConfirmation = 'tool_call_confirmation', // 도구 확인 필요
  UserCancelled = 'user_cancelled',       // 사용자 취소
  Error = 'error',                        // 에러 발생
  ChatCompressed = 'chat_compressed',     // 채팅 압축 완료
  Thought = 'thought',                    // 사고 과정
  MaxSessionTurns = 'max_session_turns',  // 최대 턴 수 도달
  Finished = 'finished',                  // 턴 완료
  LoopDetected = 'loop_detected',         // 무한 루프 감지
  Citation = 'citation',                  // 인용 정보
  Retry = 'retry',                        // 재시도
  ContextWindowWillOverflow = 'context_window_will_overflow', // 컨텍스트 초과 예상
  InvalidStream = 'invalid_stream',       // 유효하지 않은 스트림
  ModelInfo = 'model_info',               // 모델 정보
}
```

### 이벤트 타입 정의

```typescript
export type ServerGeminiContentEvent = {
  type: GeminiEventType.Content;
  value: string;
  traceId?: string;
};

export type ServerGeminiThoughtEvent = {
  type: GeminiEventType.Thought;
  value: ThoughtSummary;
  traceId?: string;
};

export type ServerGeminiToolCallRequestEvent = {
  type: GeminiEventType.ToolCallRequest;
  value: ToolCallRequestInfo;
};

export type ServerGeminiFinishedEvent = {
  type: GeminiEventType.Finished;
  value: GeminiFinishedEventValue;
};

export type ServerGeminiErrorEvent = {
  type: GeminiEventType.Error;
  value: GeminiErrorEventValue;
};

// ... 기타 이벤트 타입

export type ServerGeminiStreamEvent =
  | ServerGeminiContentEvent
  | ServerGeminiThoughtEvent
  | ServerGeminiToolCallRequestEvent
  | ServerGeminiFinishedEvent
  | ServerGeminiErrorEvent
  // ...
```

## 인용 정보 처리

### 인용 추출

```typescript
function getCitations(resp: GenerateContentResponse): string[] {
  return (resp.candidates?.[0]?.citationMetadata?.citations ?? [])
    .filter((citation) => citation.uri !== undefined)
    .map((citation) => {
      if (citation.title) {
        return `(${citation.title}) ${citation.uri}`;
      }
      return citation.uri!;
    });
}
```

### 인용 표시

```typescript
// Turn.run() 내부
if (finishReason) {
  if (this.pendingCitations.size > 0) {
    yield {
      type: GeminiEventType.Citation,
      value: `Citations:\n${[...this.pendingCitations].sort().join('\n')}`,
    };
    this.pendingCitations.clear();
  }
  // ...
}
```

**인용 처리 흐름**:
1. 각 청크에서 인용 정보 추출
2. `pendingCitations` Set에 누적
3. 턴 종료 시 모든 인용을 정렬하여 한 번에 표시

## Thought 처리

### ThoughtSummary 타입

```typescript
export type ThoughtSummary = {
  subject: string;      // 주제 (**주제** 형식)
  description: string;  // 설명
};
```

### Thought 파싱

**파일**: `packages/core/src/utils/thoughtUtils.ts`

```typescript
export function parseThought(text: string): ThoughtSummary {
  const rawText = text;
  const subjectStringMatches = rawText.match(/\*\*(.*?)\*\*/s);
  const subject = subjectStringMatches
    ? subjectStringMatches[1].trim()
    : '';
  const description = rawText.replace(/\*\*(.*?)\*\*/s, '').trim();

  return { subject, description };
}
```

**Thought 형식**:
```
**주제**
설명 내용...
```

## 에러 처리

### 에러 타입별 처리

```typescript
try {
  // Turn 실행
} catch (e) {
  // 1. 사용자 취소
  if (signal.aborted) {
    yield { type: GeminiEventType.UserCancelled };
    return;
  }

  // 2. 유효하지 않은 스트림
  if (e instanceof InvalidStreamError) {
    yield { type: GeminiEventType.InvalidStream };
    return;
  }

  // 3. 인증 오류 (재발생)
  const error = toFriendlyError(e);
  if (error instanceof UnauthorizedError) {
    throw error;
  }

  // 4. 일반 오류
  const structuredError: StructuredError = {
    message: getErrorMessage(error),
    status: /* 상태 코드 추출 */,
  };

  await this.chat.maybeIncludeSchemaDepthContext(structuredError);
  yield { type: GeminiEventType.Error, value: { error: structuredError } };
}
```

### Schema Depth 에러 처리

스키마 깊이 제한을 초과하는 경우, 순환 참조가 있는 도구를 안내합니다.

```typescript
async maybeIncludeSchemaDepthContext(error: StructuredError): Promise<void> {
  if (
    isSchemaDepthError(error.message) ||
    isInvalidArgumentError(error.message)
  ) {
    const tools = this.config.getToolRegistry().getAllTools();
    const cyclicSchemaTools: string[] = [];

    for (const tool of tools) {
      if (
        (tool.schema.parametersJsonSchema &&
          hasCycleInSchema(tool.schema.parametersJsonSchema)) ||
        (tool.schema.parameters && hasCycleInSchema(tool.schema.parameters))
      ) {
        cyclicSchemaTools.push(tool.displayName);
      }
    }

    if (cyclicSchemaTools.length > 0) {
      const extraDetails =
        `\n\nThis error was probably caused by cyclic schema references in one of the following tools, try disabling them with excludeTools:\n\n - ` +
        cyclicSchemaTools.join(`\n - `) +
        `\n`;
      error.message += extraDetails;
    }
  }
}
```

## 데이터 구조

### ToolCallRequestInfo

```typescript
export interface ToolCallRequestInfo {
  callId: string;
  name: string;
  args: Record<string, unknown>;
  isClientInitiated: boolean;
  prompt_id: string;
}
```

### ToolCallResponseInfo

```typescript
export interface ToolCallResponseInfo {
  callId: string;
  responseParts: Part[];
  resultDisplay: ToolResultDisplay | undefined;
  error: Error | undefined;
  errorType: ToolErrorType | undefined;
  outputFile?: string | undefined;
  contentLength?: number;
}
```

### GeminiFinishedEventValue

```typescript
export interface GeminiFinishedEventValue {
  reason: FinishReason | undefined;
  usageMetadata: GenerateContentResponseUsageMetadata | undefined;
}
```

## 주요 메서드 요약

| 메서드 | 설명 |
|-------|------|
| `run()` | Turn 실행 및 이벤트 스트리밍 (핵심) |
| `handlePendingFunctionCall()` | 함수 호출 이벤트 생성 |
| `getDebugResponses()` | 디버그 응답 조회 |

## 이벤트 흐름 다이어그램

```
API Response Stream
    ↓
Turn.run()
    ↓
이벤트 분류
    ├─→ Thought Part → Thought 이벤트
    ├─→ Text Content → Content 이벤트
    ├─→ Function Call → ToolCallRequest 이벤트
    ├─→ Citation → 누적 (턴 종료 시 표시)
    └─→ Finish Reason → Finished 이벤트
```

## 실전 예제

### 단순 텍스트 응답

```typescript
// API 응답
{
  candidates: [{
    content: {
      parts: [{ text: "TypeScript는 JavaScript의 상위 집합입니다." }]
    },
    finishReason: "STOP"
  }]
}

// 생성되는 이벤트
1. { type: 'content', value: "TypeScript는 JavaScript의 상위 집합입니다." }
2. { type: 'finished', value: { reason: "STOP", ... } }
```

### 도구 호출 응답

```typescript
// API 응답
{
  candidates: [{
    content: {
      parts: [{
        functionCall: {
          name: "read-file",
          args: { file_path: "/path/to/file.ts" }
        }
      }]
    },
    finishReason: "STOP"
  }]
}

// 생성되는 이벤트
1. {
     type: 'tool_call_request',
     value: {
       callId: "read-file-1234567890-abc",
       name: "read-file",
       args: { file_path: "/path/to/file.ts" },
       isClientInitiated: false,
       prompt_id: "prompt-123"
     }
   }
2. { type: 'finished', value: { reason: "STOP", ... } }
```

### Thought가 포함된 응답

```typescript
// API 응답
{
  candidates: [{
    content: {
      parts: [{
        thought: true,
        text: "**파일 읽기 계획**\n먼저 파일 경로를 확인하고..."
      }]
    }
  }],
  // ... 다음 청크에서 실제 응답
}

// 생성되는 이벤트
1. {
     type: 'thought',
     value: {
       subject: "파일 읽기 계획",
       description: "먼저 파일 경로를 확인하고..."
     }
   }
2. { type: 'content', value: "파일을 읽었습니다..." }
3. { type: 'finished', ... }
```

## 다음 장에서는

6장에서는 도구 시스템을 살펴보며, 20개 이상의 내장 도구와 MCP 통합을 알아보겠습니다.
