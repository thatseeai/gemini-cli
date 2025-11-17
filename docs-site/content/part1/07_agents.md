---
title: "7장. 에이전트 시스템 (Agent System)"
weight: 07
---

# 7장. 에이전트 시스템 (Agent System)

## 개요

에이전트 시스템은 특수화된 작업을 자율적으로 수행하는 서브 에이전트를 정의하고 실행할 수 있게 합니다. 각 에이전트는 자체 프롬프트, 모델 설정, 도구 접근 권한, 입출력 스키마를 가지며, 메인 대화 세션과 독립적으로 작동합니다.

**주요 파일**:
- `packages/core/src/agents/types.ts` - 에이전트 타입 정의
- `packages/core/src/agents/executor.ts` - 에이전트 실행 엔진
- `packages/core/src/agents/registry.ts` - 에이전트 레지스트리
- `packages/core/src/agents/codebase-investigator.ts` - 내장 에이전트 예제

## 에이전트 아키텍처

### 계층 구조

```
AgentDefinition (정의)
    ↓
AgentInvocation (인스턴스)
    ↓
AgentExecutor (실행 엔진)
    ↓
독립적인 대화 루프
```

에이전트는 도구 시스템과 유사한 빌더 패턴을 사용하지만, 더 복잡한 실행 모델을 가집니다.

## 핵심 타입

### 1. AgentDefinition

에이전트의 선언적 정의입니다.

```typescript
export interface AgentDefinition {
  /**
   * 에이전트 이름 (도구 이름으로 사용)
   */
  name: string;

  /**
   * 에이전트 설명
   */
  description: string;

  /**
   * 프롬프트 설정
   */
  promptConfig: {
    /**
     * 시스템 프롬프트 (선택적)
     */
    systemPrompt?: string;

    /**
     * 사용자 프롬프트 템플릿
     * 입력 파라미터로부터 생성
     */
    userPromptTemplate: string;
  };

  /**
   * 모델 설정
   */
  modelConfig: {
    /**
     * 사용할 모델
     */
    model?: string;

    /**
     * 온도 (0-2)
     */
    temperature?: number;

    /**
     * Thinking budget (tokens)
     */
    thinkingBudget?: number;
  };

  /**
   * 실행 설정
   */
  runConfig: {
    /**
     * 최대 턴 수
     */
    maxTurns?: number;

    /**
     * 타임아웃 (밀리초)
     */
    timeout?: number;
  };

  /**
   * 도구 접근 설정
   */
  toolConfig: {
    /**
     * 허용된 도구 목록 (undefined면 모두 허용)
     */
    allowedTools?: string[];

    /**
     * 제외할 도구 목록
     */
    excludedTools?: string[];

    /**
     * 에이전트 전용 커스텀 도구
     */
    customTools?: AnyDeclarativeTool[];
  };

  /**
   * 입력 스키마
   */
  inputConfig: {
    /**
     * JSON Schema 형식의 입력 파라미터
     */
    inputSchema: JSONSchema;

    /**
     * 필수 필드
     */
    required?: string[];
  };

  /**
   * 출력 설정
   */
  outputConfig: {
    /**
     * 출력 스키마 (구조화된 출력)
     */
    outputSchema?: JSONSchema;

    /**
     * 출력 검증 함수
     */
    validateOutput?: (output: unknown) => {
      valid: boolean;
      errors?: string[];
    };
  };
}
```

### 2. AgentInvocation

실행 가능한 에이전트 인스턴스입니다.

```typescript
export class AgentInvocation {
  constructor(
    private readonly definition: AgentDefinition,
    private readonly input: Record<string, unknown>,
    private readonly config: Config,
  ) {}

  /**
   * 에이전트 실행
   */
  async *execute(
    signal: AbortSignal,
  ): AsyncGenerator<AgentEvent> {
    const executor = new AgentExecutor(
      this.definition,
      this.config,
    );

    yield* executor.run(this.input, signal);
  }
}
```

### 3. AgentExecutor

에이전트 실행 엔진입니다.

```typescript
export class AgentExecutor {
  constructor(
    private readonly definition: AgentDefinition,
    private readonly config: Config,
  ) {}

  async *run(
    input: Record<string, unknown>,
    signal: AbortSignal,
  ): AsyncGenerator<AgentEvent> {
    // 1. 프롬프트 생성
    const userPrompt = this.generatePrompt(input);

    // 2. 도구 레지스트리 생성
    const toolRegistry = this.createToolRegistry();

    // 3. 클라이언트 생성
    const client = this.createClient(toolRegistry);

    // 4. 대화 루프 시작
    let turnCount = 0;
    const maxTurns = this.definition.runConfig.maxTurns || 100;

    while (turnCount < maxTurns && !signal.aborted) {
      const turn = await client.sendMessageStream(
        [{ text: userPrompt }],
        signal,
      );

      for await (const event of turn) {
        yield this.transformEvent(event);

        // 완료 도구 호출 감지
        if (this.isCompleteTool(event)) {
          const output = this.extractOutput(event);
          yield { type: 'output', value: output };
          return;
        }
      }

      turnCount++;
    }

    if (turnCount >= maxTurns) {
      yield { type: 'error', value: 'Max turns exceeded' };
    }
  }

  private generatePrompt(input: Record<string, unknown>): string {
    const template = this.definition.promptConfig.userPromptTemplate;
    // 템플릿 변수 치환
    return template.replace(/\{\{(\w+)\}\}/g, (_, key) => {
      return String(input[key] || '');
    });
  }

  private createToolRegistry(): ToolRegistry {
    const registry = new ToolRegistry();

    // 허용된 도구만 추가
    const { allowedTools, excludedTools, customTools } =
      this.definition.toolConfig;

    if (customTools) {
      customTools.forEach((tool) => registry.registerTool(tool));
    }

    // 전역 도구 필터링
    const globalTools = this.config.getToolRegistry().getAllTools();
    globalTools.forEach((tool) => {
      const name = tool.schema.name;
      if (allowedTools && !allowedTools.includes(name)) {
        return;
      }
      if (excludedTools && excludedTools.includes(name)) {
        return;
      }
      registry.registerTool(tool);
    });

    // 완료 도구 추가
    registry.registerTool(new CompleteTaskTool(this.definition.outputConfig));

    return registry;
  }
}
```

## 내장 에이전트: CodebaseInvestigatorAgent

코드베이스 분석을 전문으로 하는 내장 에이전트입니다.

**파일**: `packages/core/src/agents/codebase-investigator.ts`

### 정의

```typescript
export const CodebaseInvestigatorAgent: AgentDefinition = {
  name: 'codebase-investigator',
  description: 'Investigates and analyzes codebases to answer questions',

  promptConfig: {
    systemPrompt: `You are a specialized codebase analysis agent.

Your task is to thoroughly investigate the codebase to answer the user's question.

You have access to file system tools:
- read-file: Read file contents
- ls: List directory contents
- glob: Find files matching patterns
- grep: Search for text in files

When you have gathered sufficient information, call the 'complete_task' tool with your findings.

Be thorough and systematic in your investigation.`,

    userPromptTemplate: `{{question}}

Please investigate the codebase and provide a comprehensive answer.`,
  },

  modelConfig: {
    model: 'gemini-2.5-flash', // 빠른 모델 사용
    temperature: 0.7,
  },

  runConfig: {
    maxTurns: 50,
    timeout: 300000, // 5분
  },

  toolConfig: {
    // 읽기 전용 도구만 허용
    allowedTools: [
      'read-file',
      'ls',
      'glob',
      'grep',
      'rip-grep',
    ],
  },

  inputConfig: {
    inputSchema: {
      type: 'object',
      properties: {
        question: {
          type: 'string',
          description: 'The question about the codebase',
        },
        context: {
          type: 'string',
          description: 'Additional context for the investigation',
        },
      },
    },
    required: ['question'],
  },

  outputConfig: {
    outputSchema: {
      type: 'object',
      properties: {
        findings: {
          type: 'string',
          description: 'Main findings from the investigation',
        },
        relevantFiles: {
          type: 'array',
          items: { type: 'string' },
          description: 'List of relevant file paths',
        },
        explorationTrace: {
          type: 'array',
          items: {
            type: 'object',
            properties: {
              action: { type: 'string' },
              result: { type: 'string' },
            },
          },
          description: 'Trace of exploration steps',
        },
      },
      required: ['findings', 'relevantFiles'],
    },
  },
};
```

### 사용 예제

```typescript
// 에이전트 호출
const result = await runAgent(
  'codebase-investigator',
  {
    question: 'How does the authentication system work?',
    context: 'Focus on JWT token handling',
  },
  signal,
);

// 결과 처리
console.log('Findings:', result.findings);
console.log('Relevant files:', result.relevantFiles);
```

## Complete Task 도구

모든 에이전트에는 작업 완료를 위한 `complete_task` 도구가 자동으로 추가됩니다.

```typescript
export class CompleteTaskTool extends BaseDeclarativeTool<
  Record<string, unknown>,
  ToolResult,
  CompleteTaskInvocation
> {
  constructor(private readonly outputConfig: OutputConfig) {
    super();
  }

  schema = {
    name: 'complete_task',
    description: 'Mark the task as complete and return the final output',
    parametersJsonSchema: this.outputConfig.outputSchema || {
      type: 'object',
      properties: {
        result: {
          type: 'string',
          description: 'The result of the task',
        },
      },
    },
  };

  protected async buildCore(
    params: Record<string, unknown>,
  ): Promise<CompleteTaskInvocation> {
    // 출력 검증
    if (this.outputConfig.validateOutput) {
      const validation = this.outputConfig.validateOutput(params);
      if (!validation.valid) {
        throw new Error(
          `Invalid output: ${validation.errors?.join(', ')}`,
        );
      }
    }

    return new CompleteTaskInvocation(params);
  }
}

class CompleteTaskInvocation extends BaseToolInvocation<
  Record<string, unknown>,
  ToolResult
> {
  getDescription(): string {
    return 'Complete the task';
  }

  async execute(_signal: AbortSignal): Promise<ToolResult> {
    return {
      responseParts: [{ text: JSON.stringify(this.params, null, 2) }],
      resultDisplay: 'Task completed',
      metadata: {
        isComplete: true,
        output: this.params,
      },
    };
  }
}
```

## 에이전트 이벤트

에이전트 실행 중 발생하는 이벤트입니다.

```typescript
export type AgentEvent =
  | { type: 'thought'; value: ThoughtSummary }
  | { type: 'content'; value: string }
  | { type: 'tool_call'; value: ToolCallInfo }
  | { type: 'tool_result'; value: ToolResult }
  | { type: 'output'; value: Record<string, unknown> }
  | { type: 'error'; value: string };
```

## 에이전트를 도구로 래핑

에이전트는 도구처럼 호출될 수 있습니다.

```typescript
export class SubagentToolWrapper extends BaseDeclarativeTool<
  Record<string, unknown>,
  ToolResult,
  SubagentInvocation
> {
  constructor(private readonly agentDef: AgentDefinition) {
    super();
  }

  schema = {
    name: this.agentDef.name,
    description: this.agentDef.description,
    parametersJsonSchema: this.agentDef.inputConfig.inputSchema,
  };

  protected async buildCore(
    params: Record<string, unknown>,
  ): Promise<SubagentInvocation> {
    return new SubagentInvocation(this.agentDef, params, this.config);
  }
}

class SubagentInvocation extends BaseToolInvocation<
  Record<string, unknown>,
  ToolResult
> {
  getDescription(): string {
    return `Run agent: ${this.agentDef.name}`;
  }

  async execute(
    signal: AbortSignal,
    updateOutput?: (output: string) => void,
  ): Promise<ToolResult> {
    const invocation = new AgentInvocation(
      this.agentDef,
      this.params,
      this.config,
    );

    let finalOutput: Record<string, unknown> | undefined;

    for await (const event of invocation.execute(signal)) {
      // 이벤트를 사용자에게 스트리밍
      if (event.type === 'content' && updateOutput) {
        updateOutput(event.value);
      }

      if (event.type === 'output') {
        finalOutput = event.value;
      }
    }

    if (!finalOutput) {
      return {
        error: new Error('Agent did not produce output'),
        errorType: ToolErrorType.ExecutionError,
        responseParts: [{ text: 'Agent execution failed' }],
      };
    }

    return {
      responseParts: [{ text: JSON.stringify(finalOutput, null, 2) }],
      resultDisplay: `Agent ${this.agentDef.name} completed`,
    };
  }
}
```

## 커스텀 에이전트 작성

### 1. 에이전트 정의 생성

```typescript
export const MyCustomAgent: AgentDefinition = {
  name: 'my-custom-agent',
  description: 'Custom agent for specific task',

  promptConfig: {
    systemPrompt: 'You are a specialized agent for...',
    userPromptTemplate: 'Task: {{task}}\nContext: {{context}}',
  },

  modelConfig: {
    model: 'gemini-2.5-pro',
    temperature: 0.8,
  },

  runConfig: {
    maxTurns: 30,
  },

  toolConfig: {
    allowedTools: ['read-file', 'write-file'],
  },

  inputConfig: {
    inputSchema: {
      type: 'object',
      properties: {
        task: { type: 'string' },
        context: { type: 'string' },
      },
    },
    required: ['task'],
  },

  outputConfig: {
    outputSchema: {
      type: 'object',
      properties: {
        result: { type: 'string' },
        confidence: { type: 'number' },
      },
    },
  },
};
```

### 2. 에이전트 등록

```typescript
const agentRegistry = new AgentRegistry();
agentRegistry.registerAgent(MyCustomAgent);
```

### 3. 에이전트 실행

```typescript
const result = await agentRegistry.runAgent(
  'my-custom-agent',
  {
    task: 'Analyze the code',
    context: 'Focus on performance',
  },
  signal,
);
```

## 에이전트 vs 도구

| 특성 | 도구 | 에이전트 |
|-----|------|---------|
| 실행 모델 | 단일 실행 | 다중 턴 루프 |
| 자율성 | 없음 | 높음 |
| 도구 사용 | 불가 | 가능 |
| 프롬프트 | 없음 | 커스텀 시스템/사용자 프롬프트 |
| 출력 | 즉시 | 비동기 스트림 |
| 복잡도 | 낮음 | 높음 |

## 다음 장에서는

8장에서는 프롬프트 시스템을 살펴보며, 동적 시스템 프롬프트 생성과 컨텍스트 관리를 알아보겠습니다.
