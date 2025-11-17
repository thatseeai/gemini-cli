---
title: "6장. 도구 시스템 (Tool System)"
weight: 06
---

## 개요

Gemini CLI의 도구 시스템은 LLM이 실제 작업을 수행할 수 있도록 하는 핵심 메커니즘입니다. 파일 읽기/쓰기, 셸 명령 실행, 웹 검색 등 20개 이상의 내장 도구를 제공하며, MCP (Model Context Protocol)를 통해 외부 도구와도 통합할 수 있습니다.

**주요 파일**:
- `packages/core/src/tools/tools.ts` - 도구 인터페이스 정의
- `packages/core/src/tools/tool-registry.ts` - 도구 레지스트리
- `packages/core/src/tools/` - 개별 도구 구현

## 아키텍처

### 빌더 패턴 (Builder Pattern)

Gemini CLI의 도구 시스템은 빌더 패턴을 사용합니다:

```
DeclarativeTool (빌더)
    ↓ build(params)
ToolInvocation (실행 가능 인스턴스)
    ↓ execute()
ToolResult (결과)
```

이 패턴은 다음과 같은 이점을 제공합니다:
- 도구 선언과 실행의 분리
- 파라미터 검증을 빌드 단계에서 수행
- 재사용 가능한 도구 정의
- 유연한 확장성

## 핵심 인터페이스

### 1. ToolInvocation

실행 가능한 도구 인스턴스를 나타냅니다.

```typescript
export interface ToolInvocation<
  TParams extends object,
  TResult extends ToolResult,
> {
  /**
   * 검증된 파라미터
   */
  params: TParams;

  /**
   * 도구 실행 전 설명 제공
   * @returns 도구가 수행할 작업의 마크다운 설명
   */
  getDescription(): string;

  /**
   * 도구가 영향을 미칠 파일 시스템 경로
   * @returns 경로 목록
   */
  toolLocations(): ToolLocation[];

  /**
   * 실행 전 확인이 필요한지 결정
   * @returns 확인 세부사항 또는 false (확인 불필요)
   */
  shouldConfirmExecute(
    abortSignal: AbortSignal,
  ): Promise<ToolCallConfirmationDetails | false>;

  /**
   * 도구 실행
   * @param signal 취소 시그널
   * @param updateOutput 출력 스트리밍 콜백
   * @returns 실행 결과
   */
  execute(
    signal: AbortSignal,
    updateOutput?: (output: string | AnsiOutput) => void,
    shellExecutionConfig?: ShellExecutionConfig,
  ): Promise<TResult>;
}
```

### 2. BaseToolInvocation

`ToolInvocation`의 편리한 기본 구현입니다.

```typescript
export abstract class BaseToolInvocation<
  TParams extends object,
  TResult extends ToolResult,
> implements ToolInvocation<TParams, TResult>
{
  constructor(
    readonly params: TParams,
    protected readonly messageBus?: MessageBus,
    readonly _toolName?: string,
    readonly _toolDisplayName?: string,
    readonly _serverName?: string,
  ) {}

  abstract getDescription(): string;

  toolLocations(): ToolLocation[] {
    return [];
  }

  async shouldConfirmExecute(
    abortSignal: AbortSignal,
  ): Promise<ToolCallConfirmationDetails | false> {
    if (this.messageBus) {
      const decision = await this.getMessageBusDecision(abortSignal);

      if (decision === 'ALLOW') {
        return false; // 확인 불필요
      }

      if (decision === 'DENY') {
        throw new Error(`Tool execution denied by policy.`);
      }

      if (decision === 'ASK_USER') {
        return this.getConfirmationDetails(abortSignal);
      }
    }
    return this.getConfirmationDetails(abortSignal);
  }

  protected async getConfirmationDetails(
    _abortSignal: AbortSignal,
  ): Promise<ToolCallConfirmationDetails | false> {
    if (!this.messageBus) {
      return false;
    }

    const confirmationDetails: ToolCallConfirmationDetails = {
      type: 'info',
      title: `Confirm: ${this._toolDisplayName || this._toolName}`,
      prompt: this.getDescription(),
      onConfirm: async (outcome: ToolConfirmationOutcome) => {
        if (outcome === ToolConfirmationOutcome.ProceedAlways) {
          if (this.messageBus && this._toolName) {
            this.messageBus.publish({
              type: MessageBusType.UPDATE_POLICY,
              toolName: this._toolName,
            });
          }
        }
      },
    };
    return confirmationDetails;
  }

  abstract execute(
    signal: AbortSignal,
    updateOutput?: (output: string | AnsiOutput) => void,
    shellExecutionConfig?: ShellExecutionConfig,
  ): Promise<TResult>;
}
```

### 3. DeclarativeTool

도구 빌더 인터페이스입니다.

```typescript
export interface DeclarativeTool<
  TParams extends object,
  TResult extends ToolResult,
  TInvocation extends ToolInvocation<TParams, TResult>,
> {
  /**
   * 도구 스키마 (Gemini API 형식)
   */
  schema: {
    name: string;
    description: string;
    parameters?: JSONSchema;
    parametersJsonSchema?: JSONSchema;
  };

  /**
   * 파라미터로부터 ToolInvocation 생성
   * @param params 도구 파라미터
   * @returns ToolInvocation 인스턴스
   */
  build(params: Record<string, unknown>): Promise<TInvocation>;
}
```

### 4. BaseDeclarativeTool

`DeclarativeTool`의 기본 구현으로, 파라미터 검증을 포함합니다.

```typescript
export abstract class BaseDeclarativeTool<
  TParams extends object,
  TResult extends ToolResult,
  TInvocation extends ToolInvocation<TParams, TResult>,
> implements DeclarativeTool<TParams, TResult, TInvocation>
{
  abstract schema: {
    name: string;
    description: string;
    parameters?: JSONSchema;
    parametersJsonSchema?: JSONSchema;
  };

  protected messageBus?: MessageBus;

  async build(params: Record<string, unknown>): Promise<TInvocation> {
    // 스키마 검증
    const validator = new SchemaValidator(
      this.schema.parametersJsonSchema || this.schema.parameters,
    );
    const validationResult = validator.validate(params);

    if (!validationResult.valid) {
      const errorMessages = validationResult.errors
        .map((err) => `  - ${err.path}: ${err.message}`)
        .join('\n');
      throw new Error(
        `Invalid parameters for tool "${this.schema.name}":\n${errorMessages}`,
      );
    }

    return this.buildCore(params as TParams);
  }

  protected abstract buildCore(params: TParams): Promise<TInvocation>;
}
```

## 내장 도구

Gemini CLI는 20개 이상의 내장 도구를 제공합니다.

### 파일 시스템 도구

| 도구 이름 | 설명 | 파일 |
|---------|------|------|
| `read-file` | 파일 읽기 | `read-file.ts` |
| `write-file` | 파일 쓰기 | `write-file.ts` |
| `edit` | 파일 편집 (문자열 치환) | `edit.ts` |
| `smart-edit` | AI 기반 스마트 편집 | `smart-edit.ts` |
| `ls` | 디렉토리 목록 조회 | `ls.ts` |
| `glob` | 패턴 매칭 파일 검색 | `glob.ts` |
| `grep` | 파일 내용 검색 | `grep.ts` |
| `rip-grep` | 빠른 파일 검색 (ripgrep) | `rip-grep.ts` |

### 실행 도구

| 도구 이름 | 설명 | 파일 |
|---------|------|------|
| `shell` | 셸 명령 실행 | `shell.ts` |
| `terminal` | 터미널 세션 관리 | `terminal.ts` |

### 정보 도구

| 도구 이름 | 설명 | 파일 |
|---------|------|------|
| `memory` | 사용자 메모리 저장 | `memory.ts` |
| `write-todos` | TODO 리스트 작성 | `write-todos.ts` |

### 웹 도구

| 도구 이름 | 설명 | 파일 |
|---------|------|------|
| `web-fetch` | 웹 페이지 가져오기 | `web-fetch.ts` |
| `web-search` | 웹 검색 | `web-search.ts` |

### IDE 통합 도구

| 도구 이름 | 설명 | 파일 |
|---------|------|------|
| `apply-diff` | VS Code에 diff 적용 | `apply-diff.ts` |

### MCP 도구

| 도구 이름 | 설명 | 파일 |
|---------|------|------|
| `mcp-tool` | MCP 서버 도구 호출 | `mcp-tool.ts` |
| `mcp-client-manager` | MCP 클라이언트 관리 | `mcp-client-manager.ts` |

### 에이전트 도구

| 도구 이름 | 설명 | 파일 |
|---------|------|------|
| `codebase-investigator` | 코드베이스 분석 에이전트 | (agents/) |

## 도구 구현 예제

### 1. Read File 도구

**파일**: `packages/core/src/tools/read-file.ts`

```typescript
interface ReadFileParams {
  file_path: string;
  offset?: number;
  limit?: number;
}

class ReadFileInvocation extends BaseToolInvocation<
  ReadFileParams,
  ToolResult
> {
  getDescription(): string {
    const { file_path, offset, limit } = this.params;
    if (offset !== undefined || limit !== undefined) {
      return `Read lines ${offset || 0} to ${
        (offset || 0) + (limit || 'end')
      } from file: ${file_path}`;
    }
    return `Read file: ${file_path}`;
  }

  toolLocations(): ToolLocation[] {
    return [
      {
        type: ToolLocationType.FilePath,
        value: this.params.file_path,
      },
    ];
  }

  async execute(signal: AbortSignal): Promise<ToolResult> {
    const { file_path, offset = 0, limit } = this.params;

    // 파일 읽기
    const content = await fs.promises.readFile(
      file_path,
      { encoding: 'utf-8' },
    );

    const lines = content.split('\n');
    const selectedLines = limit
      ? lines.slice(offset, offset + limit)
      : lines.slice(offset);

    // 라인 넘버와 함께 포맷팅
    const numberedLines = selectedLines
      .map((line, idx) => `${offset + idx + 1}\t${line}`)
      .join('\n');

    return {
      responseParts: [{ text: numberedLines }],
      resultDisplay: `Read ${selectedLines.length} lines from ${file_path}`,
    };
  }
}

export class ReadFileTool extends BaseDeclarativeTool<
  ReadFileParams,
  ToolResult,
  ReadFileInvocation
> {
  schema = {
    name: 'read-file',
    description: 'Read the contents of a file',
    parametersJsonSchema: {
      type: 'object',
      properties: {
        file_path: {
          type: 'string',
          description: 'The path to the file to read',
        },
        offset: {
          type: 'number',
          description: 'Line number to start reading from (0-indexed)',
        },
        limit: {
          type: 'number',
          description: 'Maximum number of lines to read',
        },
      },
      required: ['file_path'],
    },
  };

  protected async buildCore(
    params: ReadFileParams,
  ): Promise<ReadFileInvocation> {
    return new ReadFileInvocation(params, this.messageBus);
  }
}
```

### 2. Shell 도구

**파일**: `packages/core/src/tools/shell.ts`

```typescript
interface ShellParams {
  command: string;
  description?: string;
  background?: boolean;
}

class ShellInvocation extends BaseToolInvocation<ShellParams, ToolResult> {
  getDescription(): string {
    const { command, description } = this.params;
    if (description) {
      return `${description}\n\`\`\`bash\n${command}\n\`\`\``;
    }
    return `Execute shell command:\n\`\`\`bash\n${command}\n\`\`\``;
  }

  protected async getConfirmationDetails(
    _abortSignal: AbortSignal,
  ): Promise<ToolCallConfirmationDetails | false> {
    return {
      type: 'warning',
      title: 'Execute Shell Command',
      prompt: this.getDescription(),
      onConfirm: async (outcome) => {
        if (outcome === ToolConfirmationOutcome.ProceedAlways) {
          // 정책 업데이트
        }
      },
    };
  }

  async execute(
    signal: AbortSignal,
    updateOutput?: (output: string) => void,
    shellExecutionConfig?: ShellExecutionConfig,
  ): Promise<ToolResult> {
    const { command, background } = this.params;

    const result = await shellExecutionService.execute(
      command,
      signal,
      updateOutput,
      background,
      shellExecutionConfig,
    );

    if (result.error) {
      return {
        error: result.error,
        errorType: ToolErrorType.ExecutionError,
        responseParts: [
          {
            text: `Command failed with exit code ${result.exitCode}:\n${result.stderr}`,
          },
        ],
      };
    }

    return {
      responseParts: [
        {
          text: result.stdout || '(No output)',
        },
      ],
      resultDisplay: `Executed: ${command}`,
    };
  }
}

export class ShellTool extends BaseDeclarativeTool<
  ShellParams,
  ToolResult,
  ShellInvocation
> {
  schema = {
    name: 'shell',
    description: 'Execute a shell command and return its output',
    parametersJsonSchema: {
      type: 'object',
      properties: {
        command: {
          type: 'string',
          description: 'The shell command to execute',
        },
        description: {
          type: 'string',
          description:
            'Brief description of what the command does (5-10 words)',
        },
        background: {
          type: 'boolean',
          description:
            'Run command in background. Use for long-running processes.',
        },
      },
      required: ['command'],
    },
  };

  protected async buildCore(params: ShellParams): Promise<ShellInvocation> {
    return new ShellInvocation(params, this.messageBus, 'shell', 'Shell');
  }
}
```

## Tool Registry

도구 레지스트리는 모든 도구를 관리하고 Gemini API에 전달할 함수 선언을 생성합니다.

**파일**: `packages/core/src/tools/tool-registry.ts`

### 주요 기능

```typescript
export class ToolRegistry {
  private tools = new Map<string, AnyDeclarativeTool>();

  /**
   * 도구 등록
   */
  registerTool(tool: AnyDeclarativeTool): void {
    this.tools.set(tool.schema.name, tool);
  }

  /**
   * 도구 조회
   */
  getTool(name: string): AnyDeclarativeTool | undefined {
    return this.tools.get(name);
  }

  /**
   * 모든 도구 이름 조회
   */
  getAllToolNames(): string[] {
    return Array.from(this.tools.keys());
  }

  /**
   * Gemini API용 함수 선언 생성
   */
  getFunctionDeclarations(): FunctionDeclaration[] {
    return Array.from(this.tools.values()).map((tool) => ({
      name: tool.schema.name,
      description: tool.schema.description,
      parameters: tool.schema.parametersJsonSchema || tool.schema.parameters,
    }));
  }

  /**
   * 도구 빌드 및 실행
   */
  async buildAndExecute(
    name: string,
    params: Record<string, unknown>,
    signal: AbortSignal,
  ): Promise<ToolResult> {
    const tool = this.getTool(name);
    if (!tool) {
      throw new Error(`Tool not found: ${name}`);
    }

    const invocation = await tool.build(params);
    return await invocation.execute(signal);
  }
}
```

## 정책 엔진 (Policy Engine)

도구 실행 전에 정책 엔진이 허용/거부/사용자 확인을 결정합니다.

### 정책 결정 흐름

```
도구 호출 요청
    ↓
MessageBus → Policy Engine
    ↓
정책 평가 (TOML 규칙)
    ├─→ ALLOW: 즉시 실행
    ├─→ DENY: 에러 발생
    └─→ ASK_USER: 사용자 확인 요청
        ↓
    사용자 선택
        ├─→ Proceed: 실행
        ├─→ Proceed Always: 실행 + 정책 업데이트
        └─→ Cancel: 취소
```

### 정책 파일 예제

**파일**: `.gemini-cli/policy.toml`

```toml
# 파일 읽기 자동 허용
[[rules]]
tool = "read-file"
decision = "allow"

# 파일 쓰기는 항상 확인
[[rules]]
tool = "write-file"
decision = "ask"

# 셸 명령은 특정 패턴만 허용
[[rules]]
tool = "shell"
pattern = "^(ls|cat|grep).*"
decision = "allow"

[[rules]]
tool = "shell"
decision = "ask"
```

## MCP (Model Context Protocol) 통합

MCP를 통해 외부 도구와 통합할 수 있습니다.

### MCP Tool 래퍼

```typescript
export class DiscoveredMCPTool extends BaseDeclarativeTool<
  Record<string, unknown>,
  ToolResult,
  MCPToolInvocation
> {
  constructor(
    private readonly mcpClient: MCPClient,
    private readonly toolDef: MCPToolDefinition,
  ) {
    super();
  }

  schema = {
    name: `mcp_${this.toolDef.name}`,
    description: this.toolDef.description,
    parametersJsonSchema: this.toolDef.inputSchema,
  };

  protected async buildCore(
    params: Record<string, unknown>,
  ): Promise<MCPToolInvocation> {
    return new MCPToolInvocation(
      this.mcpClient,
      this.toolDef.name,
      params,
      this.messageBus,
    );
  }
}

class MCPToolInvocation extends BaseToolInvocation<
  Record<string, unknown>,
  ToolResult
> {
  async execute(signal: AbortSignal): Promise<ToolResult> {
    const result = await this.mcpClient.callTool(
      this.toolName,
      this.params,
      signal,
    );

    return {
      responseParts: result.content.map((item) => ({
        text: item.text || JSON.stringify(item),
      })),
      resultDisplay: `MCP tool ${this.toolName} completed`,
    };
  }
}
```

## 커스텀 도구 작성

### 1. ToolInvocation 구현

```typescript
interface MyToolParams {
  input: string;
}

class MyToolInvocation extends BaseToolInvocation<MyToolParams, ToolResult> {
  getDescription(): string {
    return `Process input: ${this.params.input}`;
  }

  async execute(signal: AbortSignal): Promise<ToolResult> {
    // 도구 로직 구현
    const result = await processInput(this.params.input);

    return {
      responseParts: [{ text: result }],
      resultDisplay: 'Processing completed',
    };
  }
}
```

### 2. DeclarativeTool 구현

```typescript
export class MyTool extends BaseDeclarativeTool<
  MyToolParams,
  ToolResult,
  MyToolInvocation
> {
  schema = {
    name: 'my-tool',
    description: 'Description of my custom tool',
    parametersJsonSchema: {
      type: 'object',
      properties: {
        input: {
          type: 'string',
          description: 'Input to process',
        },
      },
      required: ['input'],
    },
  };

  protected async buildCore(params: MyToolParams): Promise<MyToolInvocation> {
    return new MyToolInvocation(params, this.messageBus);
  }
}
```

### 3. 도구 등록

```typescript
const toolRegistry = new ToolRegistry();
toolRegistry.registerTool(new MyTool());
```

## 다음 장에서는

7장에서는 에이전트 시스템을 살펴보며, 전문화된 작업을 위한 에이전트 구현을 알아보겠습니다.
