---
title: "3장. GeminiClient - 클라이언트 관리자"
weight: 03
---

# 3장. GeminiClient - 클라이언트 관리자

## 개요

`GeminiClient`는 Gemini CLI의 최상위 클라이언트 관리 클래스로, Gemini API와의 모든 상호작용을 조율합니다. 이 클래스는 대화 세션 관리, 메시지 스트리밍, 컨텍스트 압축, 모델 라우팅 등의 핵심 기능을 제공합니다.

**파일 위치**: `packages/core/src/core/client.ts`

## 클래스 구조

### 주요 속성

```typescript
export class GeminiClient {
  private chat?: GeminiChat;
  private readonly generateContentConfig: GenerateContentConfig = {
    temperature: 1,
    topP: 0.95,
    topK: 64,
  };
  private sessionTurnCount = 0;
  private readonly loopDetector: LoopDetectionService;
  private readonly compressionService: ChatCompressionService;
  private lastPromptId: string;
  private currentSequenceModel: string | null = null;
  private lastSentIdeContext: IdeContext | undefined;
  private forceFullIdeContext = true;
  private hasFailedCompressionAttempt = false;

  constructor(private readonly config: Config) {
    this.loopDetector = new LoopDetectionService(config);
    this.compressionService = new ChatCompressionService();
    this.lastPromptId = this.config.getSessionId();
  }
}
```

### 주요 속성 설명

- **chat**: `GeminiChat` 인스턴스, 실제 채팅 세션을 관리
- **generateContentConfig**: API 호출 시 사용할 기본 설정 (온도, topP, topK)
- **sessionTurnCount**: 현재 세션의 턴 수 추적
- **loopDetector**: 무한 루프 감지 서비스
- **compressionService**: 채팅 히스토리 압축 서비스
- **currentSequenceModel**: 현재 시퀀스에서 사용 중인 모델 (모델 고정)
- **lastSentIdeContext**: IDE 컨텍스트 변경 추적

## 핵심 기능

### 1. 초기화 (Initialization)

```typescript
async initialize() {
  this.chat = await this.startChat();
  this.updateTelemetryTokenCount();
}

async startChat(
  extraHistory?: Content[],
  resumedSessionData?: ResumedSessionData,
): Promise<GeminiChat> {
  this.forceFullIdeContext = true;
  this.hasFailedCompressionAttempt = false;

  const toolRegistry = this.config.getToolRegistry();
  const toolDeclarations = toolRegistry.getFunctionDeclarations();
  const tools: Tool[] = [{ functionDeclarations: toolDeclarations }];

  const history = await getInitialChatHistory(this.config, extraHistory);

  const userMemory = this.config.getUserMemory();
  const systemInstruction = getCoreSystemPrompt(this.config, userMemory);
  const model = this.config.getModel();

  const config: GenerateContentConfig = { ...this.generateContentConfig };

  // Thinking 모드 지원 체크
  if (isThinkingSupported(model)) {
    config.thinkingConfig = {
      includeThoughts: true,
      thinkingBudget: DEFAULT_THINKING_MODE,
    };
  }

  return new GeminiChat(
    this.config,
    {
      systemInstruction,
      ...config,
      tools,
    },
    history,
    resumedSessionData,
  );
}
```

**초기화 프로세스**:
1. 도구 레지스트리에서 함수 선언 가져오기
2. 초기 채팅 히스토리 구성
3. 시스템 프롬프트 생성
4. Thinking 모드 설정 (모델이 지원하는 경우)
5. GeminiChat 인스턴스 생성

### 2. 메시지 스트리밍 (Message Streaming)

`sendMessageStream`은 GeminiClient의 핵심 메서드로, 사용자 메시지를 받아 API 응답을 스트리밍합니다.

```typescript
async *sendMessageStream(
  request: PartListUnion,
  signal: AbortSignal,
  prompt_id: string,
  turns: number = MAX_TURNS,
  isInvalidStreamRetry: boolean = false,
): AsyncGenerator<ServerGeminiStreamEvent, Turn> {
  // 1. 프롬프트 ID 체크 및 초기화
  if (this.lastPromptId !== prompt_id) {
    this.loopDetector.reset(prompt_id);
    this.lastPromptId = prompt_id;
    this.currentSequenceModel = null;
  }

  // 2. 세션 턴 제한 체크
  this.sessionTurnCount++;
  if (
    this.config.getMaxSessionTurns() > 0 &&
    this.sessionTurnCount > this.config.getMaxSessionTurns()
  ) {
    yield { type: GeminiEventType.MaxSessionTurns };
    return new Turn(this.getChat(), prompt_id);
  }

  // 3. 컨텍스트 윈도우 오버플로우 체크
  const modelForLimitCheck = this._getEffectiveModelForCurrentTurn();
  const estimatedRequestTokenCount = Math.floor(
    JSON.stringify(request).length / 4,
  );
  const remainingTokenCount =
    tokenLimit(modelForLimitCheck) - this.getChat().getLastPromptTokenCount();

  if (estimatedRequestTokenCount > remainingTokenCount * 0.95) {
    yield {
      type: GeminiEventType.ContextWindowWillOverflow,
      value: { estimatedRequestTokenCount, remainingTokenCount },
    };
    return new Turn(this.getChat(), prompt_id);
  }

  // 4. 채팅 히스토리 압축 시도
  const compressed = await this.tryCompressChat(prompt_id, false);
  if (compressed.compressionStatus === CompressionStatus.COMPRESSED) {
    yield { type: GeminiEventType.ChatCompressed, value: compressed };
  }

  // 5. IDE 컨텍스트 추가 (해당하는 경우)
  if (this.config.getIdeMode() && !hasPendingToolCall) {
    const { contextParts, newIdeContext } = this.getIdeContextParts(
      this.forceFullIdeContext || history.length === 0,
    );
    if (contextParts.length > 0) {
      this.getChat().addHistory({
        role: 'user',
        parts: [{ text: contextParts.join('\n') }],
      });
    }
    this.lastSentIdeContext = newIdeContext;
    this.forceFullIdeContext = false;
  }

  // 6. Turn 생성 및 실행
  const turn = new Turn(this.getChat(), prompt_id);

  // 7. 루프 감지
  const loopDetected = await this.loopDetector.turnStarted(signal);
  if (loopDetected) {
    yield { type: GeminiEventType.LoopDetected };
    return turn;
  }

  // 8. 모델 라우팅 (최초 또는 모델 변경 시)
  let modelToUse: string;
  if (this.currentSequenceModel) {
    modelToUse = this.currentSequenceModel;
  } else {
    const router = await this.config.getModelRouterService();
    const decision = await router.route(routingContext);
    modelToUse = decision.model;
    this.currentSequenceModel = modelToUse; // 모델 고정
    yield { type: GeminiEventType.ModelInfo, value: modelToUse };
  }

  // 9. Turn 실행 및 이벤트 스트리밍
  const resultStream = turn.run(modelToUse, request, linkedSignal);
  for await (const event of resultStream) {
    // 루프 감지
    if (this.loopDetector.addAndCheck(event)) {
      yield { type: GeminiEventType.LoopDetected };
      controller.abort();
      return turn;
    }
    yield event;

    // 텔레메트리 업데이트
    this.updateTelemetryTokenCount();

    // InvalidStream 처리 (재시도)
    if (event.type === GeminiEventType.InvalidStream) {
      if (this.config.getContinueOnFailedApiCall() && !isInvalidStreamRetry) {
        const nextRequest = [{ text: 'System: Please continue.' }];
        yield* this.sendMessageStream(
          nextRequest,
          signal,
          prompt_id,
          boundedTurns - 1,
          true,
        );
        return turn;
      }
    }

    if (event.type === GeminiEventType.Error) {
      return turn;
    }
  }

  // 10. Next Speaker 체크 (도구 호출 없이 완료된 경우)
  if (!turn.pendingToolCalls.length && signal && !signal.aborted) {
    if (this.config.getSkipNextSpeakerCheck()) {
      return turn;
    }

    const nextSpeakerCheck = await checkNextSpeaker(
      this.getChat(),
      this.config.getBaseLlmClient(),
      signal,
      prompt_id,
    );

    if (nextSpeakerCheck?.next_speaker === 'model') {
      const nextRequest = [{ text: 'Please continue.' }];
      yield* this.sendMessageStream(
        nextRequest,
        signal,
        prompt_id,
        boundedTurns - 1,
      );
    }
  }

  return turn;
}
```

### 3. 채팅 압축 (Chat Compression)

대화 히스토리가 길어지면 컨텍스트 윈도우를 초과할 수 있습니다. `tryCompressChat` 메서드는 이를 방지하기 위해 히스토리를 압축합니다.

```typescript
async tryCompressChat(
  prompt_id: string,
  force: boolean = false,
): Promise<ChatCompressionInfo> {
  const model = this._getEffectiveModelForCurrentTurn();

  const { newHistory, info } = await this.compressionService.compress(
    this.getChat(),
    prompt_id,
    force,
    model,
    this.config,
    this.hasFailedCompressionAttempt,
  );

  if (
    info.compressionStatus ===
    CompressionStatus.COMPRESSION_FAILED_INFLATED_TOKEN_COUNT
  ) {
    this.hasFailedCompressionAttempt = !force && true;
  } else if (info.compressionStatus === CompressionStatus.COMPRESSED) {
    if (newHistory) {
      this.chat = await this.startChat(newHistory);
      this.updateTelemetryTokenCount();
      this.forceFullIdeContext = true;
    }
  }

  return info;
}
```

**압축 프로세스**:
1. 현재 히스토리를 압축 서비스에 전달
2. 압축된 새 히스토리 생성
3. 압축 성공 시 새 채팅 세션 시작
4. IDE 컨텍스트 재전송 플래그 설정

### 4. IDE 컨텍스트 관리

VS Code와 같은 IDE와 통합할 때, 현재 열린 파일, 커서 위치, 선택된 텍스트 등의 컨텍스트를 LLM에 제공합니다.

```typescript
private getIdeContextParts(forceFullContext: boolean): {
  contextParts: string[];
  newIdeContext: IdeContext | undefined;
} {
  const currentIdeContext = ideContextStore.get();
  if (!currentIdeContext) {
    return { contextParts: [], newIdeContext: undefined };
  }

  if (forceFullContext || !this.lastSentIdeContext) {
    // 전체 컨텍스트 전송
    const openFiles = currentIdeContext.workspaceState?.openFiles || [];
    const activeFile = openFiles.find((f) => f.isActive);
    const otherOpenFiles = openFiles
      .filter((f) => !f.isActive)
      .map((f) => f.path);

    const contextData: Record<string, unknown> = {};

    if (activeFile) {
      contextData['activeFile'] = {
        path: activeFile.path,
        cursor: activeFile.cursor
          ? {
              line: activeFile.cursor.line,
              character: activeFile.cursor.character,
            }
          : undefined,
        selectedText: activeFile.selectedText || undefined,
      };
    }

    if (otherOpenFiles.length > 0) {
      contextData['otherOpenFiles'] = otherOpenFiles;
    }

    if (Object.keys(contextData).length === 0) {
      return { contextParts: [], newIdeContext: currentIdeContext };
    }

    const jsonString = JSON.stringify(contextData, null, 2);
    const contextParts = [
      "Here is the user's editor context as a JSON object. This is for your information only.",
      '```json',
      jsonString,
      '```',
    ];

    return { contextParts, newIdeContext: currentIdeContext };
  } else {
    // 델타 컨텍스트 전송 (변경 사항만)
    const delta: Record<string, unknown> = {};
    const changes: Record<string, unknown> = {};

    // 파일 열림/닫힘 감지
    // 활성 파일 변경 감지
    // 커서 이동 감지
    // 선택 텍스트 변경 감지
    // ... (델타 계산 로직)

    if (Object.keys(changes).length === 0) {
      return { contextParts: [], newIdeContext: currentIdeContext };
    }

    delta['changes'] = changes;
    const jsonString = JSON.stringify(delta, null, 2);
    const contextParts = [
      "Here is a summary of changes in the user's editor context, in JSON format. This is for your information only.",
      '```json',
      jsonString,
      '```',
    ];

    return { contextParts, newIdeContext: currentIdeContext };
  }
}
```

**IDE 컨텍스트 최적화**:
- 최초 또는 강제 업데이트 시: 전체 컨텍스트 전송
- 이후: 델타만 전송하여 토큰 절약
- JSON 형식으로 구조화된 정보 제공

## 모델 고정 (Model Stickiness)

한 번 모델이 선택되면, 해당 시퀀스(prompt_id가 동일한 동안)에서는 계속 같은 모델을 사용합니다.

```typescript
// 모델 결정 로직
if (this.currentSequenceModel) {
  modelToUse = this.currentSequenceModel; // 기존 모델 유지
} else {
  const router = await this.config.getModelRouterService();
  const decision = await router.route(routingContext);
  modelToUse = decision.model;
  this.currentSequenceModel = modelToUse; // 새 모델 고정
  yield { type: GeminiEventType.ModelInfo, value: modelToUse };
}
```

**목적**:
- 일관된 응답 품질 유지
- 모델 전환으로 인한 컨텍스트 손실 방지
- 사용자 경험 개선

## 에러 처리 및 재시도

### Invalid Stream 재시도

API가 유효하지 않은 응답을 반환하는 경우, 자동으로 재시도합니다.

```typescript
if (event.type === GeminiEventType.InvalidStream) {
  if (this.config.getContinueOnFailedApiCall()) {
    if (isInvalidStreamRetry) {
      // 이미 재시도한 경우 중단
      return turn;
    }
    const nextRequest = [{ text: 'System: Please continue.' }];
    yield* this.sendMessageStream(
      nextRequest,
      signal,
      prompt_id,
      boundedTurns - 1,
      true, // isInvalidStreamRetry 플래그 설정
    );
    return turn;
  }
}
```

### Next Speaker Check

도구 호출 없이 응답이 완료된 경우, 모델이 계속 말하고 싶은지 확인합니다.

```typescript
const nextSpeakerCheck = await checkNextSpeaker(
  this.getChat(),
  this.config.getBaseLlmClient(),
  signal,
  prompt_id,
);

if (nextSpeakerCheck?.next_speaker === 'model') {
  const nextRequest = [{ text: 'Please continue.' }];
  yield* this.sendMessageStream(
    nextRequest,
    signal,
    prompt_id,
    boundedTurns - 1,
  );
}
```

## 주요 메서드 요약

| 메서드 | 설명 |
|-------|------|
| `initialize()` | 클라이언트 초기화 |
| `startChat()` | 새 채팅 세션 시작 |
| `sendMessageStream()` | 메시지 스트리밍 (핵심) |
| `tryCompressChat()` | 채팅 히스토리 압축 |
| `getIdeContextParts()` | IDE 컨텍스트 구성 |
| `setTools()` | 도구 설정 업데이트 |
| `getHistory()` | 채팅 히스토리 조회 |
| `setHistory()` | 채팅 히스토리 설정 |

## 다음 장에서는

4장에서는 GeminiChat 클래스를 살펴보며, 실제 API 통신과 히스토리 관리가 어떻게 이루어지는지 알아보겠습니다.
