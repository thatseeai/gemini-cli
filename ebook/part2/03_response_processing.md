# 3장. 응답 처리 (Response Processing)

## 개요

이 장에서는 Gemini API로부터 받은 스트리밍 응답을 어떻게 처리하는지 살펴봅니다. 스트림 청크 파싱, 이벤트 변환, 응답 검증, 그리고 에러 처리를 실제 코드와 응답 예제와 함께 분석합니다.

## 응답 스트림 구조

### GenerateContentResponse

```typescript
interface GenerateContentResponse {
  candidates?: Candidate[];
  usageMetadata?: UsageMetadata;
  responseId?: string;
  promptFeedback?: PromptFeedback;
}

interface Candidate {
  content?: Content;
  finishReason?: FinishReason;
  citationMetadata?: CitationMetadata;
  safetyRatings?: SafetyRating[];
}

interface Content {
  role: 'model';
  parts: Part[];
}

type Part =
  | { text: string }
  | { functionCall: FunctionCall }
  | { thought: boolean; text: string };

interface UsageMetadata {
  promptTokenCount?: number;
  candidatesTokenCount?: number;
  totalTokenCount?: number;
}
```

### 실제 응답 청크 예제

#### 청크 1: Thought 시작

```json
{
  "candidates": [
    {
      "content": {
        "role": "model",
        "parts": [
          {
            "thought": true,
            "text": "**파일 읽기 계획**\n사용자가 README.md 파일을 읽기를 요청했습니다. read-file 도구를 사용하여 파일 내용을 가져와야 합니다."
          }
        ]
      }
    }
  ],
  "responseId": "abc123"
}
```

#### 청크 2: Function Call

```json
{
  "candidates": [
    {
      "content": {
        "role": "model",
        "parts": [
          {
            "functionCall": {
              "name": "read-file",
              "args": {
                "file_path": "README.md"
              }
            }
          }
        ]
      },
      "finishReason": "STOP"
    }
  ],
  "usageMetadata": {
    "promptTokenCount": 1234,
    "candidatesTokenCount": 56,
    "totalTokenCount": 1290
  }
}
```

#### 청크 3: 텍스트 응답 (여러 청크로 분할됨)

```json
// 첫 번째 텍스트 청크
{
  "candidates": [
    {
      "content": {
        "role": "model",
        "parts": [
          {
            "text": "README.md 파일의 내용은 다음과 같습니다:\n\n"
          }
        ]
      }
    }
  ]
}

// 두 번째 텍스트 청크
{
  "candidates": [
    {
      "content": {
        "role": "model",
        "parts": [
          {
            "text": "이 파일은 프로젝트에 대한 개요와 설치 방법을 설명하고 있으며"
          }
        ]
      }
    }
  ]
}

// 마지막 텍스트 청크
{
  "candidates": [
    {
      "content": {
        "role": "model",
        "parts": [
          {
            "text": " 사용 예제도 포함되어 있습니다."
          }
        ]
      },
      "finishReason": "STOP"
    }
  ],
  "usageMetadata": {
    "promptTokenCount": 1500,
    "candidatesTokenCount": 150,
    "totalTokenCount": 1650
  }
}
```

## 응답 처리 파이프라인

### 1단계: 스트림 수신 및 검증

**파일**: `packages/core/src/core/geminiChat.ts`

```typescript
private async *processStreamResponse(
  model: string,
  streamResponse: AsyncGenerator<GenerateContentResponse>,
): AsyncGenerator<GenerateContentResponse> {
  const modelResponseParts: Part[] = [];
  let hasToolCall = false;
  let finishReason: FinishReason | undefined;

  for await (const chunk of streamResponse) {
    // 1. Finish Reason 추출
    const candidateWithReason = chunk?.candidates?.find(
      (candidate) => candidate.finishReason,
    );
    if (candidateWithReason) {
      finishReason = candidateWithReason.finishReason as FinishReason;
    }

    // 2. 유효성 검증
    if (isValidResponse(chunk)) {
      const content = chunk.candidates?.[0]?.content;
      if (content?.parts) {
        // Thought 기록
        if (content.parts.some((part) => part.thought)) {
          this.recordThoughtFromContent(content);
        }

        // Function Call 감지
        if (content.parts.some((part) => part.functionCall)) {
          hasToolCall = true;
        }

        // Thought 제외하고 수집
        modelResponseParts.push(
          ...content.parts.filter((part) => !part.thought),
        );
      }
    }

    // 3. 토큰 사용량 기록
    if (chunk.usageMetadata) {
      this.chatRecordingService.recordMessageTokens(chunk.usageMetadata);
      if (chunk.usageMetadata.promptTokenCount !== undefined) {
        this.lastPromptTokenCount = chunk.usageMetadata.promptTokenCount;
      }
    }

    // 4. UI에 즉시 전달
    yield chunk;
  }

  // 후처리 계속...
}
```

### 2단계: 파트 통합 및 검증

```typescript
// Thought 제거 및 텍스트 파트 통합
const consolidatedParts: Part[] = [];
for (const part of modelResponseParts) {
  const lastPart = consolidatedParts[consolidatedParts.length - 1];

  // 연속된 텍스트 파트 병합
  if (
    lastPart?.text &&
    isValidNonThoughtTextPart(lastPart) &&
    isValidNonThoughtTextPart(part)
  ) {
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
```

### 3단계: 스트림 검증

```typescript
// 스트림 검증 로직
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
```

**검증 규칙**:
1. 도구 호출이 있으면 항상 유효
2. 도구 호출이 없으면:
   - Finish Reason 필수
   - MALFORMED_FUNCTION_CALL 아니어야 함
   - 응답 텍스트 비어있지 않아야 함

### 4단계: Turn에서 이벤트 생성

**파일**: `packages/core/src/core/turn.ts`

```typescript
async *run(
  model: string,
  req: PartListUnion,
  signal: AbortSignal,
): AsyncGenerator<ServerGeminiStreamEvent> {
  const responseStream = await this.chat.sendMessageStream(
    model,
    { message: req, config: { abortSignal: signal } },
    this.prompt_id,
  );

  for await (const streamEvent of responseStream) {
    // RETRY 이벤트
    if (streamEvent.type === 'retry') {
      yield { type: GeminiEventType.Retry };
      continue;
    }

    const resp = streamEvent.value as GenerateContentResponse;
    const traceId = resp.responseId;

    // Thought 이벤트
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

    // Content 이벤트
    const text = getResponseText(resp);
    if (text) {
      yield { type: GeminiEventType.Content, value: text, traceId };
    }

    // Tool Call 이벤트
    const functionCalls = resp.functionCalls ?? [];
    for (const fnCall of functionCalls) {
      const event = this.handlePendingFunctionCall(fnCall);
      if (event) {
        yield event;
      }
    }

    // Citation 수집
    for (const citation of getCitations(resp)) {
      this.pendingCitations.add(citation);
    }

    // Finished 이벤트
    const finishReason = resp.candidates?.[0]?.finishReason;
    if (finishReason) {
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
}
```

## 이벤트 타입별 처리 예제

### 1. Thought 이벤트

**원본 응답**:
```json
{
  "candidates": [{
    "content": {
      "parts": [{
        "thought": true,
        "text": "**파일 읽기 전략**\n먼저 파일 존재 여부를 확인하고, 파일이 크면 일부만 읽어야 할 것 같습니다."
      }]
    }
  }]
}
```

**생성된 이벤트**:
```typescript
{
  type: GeminiEventType.Thought,
  value: {
    subject: "파일 읽기 전략",
    description: "먼저 파일 존재 여부를 확인하고, 파일이 크면 일부만 읽어야 할 것 같습니다."
  },
  traceId: "abc123"
}
```

**UI 표시**:
```
💭 파일 읽기 전략
   먼저 파일 존재 여부를 확인하고, 파일이 크면 일부만 읽어야 할 것 같습니다.
```

### 2. Content 이벤트

**원본 응답** (여러 청크):
```json
// 청크 1
{ "candidates": [{ "content": { "parts": [{ "text": "README.md 파일을 " }] } }] }

// 청크 2
{ "candidates": [{ "content": { "parts": [{ "text": "읽었습니다. " }] } }] }

// 청크 3
{ "candidates": [{ "content": { "parts": [{ "text": "내용은 다음과 같습니다:" }] } }] }
```

**생성된 이벤트** (각 청크마다):
```typescript
{ type: GeminiEventType.Content, value: "README.md 파일을 ", traceId: "abc123" }
{ type: GeminiEventType.Content, value: "읽었습니다. ", traceId: "abc123" }
{ type: GeminiEventType.Content, value: "내용은 다음과 같습니다:", traceId: "abc123" }
```

**UI 표시** (스트리밍):
```
README.md 파일을 ▋
README.md 파일을 읽었습니다. ▋
README.md 파일을 읽었습니다. 내용은 다음과 같습니다:▋
```

### 3. ToolCallRequest 이벤트

**원본 응답**:
```json
{
  "candidates": [{
    "content": {
      "parts": [{
        "functionCall": {
          "name": "read-file",
          "args": {
            "file_path": "README.md"
          }
        }
      }]
    },
    "finishReason": "STOP"
  }]
}
```

**생성된 이벤트**:
```typescript
{
  type: GeminiEventType.ToolCallRequest,
  value: {
    callId: "read-file-1701234567890-abc",
    name: "read-file",
    args: {
      file_path: "README.md"
    },
    isClientInitiated: false,
    prompt_id: "prompt-xyz"
  }
}
```

**UI 표시**:
```
🔧 read-file
   file_path: "README.md"

   [Confirm] [Deny] [Proceed Always]
```

### 4. Citation 이벤트

**원본 응답**:
```json
{
  "candidates": [{
    "citationMetadata": {
      "citations": [
        {
          "uri": "https://example.com/doc1",
          "title": "Example Documentation"
        },
        {
          "uri": "https://example.com/doc2"
        }
      ]
    }
  }]
}
```

**생성된 이벤트**:
```typescript
{
  type: GeminiEventType.Citation,
  value: `Citations:
(Example Documentation) https://example.com/doc1
https://example.com/doc2`
}
```

### 5. Finished 이벤트

**원본 응답**:
```json
{
  "candidates": [{
    "finishReason": "STOP"
  }],
  "usageMetadata": {
    "promptTokenCount": 1500,
    "candidatesTokenCount": 150,
    "totalTokenCount": 1650
  }
}
```

**생성된 이벤트**:
```typescript
{
  type: GeminiEventType.Finished,
  value: {
    reason: "STOP",
    usageMetadata: {
      promptTokenCount: 1500,
      candidatesTokenCount: 150,
      totalTokenCount: 1650
    }
  }
}
```

## FinishReason 타입

```typescript
enum FinishReason {
  STOP = "STOP",                           // 정상 완료
  MAX_TOKENS = "MAX_TOKENS",               // 최대 토큰 도달
  SAFETY = "SAFETY",                       // 안전 필터링
  RECITATION = "RECITATION",               // 인용 감지
  OTHER = "OTHER",                         // 기타
  MALFORMED_FUNCTION_CALL = "MALFORMED_FUNCTION_CALL", // 잘못된 함수 호출
}
```

## 에러 처리

### InvalidStreamError 처리

```typescript
try {
  const stream = await this.makeApiCallAndProcessStream(...);
  for await (const chunk of stream) {
    yield { type: StreamEventType.CHUNK, value: chunk };
  }
} catch (error) {
  if (error instanceof InvalidStreamError) {
    // 재시도 로직
    if (attempt < maxAttempts - 1) {
      logContentRetry(...);
      await sleep(500 * (attempt + 1));
      continue;
    }

    // 재시도 실패
    logContentRetryFailure(...);
    throw error;
  }
  throw error;
}
```

### Safety Filter 처리

```typescript
const finishReason = resp.candidates?.[0]?.finishReason;

if (finishReason === FinishReason.SAFETY) {
  const safetyRatings = resp.candidates?.[0]?.safetyRatings;
  const blockedCategories = safetyRatings
    ?.filter((rating) => rating.blocked)
    .map((rating) => rating.category);

  throw new Error(
    `Response blocked by safety filters: ${blockedCategories?.join(', ')}`,
  );
}
```

### Recitation 처리

```typescript
if (finishReason === FinishReason.RECITATION) {
  // 인용 감지로 응답 차단
  yield {
    type: GeminiEventType.Error,
    value: {
      error: {
        message: 'Response blocked due to potential recitation of copyrighted content.',
      },
    },
  };
}
```

## 응답 처리 최적화

### 1. 텍스트 파트 통합

연속된 텍스트 파트를 하나로 병합하여 히스토리 크기 감소:

```typescript
const consolidatedParts: Part[] = [];
for (const part of modelResponseParts) {
  const lastPart = consolidatedParts[consolidatedParts.length - 1];

  if (
    lastPart?.text &&
    isValidNonThoughtTextPart(lastPart) &&
    isValidNonThoughtTextPart(part)
  ) {
    // 병합
    lastPart.text += part.text;
  } else {
    consolidatedParts.push(part);
  }
}
```

**병합 전**:
```typescript
[
  { text: "README.md 파일을 " },
  { text: "읽었습니다. " },
  { text: "내용은 다음과 같습니다:" }
]
```

**병합 후**:
```typescript
[
  { text: "README.md 파일을 읽었습니다. 내용은 다음과 같습니다:" }
]
```

### 2. Thought 제거

Thought는 UI에 표시하지만 히스토리에는 저장하지 않음:

```typescript
modelResponseParts.push(
  ...content.parts.filter((part) => !part.thought),
);
```

### 3. 토큰 카운팅

```typescript
if (chunk.usageMetadata) {
  this.chatRecordingService.recordMessageTokens(chunk.usageMetadata);
  if (chunk.usageMetadata.promptTokenCount !== undefined) {
    this.lastPromptTokenCount = chunk.usageMetadata.promptTokenCount;
  }
}
```

## 응답 처리 흐름 요약

```
API 응답 스트림
    ↓
[1] 청크 수신
    ↓
[2] 유효성 검증 (isValidResponse)
    ↓
[3] Thought 기록 및 제거
    ↓
[4] 파트 수집 (modelResponseParts)
    ↓
[5] 토큰 사용량 기록
    ↓
[6] UI에 즉시 전달 (yield chunk)
    ↓
스트림 종료
    ↓
[7] 텍스트 파트 통합
    ↓
[8] 스트림 검증
    ↓
[9] 히스토리에 추가
    ↓
[10] Turn에서 이벤트 생성
    ↓
최종 이벤트 스트림 →
```

## 다음 장에서는

4장에서는 도구 실행 프로세스를 자세히 살펴보며, LLM이 요청한 도구 호출을 어떻게 처리하고 결과를 반환하는지 알아보겠습니다.
