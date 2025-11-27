---
title: "2장. 전체 아키텍처"
weight: 02
---

## 아키텍처 개요

Gemini CLI의 아키텍처는 **이벤트 기반 스트리밍 아키텍처**를 채택하여, 사용자와 LLM 간의 실시간 상호작용을 지원합니다. 이 장에서는 전체 시스템의 데이터 흐름과 주요 컴포넌트 간의 상호작용을 살펴봅니다.

## 전체 데이터 흐름

```
사용자 입력
    ↓
CLI Hook (useGeminiStream)
    ↓
GeminiClient
    ↓
프롬프트 구성 (getCoreSystemPrompt)
    ↓
Gemini API
    ↓
스트림 응답
    ↓
Turn (이벤트 파싱)
    ↓
이벤트 분류
    ├─→ Content: 텍스트 표시
    ├─→ Thought: 사고 과정 표시
    └─→ ToolCallRequest: 도구 실행
        ↓
    ToolRegistry → Build → Execute
        ↓
    ToolResult
        ↓
    FunctionResponse
        ↓
    Gemini API (반복)
        ↓
    대화 완료
```

## 계층별 구조

### 1. 프레젠테이션 계층 (Presentation Layer)

**위치**: `packages/cli/src/`

**역할**: 사용자 인터페이스와 상호작용

**주요 컴포넌트**:
- `AppContainer.tsx`: 메인 오케스트레이터
- `useGeminiStream.ts`: 스트리밍 훅
- `useReactToolScheduler.ts`: 도구 실행 스케줄링

**코드 예제** (`packages/cli/src/hooks/useGeminiStream.ts`):
```typescript
export function useGeminiStream(config: Config) {
  const startTurn = async (
    message: string,
    signal: AbortSignal,
  ): Promise<void> => {
    const client = config.getClient();
    const stream = client.sendMessageStream(
      [{ text: message }],
      signal,
      promptId,
    );

    for await (const event of stream) {
      // 이벤트 처리
      switch (event.type) {
        case GeminiEventType.Content:
          // 텍스트 콘텐츠 표시
          break;
        case GeminiEventType.ToolCallRequest:
          // 도구 호출 요청 처리
          break;
        // ...
      }
    }
  };

  return { startTurn };
}
```

### 2. 비즈니스 로직 계층 (Business Logic Layer)

**위치**: `packages/core/src/core/`

**역할**: 핵심 비즈니스 로직 처리

**주요 컴포넌트**:
- `GeminiClient`: 클라이언트 관리
- `GeminiChat`: 채팅 세션 관리
- `Turn`: 턴 기반 대화 처리

**코드 예제** (`packages/core/src/core/client.ts`):
```typescript
export class GeminiClient {
  private chat?: GeminiChat;
  private readonly generateContentConfig: GenerateContentConfig = {
    temperature: 1,
    topP: 0.95,
    topK: 64,
  };

  async *sendMessageStream(
    request: PartListUnion,
    signal: AbortSignal,
    prompt_id: string,
    turns: number = MAX_TURNS,
  ): AsyncGenerator<ServerGeminiStreamEvent, Turn> {
    // 컨텍스트 압축 시도
    const compressed = await this.tryCompressChat(prompt_id, false);

    if (compressed.compressionStatus === CompressionStatus.COMPRESSED) {
      yield { type: GeminiEventType.ChatCompressed, value: compressed };
    }

    // 모델 라우팅
    const router = await this.config.getModelRouterService();
    const decision = await router.route(routingContext);
    const modelToUse = decision.model;

    // Turn 실행
    const turn = new Turn(this.getChat(), prompt_id);
    const resultStream = turn.run(modelToUse, request, linkedSignal);

    for await (const event of resultStream) {
      yield event;
    }

    return turn;
  }
}
```

### 3. 데이터 액세스 계층 (Data Access Layer)

**위치**: `packages/core/src/tools/`, `packages/core/src/services/`

**역할**: 외부 리소스 및 API 접근

**주요 컴포넌트**:
- `ContentGenerator`: API 통신 인터페이스
- `ToolRegistry`: 도구 등록 및 관리
- `ChatRecordingService`: 대화 기록 서비스

## 주요 상호작용 시나리오

### 시나리오 1: 단순 질문-응답

```
1. 사용자: "TypeScript란 무엇인가?"
2. CLI → GeminiClient.sendMessageStream()
3. GeminiClient → 시스템 프롬프트 + 사용자 메시지 구성
4. GeminiClient → GeminiChat.sendMessageStream()
5. GeminiChat → ContentGenerator.generateContentStream()
6. ContentGenerator → Gemini API 호출
7. API 응답 스트림 → GeminiChat
8. GeminiChat → Turn.run() (이벤트 파싱)
9. Turn → Content 이벤트 생성
10. Content 이벤트 → CLI → 사용자에게 표시
```

### 시나리오 2: 도구 사용

```
1. 사용자: "README.md 파일을 읽어줘"
2. CLI → GeminiClient.sendMessageStream()
3. [위와 동일한 흐름]
4. API 응답: FunctionCall (read-file)
5. Turn → ToolCallRequest 이벤트 생성
6. CLI → ToolScheduler → 도구 실행 확인 요청
7. 사용자 승인
8. ToolRegistry.getTool("read-file")
9. Tool.build(params) → ToolInvocation
10. ToolInvocation.execute() → 파일 읽기
11. ToolResult → FunctionResponse 변환
12. FunctionResponse → Gemini API
13. API 응답: 파일 내용 요약
14. Content 이벤트 → 사용자에게 표시
```

### 시나리오 3: 다중 턴 대화 (Agentic Loop)

```
1. 사용자: "프로젝트의 모든 TODO를 찾아서 정리해줘"
2. [API 호출]
3. API: FunctionCall (grep "TODO")
4. [도구 실행: grep]
5. ToolResult → API
6. API: FunctionCall (read-file "file1.ts")
7. [도구 실행: read-file]
8. ToolResult → API
9. API: FunctionCall (write-file "todo-list.md")
10. [도구 실행: write-file]
11. ToolResult → API
12. API: 최종 응답 (Content)
13. 사용자에게 결과 표시
```

## 이벤트 스트림 구조

Gemini CLI는 이벤트 기반 아키텍처를 사용하여 다양한 이벤트를 처리합니다.

### ServerGeminiStreamEvent 타입

**파일**: `packages/core/src/core/turn.ts`

```typescript
export enum GeminiEventType {
  Content = 'content',                    // 텍스트 콘텐츠
  ToolCallRequest = 'tool_call_request',  // 도구 호출 요청
  ToolCallResponse = 'tool_call_response', // 도구 실행 결과
  ToolCallConfirmation = 'tool_call_confirmation', // 도구 확인 필요
  Thought = 'thought',                    // 모델의 사고 과정
  Error = 'error',                        // 에러 발생
  ChatCompressed = 'chat_compressed',     // 채팅 압축 완료
  Finished = 'finished',                  // 턴 완료
  LoopDetected = 'loop_detected',         // 무한 루프 감지
  Citation = 'citation',                  // 인용 정보
  Retry = 'retry',                        // 재시도
  ModelInfo = 'model_info',               // 모델 정보
  // ...
}

export type ServerGeminiStreamEvent =
  | ServerGeminiContentEvent
  | ServerGeminiToolCallRequestEvent
  | ServerGeminiToolCallResponseEvent
  | ServerGeminiThoughtEvent
  | ServerGeminiErrorEvent
  | ServerGeminiFinishedEvent
  // ...
```

### 이벤트 처리 예제

**파일**: `packages/core/src/core/turn.ts`

```typescript
export class Turn {
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
      if (streamEvent.type === 'retry') {
        yield { type: GeminiEventType.Retry };
        continue;
      }

      const resp = streamEvent.value as GenerateContentResponse;

      // Thought 파트 처리
      const thoughtPart = resp.candidates?.[0]?.content?.parts?.[0];
      if (thoughtPart?.thought) {
        const thought = parseThought(thoughtPart.text ?? '');
        yield { type: GeminiEventType.Thought, value: thought };
        continue;
      }

      // 텍스트 콘텐츠 처리
      const text = getResponseText(resp);
      if (text) {
        yield { type: GeminiEventType.Content, value: text };
      }

      // 함수 호출 처리
      const functionCalls = resp.functionCalls ?? [];
      for (const fnCall of functionCalls) {
        const event = this.handlePendingFunctionCall(fnCall);
        if (event) {
          yield event;
        }
      }

      // 완료 처리
      const finishReason = resp.candidates?.[0]?.finishReason;
      if (finishReason) {
        yield {
          type: GeminiEventType.Finished,
          value: { reason: finishReason, usageMetadata: resp.usageMetadata },
        };
      }
    }
  }
}
```

## 구성 및 설정 관리

### Config 클래스

전체 시스템의 설정을 중앙 집중화하여 관리합니다.

**주요 설정 항목**:
- 모델 선택 (gemini-2.5-pro, gemini-2.5-flash, etc.)
- 도구 레지스트리
- 컨텐츠 제너레이터
- 사용자 메모리
- IDE 모드
- 디버그 모드
- 샌드박스 설정

**코드 예제**:
```typescript
export class Config {
  getModel(): string;
  getToolRegistry(): ToolRegistry;
  getContentGenerator(): ContentGenerator;
  getUserMemory(): string | undefined;
  getIdeMode(): boolean;
  getDebugMode(): boolean;
  // ...
}
```

## 확장성과 유지보수성

### 모듈화된 구조
- 각 컴포넌트는 독립적으로 테스트 가능
- 명확한 인터페이스 정의
- 의존성 주입 패턴 사용

### 타입 안전성
- TypeScript를 통한 강력한 타입 체크
- 인터페이스 기반 설계
- 런타임 검증

### 테스트 가능성
- Vitest를 사용한 단위 테스트
- 모킹 가능한 구조
- 통합 테스트 지원

## 다음 장에서는

3장부터는 각 핵심 컴포넌트를 하나씩 자세히 살펴보겠습니다. 먼저 GeminiClient부터 시작합니다.
