# 2장. 요청 흐름 (Request Flow)

## 개요

이 장에서는 사용자 메시지가 어떻게 Gemini API 요청으로 변환되는지 단계별로 살펴봅니다. 요청 구성의 각 단계와 실제 페이로드를 코드와 함께 분석합니다.

## 요청 구성 단계

### 1단계: 사용자 입력 수신

```typescript
// packages/cli/src/hooks/useGeminiStream.ts
const startTurn = async (message: string, signal: AbortSignal) => {
  const client = config.getClient();
  const stream = client.sendMessageStream(
    [{ text: message }], // PartListUnion
    signal,
    promptId,
  );
  // ...
};
```

**사용자 입력**:
```
"README.md 파일을 읽어줘"
```

**변환 후**:
```typescript
[{ text: "README.md 파일을 읽어줘" }]
```

### 2단계: 시스템 프롬프트 생성

```typescript
// packages/core/src/core/client.ts
async startChat() {
  const userMemory = this.config.getUserMemory();
  const systemInstruction = getCoreSystemPrompt(this.config, userMemory);
  // ...
}
```

**시스템 프롬프트 구성** (packages/core/src/core/prompts.ts):

```typescript
export function getCoreSystemPrompt(
  config: Config,
  userMemory?: string,
): string {
  // 1. 기본 프롬프트 구성
  const basePrompt = buildPrompt(config);

  // 2. 사용자 메모리 추가
  const memorySuffix =
    userMemory && userMemory.trim().length > 0
      ? `\n\n---\n\n${userMemory.trim()}`
      : '';

  return `${basePrompt}${memorySuffix}`;
}
```

**생성된 시스템 프롬프트 예제**:
```
You are an interactive CLI agent specializing in software engineering tasks...

# Core Mandates
- **Conventions:** Rigorously adhere to existing project conventions...
- **Libraries/Frameworks:** NEVER assume a library/framework is available...

# Primary Workflows
...

# Operational Guidelines
...

---

User prefers TypeScript strict mode.
```

### 3단계: 도구 정의 생성

```typescript
// packages/core/src/core/client.ts
async startChat() {
  const toolRegistry = this.config.getToolRegistry();
  const toolDeclarations = toolRegistry.getFunctionDeclarations();
  const tools: Tool[] = [{ functionDeclarations: toolDeclarations }];
  // ...
}
```

**도구 정의 생성** (packages/core/src/tools/tool-registry.ts):

```typescript
export class ToolRegistry {
  getFunctionDeclarations(): FunctionDeclaration[] {
    return Array.from(this.tools.values()).map((tool) => ({
      name: tool.schema.name,
      description: tool.schema.description,
      parameters: tool.schema.parametersJsonSchema || tool.schema.parameters,
    }));
  }
}
```

**생성된 도구 정의 예제**:
```json
[
  {
    "name": "read-file",
    "description": "Read the contents of a file",
    "parameters": {
      "type": "object",
      "properties": {
        "file_path": {
          "type": "string",
          "description": "The path to the file to read"
        },
        "offset": {
          "type": "number",
          "description": "Line number to start reading from (0-indexed)"
        },
        "limit": {
          "type": "number",
          "description": "Maximum number of lines to read"
        }
      },
      "required": ["file_path"]
    }
  },
  {
    "name": "write-file",
    "description": "Write content to a file",
    "parameters": {
      "type": "object",
      "properties": {
        "file_path": { "type": "string" },
        "content": { "type": "string" }
      },
      "required": ["file_path", "content"]
    }
  }
  // ... 기타 도구들
]
```

### 4단계: 히스토리 구성

```typescript
// packages/core/src/core/geminiChat.ts
async sendMessageStream(model: string, params: SendMessageParameters) {
  const userContent = createUserContent(params.message);
  this.history.push(userContent);
  const requestContents = this.getHistory(true); // curated history
  // ...
}
```

**히스토리 예제** (이전 대화가 있는 경우):

```typescript
[
  // 이전 사용자 메시지
  {
    role: 'user',
    parts: [{ text: '현재 디렉토리의 파일 목록을 보여줘' }],
  },
  // 이전 모델 응답
  {
    role: 'model',
    parts: [
      {
        functionCall: {
          name: 'ls',
          args: { directory: '.' },
        },
      },
    ],
  },
  // 도구 실행 결과
  {
    role: 'user',
    parts: [
      {
        functionResponse: {
          name: 'ls',
          response: {
            files: ['README.md', 'package.json', 'src/'],
          },
        },
      },
    ],
  },
  // 모델의 최종 응답
  {
    role: 'model',
    parts: [
      {
        text: '현재 디렉토리에는 README.md, package.json, 그리고 src 폴더가 있습니다.',
      },
    ],
  },
  // 새로운 사용자 메시지
  {
    role: 'user',
    parts: [{ text: 'README.md 파일을 읽어줘' }],
  },
]
```

### 5단계: GenerateContentConfig 구성

```typescript
// packages/core/src/core/client.ts
private readonly generateContentConfig: GenerateContentConfig = {
  temperature: 1,
  topP: 0.95,
  topK: 64,
};

const config: GenerateContentConfig = { ...this.generateContentConfig };

// Thinking 모드 지원 체크
if (isThinkingSupported(model)) {
  config.thinkingConfig = {
    includeThoughts: true,
    thinkingBudget: DEFAULT_THINKING_MODE,
  };
}
```

**최종 Config**:
```typescript
{
  temperature: 1,
  topP: 0.95,
  topK: 64,
  systemInstruction: "You are an interactive CLI agent...",
  tools: [{ functionDeclarations: [...] }],
  thinkingConfig: {
    includeThoughts: true,
    thinkingBudget: 8192
  },
  abortSignal: signal
}
```

### 6단계: API 호출

```typescript
// packages/core/src/core/geminiChat.ts
private async makeApiCallAndProcessStream(
  model: string,
  requestContents: Content[],
  params: SendMessageParameters,
  prompt_id: string,
) {
  const apiCall = () => {
    const modelToUse = getEffectiveModel(
      this.config.isInFallbackMode(),
      model,
    );

    return this.config.getContentGenerator().generateContentStream(
      {
        model: modelToUse,
        contents: requestContents,
        config: { ...this.generationConfig, ...params.config },
      },
      prompt_id,
    );
  };

  const streamResponse = await retryWithBackoff(apiCall, {
    onPersistent429: onPersistent429Callback,
    authType: this.config.getContentGeneratorConfig()?.authType,
    retryFetchErrors: this.config.getRetryFetchErrors(),
    signal: params.config?.abortSignal,
  });

  return this.processStreamResponse(model, streamResponse);
}
```

**실제 API 요청 구조**:

```http
POST https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-pro:streamGenerateContent

Headers:
  Content-Type: application/json
  Authorization: Bearer <API_KEY>

Body:
{
  "contents": [
    {
      "role": "user",
      "parts": [
        { "text": "README.md 파일을 읽어줘" }
      ]
    }
  ],
  "systemInstruction": {
    "parts": [
      { "text": "You are an interactive CLI agent..." }
    ]
  },
  "generationConfig": {
    "temperature": 1,
    "topP": 0.95,
    "topK": 64
  },
  "tools": [
    {
      "functionDeclarations": [
        {
          "name": "read-file",
          "description": "Read the contents of a file",
          "parameters": {
            "type": "object",
            "properties": {
              "file_path": {
                "type": "string",
                "description": "The path to the file to read"
              }
            },
            "required": ["file_path"]
          }
        }
        // ... 기타 도구들
      ]
    }
  ],
  "thinkingConfig": {
    "includeThoughts": true,
    "thinkingBudget": 8192
  }
}
```

## 특수 상황별 요청 구성

### IDE 컨텍스트 포함

VS Code와 통합된 경우, 현재 열린 파일 등의 컨텍스트를 추가합니다.

```typescript
// packages/core/src/core/client.ts
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
}
```

**IDE 컨텍스트 메시지**:
```
Here is the user's editor context as a JSON object. This is for your information only.
```json
{
  "activeFile": {
    "path": "/workspace/src/index.ts",
    "cursor": {
      "line": 42,
      "character": 15
    },
    "selectedText": "const result = processData(input);"
  },
  "otherOpenFiles": [
    "/workspace/src/utils.ts",
    "/workspace/tests/index.test.ts"
  ]
}
```
```

### 압축된 히스토리

대화가 길어진 경우 압축된 히스토리를 사용합니다.

```typescript
// packages/core/src/core/client.ts
const compressed = await this.tryCompressChat(prompt_id, false);

if (compressed.compressionStatus === CompressionStatus.COMPRESSED) {
  yield { type: GeminiEventType.ChatCompressed, value: compressed };
}
```

**압축 전 히스토리** (50개 메시지, 10,000 토큰):
```typescript
[
  { role: 'user', parts: [...] },
  { role: 'model', parts: [...] },
  // ... 48개 더
]
```

**압축 후 히스토리** (1개 요약, 2,000 토큰):
```typescript
[
  {
    role: 'user',
    parts: [
      {
        text: `<state_snapshot>
<overall_goal>
  사용자가 인증 시스템을 JWT로 마이그레이션하려고 함
</overall_goal>
<key_knowledge>
  - 프로젝트는 TypeScript를 사용
  - 기존 인증은 session-based
  - package.json에 jsonwebtoken 라이브러리 추가 완료
</key_knowledge>
<file_system_state>
  - READ: package.json, src/auth/session.ts
  - MODIFIED: src/auth/jwt.ts (새로 생성)
  - CREATED: tests/auth.test.ts
</file_system_state>
<recent_actions>
  - jsonwebtoken 라이브러리 설치
  - JWT 토큰 생성 함수 구현
  - 테스트 작성 시작
</recent_actions>
<current_plan>
  1. [DONE] JWT 라이브러리 설치
  2. [DONE] 토큰 생성 함수 구현
  3. [IN PROGRESS] 테스트 작성
  4. [TODO] 기존 세션 코드 마이그레이션
  5. [TODO] 미들웨어 업데이트
</current_plan>
</state_snapshot>`,
      },
    ],
  },
  {
    role: 'user',
    parts: [{ text: '다음 단계를 계속 진행해줘' }],
  },
]
```

### Function Response

도구 실행 결과를 다음 요청에 포함합니다.

```typescript
// 도구 실행 결과 → FunctionResponse 변환
{
  role: 'user',
  parts: [
    {
      functionResponse: {
        name: 'read-file',
        response: {
          content: "# My Project\n\nThis is a sample README...",
          lines: 42
        }
      }
    }
  ]
}
```

## 재시도 메커니즘

### 429 에러 (Rate Limit)

```typescript
// packages/core/src/utils/retry.ts
export async function retryWithBackoff<T>(
  apiCall: () => Promise<T>,
  options: RetryOptions,
): Promise<T> {
  const maxAttempts = 5;
  const baseDelay = 2000; // 2초

  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    try {
      return await apiCall();
    } catch (error) {
      if (is429Error(error)) {
        if (attempt === maxAttempts - 1) {
          // 폴백 모델로 전환
          await options.onPersistent429?.();
          // 재시도
          return await apiCall();
        }
        // 지수 백오프
        const delay = baseDelay * Math.pow(2, attempt);
        await sleep(delay);
        continue;
      }
      throw error;
    }
  }
  throw new Error('Max retries exceeded');
}
```

### Invalid Stream 재시도

```typescript
// packages/core/src/core/geminiChat.ts
for (let attempt = 0; attempt < maxAttempts; attempt++) {
  try {
    if (attempt > 0) {
      yield { type: StreamEventType.RETRY };
      // 재시도 시 temperature를 1로 설정하여 다른 출력 유도
      currentParams.config = {
        ...currentParams.config,
        temperature: 1,
      };
    }

    const stream = await this.makeApiCallAndProcessStream(
      model,
      requestContents,
      currentParams,
      prompt_id,
    );

    for await (const chunk of stream) {
      yield { type: StreamEventType.CHUNK, value: chunk };
    }

    break; // 성공
  } catch (error) {
    if (error instanceof InvalidStreamError) {
      if (attempt < maxAttempts - 1) {
        // 재시도
        await sleep(500 * (attempt + 1));
        continue;
      }
    }
    throw error;
  }
}
```

## 요청 최적화

### 1. 토큰 제한 체크

```typescript
// packages/core/src/core/client.ts
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
```

### 2. 히스토리 선별 (Curated History)

```typescript
// packages/core/src/core/geminiChat.ts
function extractCuratedHistory(comprehensiveHistory: Content[]): Content[] {
  const curatedHistory: Content[] = [];
  let i = 0;

  while (i < length) {
    if (comprehensiveHistory[i].role === 'user') {
      curatedHistory.push(comprehensiveHistory[i]);
      i++;
    } else {
      // 모델 응답 그룹 검증
      const modelOutput: Content[] = [];
      let isValid = true;

      while (i < length && comprehensiveHistory[i].role === 'model') {
        modelOutput.push(comprehensiveHistory[i]);
        if (isValid && !isValidContent(comprehensiveHistory[i])) {
          isValid = false;
        }
        i++;
      }

      // 유효한 응답만 추가
      if (isValid) {
        curatedHistory.push(...modelOutput);
      }
    }
  }

  return curatedHistory;
}
```

## 요청 흐름 요약

```
사용자 입력
    ↓
[1] PartListUnion으로 변환
    ↓
[2] 시스템 프롬프트 생성
    ↓
[3] 도구 정의 수집
    ↓
[4] 히스토리 조회 (Curated)
    ↓
[5] GenerateContentConfig 구성
    ↓
[6] IDE 컨텍스트 추가 (해당 시)
    ↓
[7] 토큰 제한 체크
    ↓
[8] 히스토리 압축 (필요 시)
    ↓
[9] API 요청 생성 및 전송
    ↓
[10] 재시도 로직 적용
    ↓
API 응답 스트림 →
```

## 다음 장에서는

3장에서는 Gemini API의 응답을 어떻게 처리하는지, 스트림 청크를 어떻게 파싱하고 이벤트로 변환하는지 살펴보겠습니다.
