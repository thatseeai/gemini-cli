---
title: "4장. 도구 실행 (Tool Execution)"
weight: 04
---

# 4장. 도구 실행 (Tool Execution)

## 개요

이 장에서는 LLM이 요청한 도구 호출을 어떻게 실행하고 결과를 다시 LLM에 전달하는지 전체 프로세스를 살펴봅니다. 도구 호출 요청부터 결과 반환까지의 전체 흐름을 실제 코드와 예제로 분석합니다.

## 도구 실행 흐름

```
LLM Function Call 응답
    ↓
Turn: ToolCallRequest 이벤트 생성
    ↓
CLI: ToolScheduler로 전달
    ↓
정책 엔진: ALLOW/DENY/ASK_USER
    ↓
사용자 확인 (필요 시)
    ↓
ToolRegistry: 도구 조회
    ↓
Tool.build(params): ToolInvocation 생성
    ↓
ToolInvocation.execute(): 실행
    ↓
ToolResult 생성
    ↓
FunctionResponse로 변환
    ↓
다음 턴에 FunctionResponse 포함하여 LLM 호출
    ↓
LLM 최종 응답
```

## 단계별 상세 분석

### 1단계: Function Call 수신

**API 응답**:
```json
{
  "candidates": [{
    "content": {
      "role": "model",
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

### 2단계: ToolCallRequest 이벤트 생성

**파일**: `packages/core/src/core/turn.ts`

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

### 3단계: Tool Scheduler에서 처리

**파일**: `packages/cli/src/hooks/useReactToolScheduler.ts`

```typescript
const scheduleToolCall = async (request: ToolCallRequestInfo) => {
  const toolRegistry = config.getToolRegistry();
  const tool = toolRegistry.getTool(request.name);

  if (!tool) {
    // 도구를 찾을 수 없음
    return {
      callId: request.callId,
      responseParts: [{ text: `Tool not found: ${request.name}` }],
      error: new Error(`Tool not found: ${request.name}`),
      errorType: ToolErrorType.NotFound,
    };
  }

  try {
    // 1. Tool Invocation 생성
    const invocation = await tool.build(request.args);

    // 2. 확인 필요 여부 체크
    const confirmationDetails = await invocation.shouldConfirmExecute(signal);

    if (confirmationDetails) {
      // 사용자 확인 요청
      const userDecision = await askUser(confirmationDetails);

      if (userDecision === 'deny') {
        throw new Error('Tool execution denied by user');
      }
    }

    // 3. 도구 실행
    const result = await invocation.execute(signal, updateOutput);

    // 4. ToolCallResponse 이벤트 생성
    return {
      callId: request.callId,
      responseParts: result.responseParts,
      resultDisplay: result.resultDisplay,
      error: result.error,
      errorType: result.errorType,
    };
  } catch (error) {
    return {
      callId: request.callId,
      responseParts: [{ text: `Error: ${getErrorMessage(error)}` }],
      error: error as Error,
      errorType: ToolErrorType.ExecutionError,
    };
  }
};
```

### 4단계: 정책 엔진 결정

**파일**: `packages/core/src/tools/tools.ts`

```typescript
async shouldConfirmExecute(
  abortSignal: AbortSignal,
): Promise<ToolCallConfirmationDetails | false> {
  if (this.messageBus) {
    const decision = await this.getMessageBusDecision(abortSignal);

    if (decision === 'ALLOW') {
      return false; // 확인 불필요, 즉시 실행
    }

    if (decision === 'DENY') {
      throw new Error('Tool execution denied by policy.');
    }

    if (decision === 'ASK_USER') {
      return this.getConfirmationDetails(abortSignal);
    }
  }

  return this.getConfirmationDetails(abortSignal);
}
```

**정책 평가 예제**:

```toml
# .gemini-cli/policy.toml

# read-file은 자동 허용
[[rules]]
tool = "read-file"
decision = "allow"

# write-file은 항상 확인
[[rules]]
tool = "write-file"
decision = "ask"

# shell은 특정 패턴만 자동 허용
[[rules]]
tool = "shell"
pattern = "^(ls|cat|grep).*"
decision = "allow"

[[rules]]
tool = "shell"
decision = "ask"
```

### 5단계: 도구 실행

**read-file 도구 실행 예제**:

**파일**: `packages/core/src/tools/read-file.ts`

```typescript
async execute(signal: AbortSignal): Promise<ToolResult> {
  const { file_path, offset = 0, limit } = this.params;

  try {
    // 파일 읽기
    const content = await fs.promises.readFile(file_path, {
      encoding: 'utf-8',
    });

    // 라인 분할
    const lines = content.split('\n');

    // offset과 limit 적용
    const selectedLines = limit
      ? lines.slice(offset, offset + limit)
      : lines.slice(offset);

    // 라인 넘버링
    const numberedLines = selectedLines
      .map((line, idx) => `${offset + idx + 1}\t${line}`)
      .join('\n');

    return {
      responseParts: [{ text: numberedLines }],
      resultDisplay: `Read ${selectedLines.length} lines from ${file_path}`,
    };
  } catch (error) {
    return {
      error: error as Error,
      errorType: ToolErrorType.ExecutionError,
      responseParts: [
        { text: `Failed to read file: ${getErrorMessage(error)}` },
      ],
    };
  }
}
```

**실행 결과**:
```typescript
{
  responseParts: [
    {
      text: `1\t# My Project
2\t
3\tThis is a sample README file.
4\t
5\t## Installation
6\t
7\t\`\`\`bash
8\tnpm install
9\t\`\`\``
    }
  ],
  resultDisplay: "Read 9 lines from README.md"
}
```

### 6단계: FunctionResponse 생성

**파일**: `packages/cli/src/hooks/useGeminiStream.ts`

```typescript
// ToolResult를 FunctionResponse로 변환
const functionResponse: FunctionResponse = {
  name: request.name,
  response: {
    content: result.responseParts[0]?.text || '',
    error: result.error?.message,
    resultDisplay: result.resultDisplay,
  },
};

// 히스토리에 추가
client.addHistory({
  role: 'user',
  parts: [{ functionResponse }],
});
```

**생성된 FunctionResponse**:
```typescript
{
  role: 'user',
  parts: [
    {
      functionResponse: {
        name: 'read-file',
        response: {
          content: `1\t# My Project
2\t
3\tThis is a sample README file.
4\t
5\t## Installation
6\t
7\t\`\`\`bash
8\tnpm install
9\t\`\`\``,
          resultDisplay: 'Read 9 lines from README.md'
        }
      }
    }
  ]
}
```

### 7단계: 다음 턴 시작

FunctionResponse를 포함하여 다음 API 요청을 보냅니다.

**API 요청**:
```http
POST /v1beta/models/gemini-2.5-pro:streamGenerateContent

Body:
{
  "contents": [
    // 이전 사용자 메시지
    {
      "role": "user",
      "parts": [{ "text": "README.md 파일을 읽어줘" }]
    },
    // 이전 모델 응답 (Function Call)
    {
      "role": "model",
      "parts": [
        {
          "functionCall": {
            "name": "read-file",
            "args": { "file_path": "README.md" }
          }
        }
      ]
    },
    // Function Response (새로 추가)
    {
      "role": "user",
      "parts": [
        {
          "functionResponse": {
            "name": "read-file",
            "response": {
              "content": "1\t# My Project\n...",
              "resultDisplay": "Read 9 lines from README.md"
            }
          }
        }
      ]
    }
  ],
  // ... systemInstruction, tools, etc.
}
```

### 8단계: LLM 최종 응답

**API 응답**:
```json
{
  "candidates": [{
    "content": {
      "role": "model",
      "parts": [{
        "text": "README.md 파일의 내용은 다음과 같습니다:\n\n이 프로젝트는 'My Project'라는 이름으로, 샘플 프로젝트입니다. 설치는 `npm install` 명령으로 할 수 있습니다."
      }]
    },
    "finishReason": "STOP"
  }],
  "usageMetadata": {
    "promptTokenCount": 1650,
    "candidatesTokenCount": 75,
    "totalTokenCount": 1725
  }
}
```

## 복잡한 시나리오

### 다중 도구 호출

LLM이 여러 도구를 동시에 호출할 수 있습니다.

**API 응답**:
```json
{
  "candidates": [{
    "content": {
      "parts": [
        {
          "functionCall": {
            "name": "read-file",
            "args": { "file_path": "package.json" }
          }
        },
        {
          "functionCall": {
            "name": "read-file",
            "args": { "file_path": "tsconfig.json" }
          }
        }
      ]
    }
  }]
}
```

**처리**:
```typescript
// 모든 도구 호출을 수집
const toolCalls: ToolCallRequestInfo[] = [];
for (const fnCall of functionCalls) {
  const event = this.handlePendingFunctionCall(fnCall);
  if (event) {
    toolCalls.push(event.value);
    yield event;
  }
}

// 병렬 실행
const results = await Promise.all(
  toolCalls.map((call) => scheduleToolCall(call)),
);

// 모든 결과를 FunctionResponse로 변환하여 다음 턴에 포함
const functionResponses = results.map((result) => ({
  functionResponse: {
    name: result.name,
    response: result.response,
  },
}));

client.addHistory({
  role: 'user',
  parts: functionResponses,
});
```

### 에러 처리

**도구 실행 실패 예제**:

```typescript
// 파일을 찾을 수 없는 경우
{
  responseParts: [
    { text: "Error: ENOENT: no such file or directory, open 'nonexistent.txt'" }
  ],
  error: Error("ENOENT: no such file or directory"),
  errorType: ToolErrorType.ExecutionError,
  resultDisplay: undefined
}
```

**FunctionResponse**:
```typescript
{
  role: 'user',
  parts: [
    {
      functionResponse: {
        name: 'read-file',
        response: {
          error: "ENOENT: no such file or directory, open 'nonexistent.txt'",
          content: "Error: ENOENT: no such file or directory, open 'nonexistent.txt'"
        }
      }
    }
  ]
}
```

**LLM의 에러 처리**:
```
"죄송합니다. 'nonexistent.txt' 파일을 찾을 수 없습니다. 파일 경로를 확인해 주시겠습니까?"
```

### 스트리밍 출력

일부 도구(예: shell)는 실행 중 출력을 스트리밍합니다.

```typescript
async execute(
  signal: AbortSignal,
  updateOutput?: (output: string) => void,
): Promise<ToolResult> {
  const { command } = this.params;

  const result = await shellExecutionService.execute(
    command,
    signal,
    updateOutput, // 실시간 출력 콜백
  );

  return {
    responseParts: [{ text: result.stdout }],
    resultDisplay: `Executed: ${command}`,
  };
}
```

**스트리밍 예제**:
```
$ npm test

> Running tests...
✓ test/unit/utils.test.ts (5ms)
✓ test/unit/parser.test.ts (3ms)
✓ test/integration/api.test.ts (120ms)

Tests passed: 3/3
```

## 도구 확인 UI

### 확인 프롬프트 예제

**일반 도구** (read-file):
```
┌─────────────────────────────────────────┐
│ Tool: read-file                         │
│                                         │
│ Read file: README.md                    │
│                                         │
│ [Proceed] [Deny] [Proceed Always]      │
└─────────────────────────────────────────┘
```

**위험한 도구** (shell):
```
┌─────────────────────────────────────────┐
│ ⚠️  Shell Command                       │
│                                         │
│ Execute shell command:                  │
│ ```bash                                 │
│ rm -rf dist/                            │
│ ```                                     │
│                                         │
│ [Proceed] [Deny] [Proceed Always]      │
└─────────────────────────────────────────┘
```

### 확인 결과 처리

```typescript
enum ToolConfirmationOutcome {
  Proceed = 'proceed',           // 한 번만 실행
  ProceedAlways = 'proceed_always', // 정책 업데이트하여 항상 허용
  Deny = 'deny',                 // 거부
}

// 사용자 선택 처리
if (outcome === ToolConfirmationOutcome.ProceedAlways) {
  // 정책 업데이트
  messageBus.publish({
    type: MessageBusType.UPDATE_POLICY,
    toolName: this._toolName,
  });
}
```

## 완전한 도구 실행 예제

### Shell 도구 실행

**1. LLM Function Call**:
```json
{
  "functionCall": {
    "name": "shell",
    "args": {
      "command": "npm run build",
      "description": "Build the project"
    }
  }
}
```

**2. 사용자 확인**:
```
┌─────────────────────────────────────────┐
│ Shell Command                           │
│                                         │
│ Build the project                       │
│ ```bash                                 │
│ npm run build                           │
│ ```                                     │
│                                         │
│ [Proceed] [Deny] [Proceed Always]      │
└─────────────────────────────────────────┘

→ Proceed
```

**3. 실행 (스트리밍 출력)**:
```
> building...
✓ compiled src/index.ts
✓ compiled src/utils.ts
✓ build complete in 2.3s
```

**4. FunctionResponse**:
```typescript
{
  role: 'user',
  parts: [
    {
      functionResponse: {
        name: 'shell',
        response: {
          content: "> building...\n✓ compiled src/index.ts\n✓ compiled src/utils.ts\n✓ build complete in 2.3s",
          resultDisplay: "Executed: npm run build"
        }
      }
    }
  ]
}
```

**5. LLM 최종 응답**:
```
빌드가 성공적으로 완료되었습니다. src/index.ts와 src/utils.ts가 컴파일되었으며, 총 2.3초가 걸렸습니다.
```

## 도구 실행 최적화

### 1. 병렬 실행

```typescript
// 독립적인 도구 호출을 병렬로 실행
const results = await Promise.all(
  toolCalls.map((call) => scheduleToolCall(call)),
);
```

### 2. 타임아웃

```typescript
const result = await Promise.race([
  invocation.execute(signal),
  timeout(30000), // 30초 타임아웃
]);
```

### 3. 취소 지원

```typescript
const abortController = new AbortController();

// 사용자가 취소 버튼 클릭 시
onCancel(() => abortController.abort());

// 도구 실행 시 signal 전달
const result = await invocation.execute(abortController.signal);
```

## 도구 실행 흐름 요약

```
Function Call 수신
    ↓
ToolCallRequest 이벤트 생성
    ↓
Tool Scheduler
    ↓
Tool Registry: 도구 조회
    ↓
Tool.build(params)
    ├─ 파라미터 검증 (JSON Schema)
    └─ ToolInvocation 생성
        ↓
    shouldConfirmExecute()
        ├─ 정책 엔진: ALLOW → 즉시 실행
        ├─ 정책 엔진: DENY → 에러
        └─ 정책 엔진: ASK_USER → 사용자 확인
            ↓
        execute(signal, updateOutput)
            ├─ 실행 로직
            ├─ 스트리밍 출력 (선택)
            └─ ToolResult 반환
                ↓
            FunctionResponse 생성
                ↓
            다음 턴에 포함하여 LLM 호출
                ↓
            LLM 최종 응답
```

## 다음 장에서는

5장에서는 다양한 실제 사용 시나리오를 살펴보며, 지금까지 배운 모든 개념이 어떻게 통합되어 작동하는지 엔드투엔드로 분석하겠습니다.
