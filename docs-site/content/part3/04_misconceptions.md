---
title: "4장. 흔한 오해와 명확화"
weight: 04
---

## 개요

Gemini CLI를 사용하다 보면 LLM과 Agent의 역할 구분에 대해 흔히 오해하는 부분들이 있습니다. 이 장에서는 가장 흔한 오해들을 다루고, 실제로는 어떻게 동작하는지 명확히 설명합니다.

## 오해 1: "LLM이 직접 파일을 읽는다"

### 흔한 생각

```
사용자: "README.md 파일 읽어줘"

사용자 생각:
"Gemini API가 내 컴퓨터의 파일을 직접 읽는구나"
```

### 실제 동작

```
┌─────────────────────────────────────┐
│ 1. 사용자 요청                       │
│    "README.md 파일 읽어줘"           │
└─────────────────────────────────────┘
         ↓
┌─────────────────────────────────────┐
│ 2. Agent → Gemini API               │
│    사용자 메시지 전송                │
└─────────────────────────────────────┘
         ↓
┌─────────────────────────────────────┐
│ 3. LLM 판단                          │
│    "read-file 도구를 호출해야겠다"   │
│    도구 호출 응답 생성               │
└─────────────────────────────────────┘
         ↓
┌─────────────────────────────────────┐
│ 4. Agent가 도구 호출 감지            │
│    read-file 도구 실행               │
│    fs.readFile('README.md')          │
└─────────────────────────────────────┘
         ↓
┌─────────────────────────────────────┐
│ 5. Agent → Gemini API               │
│    파일 내용을 도구 결과로 전송      │
└─────────────────────────────────────┘
         ↓
┌─────────────────────────────────────┐
│ 6. LLM이 내용 분석 및 응답           │
│    "README 파일의 내용은..."         │
└─────────────────────────────────────┘
```

### 명확화

**LLM은 절대 직접 파일에 접근하지 않습니다.**

- **LLM의 역할**: "어떤 파일을 읽을지" 결정
- **Agent의 역할**: 실제로 파일 시스템에서 파일 읽기
- **LLM이 보는 것**: Agent가 보내준 파일 내용 (문자열)

### 코드로 보기

```typescript
// packages/core/src/tools/read-file.ts
export class ReadFileInvocation extends ToolInvocation {
  async execute(signal: AbortSignal): Promise<ToolResult> {
    // Agent가 실제로 파일을 읽음
    const content = await fs.promises.readFile(
      this.params.file_path,
      'utf-8'
    );

    // LLM에게 전달할 결과 생성
    return {
      tool: 'read-file',
      result: content  // LLM은 이 문자열만 봄
    };
  }
}
```

## 오해 2: "Agent가 알아서 판단한다"

### 흔한 생각

```
사용자: "프로젝트를 분석해줘"

사용자 생각:
"Gemini CLI가 알아서 필요한 파일들을 찾아서 읽겠지"
```

### 실제 동작

Agent는 **절대 스스로 판단하지 않습니다**. 모든 결정은 LLM이 합니다.

```typescript
// packages/core/src/core/turn.ts
async function* executeToolCalls(toolCalls: ToolCall[]) {
  for (const toolCall of toolCalls) {
    // LLM이 지시한 도구만 실행
    const tool = toolRegistry.getTool(toolCall.tool);

    // LLM이 제공한 파라미터만 사용
    const invocation = await tool.build(toolCall.args);

    // 실행만 함 (판단 없음)
    const result = await invocation.execute(signal);

    yield result;
  }
}
```

### 명확화

**Agent는 "실행 엔진"일 뿐입니다.**

LLM이 다음을 모두 결정합니다:
1. 어떤 도구를 사용할지
2. 어떤 순서로 실행할지
3. 어떤 파라미터를 전달할지
4. 결과를 어떻게 해석할지
5. 다음에 무엇을 할지

Agent는 그저 지시받은 대로 실행만 합니다.

### 예시

```
사용자: "TypeScript 파일들을 찾아서 분석해줘"

❌ Agent가 하지 않는 것:
   - TypeScript 파일 패턴 결정 (*.ts, *.tsx)
   - 어떤 디렉토리를 검색할지
   - 어떤 파일을 읽을지
   - 어떻게 분석할지

✅ LLM이 하는 것:
   1. "glob('**/*.{ts,tsx}') 로 파일 찾기"
   2. 결과 받기
   3. "가장 큰 파일 3개를 읽어야겠다"
   4. "read-file('src/main.ts')"
   5. "read-file('src/utils.ts')"
   6. 내용 분석 및 요약

✅ Agent가 하는 것:
   - glob 도구 실행
   - read-file 도구 실행 (3번)
   - 결과를 LLM에게 전달
```

## 오해 3: "LLM이 내 시스템에 접근한다"

### 흔한 생각

```
사용자 걱정:
"Gemini API가 내 컴퓨터에 직접 접근하는 건 아닐까?"
"내 파일이 구글 서버로 가는 건가?"
```

### 실제 동작

**LLM(Gemini API)은 원격 서버에서 실행됩니다.**

```
┌─────────────────────────────────────┐
│ 내 컴퓨터 (로컬)                     │
│                                      │
│  ┌────────────────────────────────┐ │
│  │ Agent (Gemini CLI)             │ │
│  │                                │ │
│  │ - 파일 시스템 접근             │ │
│  │ - 셸 명령 실행                 │ │
│  │ - 히스토리 저장                │ │
│  │ - 도구 실행                    │ │
│  └────────────────────────────────┘ │
│         ↕ (HTTPS)                    │
└─────────────────────────────────────┘
         ↕
┌─────────────────────────────────────┐
│ Google Cloud (원격)                  │
│                                      │
│  ┌────────────────────────────────┐ │
│  │ LLM (Gemini API)               │ │
│  │                                │ │
│  │ - 텍스트 생성                  │ │
│  │ - 도구 호출 결정               │ │
│  │ - 응답 생성                    │ │
│  │                                │ │
│  │ ❌ 파일 시스템 접근 불가       │ │
│  │ ❌ 시스템 명령 실행 불가       │ │
│  └────────────────────────────────┘ │
└─────────────────────────────────────┘
```

### 명확화

**LLM은 절대 로컬 시스템에 접근하지 않습니다.**

LLM과 주고받는 것:
1. **보내는 것**: 사용자 메시지, 대화 히스토리, 도구 결과 (텍스트)
2. **받는 것**: LLM의 응답 텍스트, 도구 호출 지시

보안 계층:
- **Agent**: 샌드박스, 권한 확인, 경로 검증
- **LLM**: 시스템 접근 권한 없음 (원격이므로)

```typescript
// packages/core/src/tools/read-file.ts
async execute(): Promise<ToolResult> {
  // Agent가 로컬에서 실행
  if (!isWithinSandbox(this.params.file_path)) {
    // Agent가 보안 검사
    throw new Error('Access denied: outside sandbox');
  }

  // Agent가 파일 읽기
  const content = await fs.promises.readFile(...);

  // 텍스트만 LLM에게 전송
  return { result: content };

  // LLM은 content 문자열만 받음
}
```

## 오해 4: "응답이 느린 건 LLM이 생각하고 있어서다"

### 흔한 생각

```
사용자: "코드를 분석해줘"
(10초 기다림)

사용자 생각:
"Gemini가 복잡한 코드를 열심히 분석하고 있구나"
```

### 실제 원인들

응답이 느린 이유는 **여러 가지**입니다:

#### 1. 네트워크 지연 (가장 흔함)

```typescript
// packages/core/src/core/client.ts
async sendMessage(message: string): Promise<Response> {
  // 1. API 요청 전송 (네트워크)
  const response = await fetch('https://generativelanguage.googleapis.com/...', {
    method: 'POST',
    body: JSON.stringify(request)
  });

  // 2. 응답 대기 (네트워크)
  return response;
}
```

**원인**: 인터넷 속도, API 서버 위치, 네트워크 혼잡도

#### 2. 도구 실행 시간

```
타임라인:
0초  - 사용자: "프로젝트 분석해줘"
1초  - LLM 응답: glob('**/*.ts') 호출
2초  - Agent: glob 실행 (1초 소요) ← 느림!
3초  - LLM에게 결과 전송
4초  - LLM 응답: 10개 파일 읽기
14초 - Agent: 10개 파일 읽기 (10초 소요) ← 느림!
15초 - LLM에게 결과 전송
17초 - LLM: 최종 분석 응답
```

**원인**: 파일 I/O, 셸 명령 실행, 느린 디스크

#### 3. LLM 생성 시간

```typescript
// 스트리밍 응답
for await (const chunk of stream) {
  // 청크가 하나씩 도착
  yield chunk.text();
}
```

**원인**:
- 프롬프트 길이 (대화 히스토리가 길면 느림)
- 응답 길이 (긴 답변은 생성에 시간 소요)
- API 서버 부하

#### 4. 여러 턴의 왕복

```
1턴: 사용자 → LLM → "glob 실행" → Agent
2턴: 결과 → LLM → "파일 읽기" → Agent
3턴: 결과 → LLM → "분석 완료" → 사용자

총 3번 왕복 = 3 × (네트워크 + LLM 생성)
```

### 디버깅: 어디가 느린지 확인하기

```typescript
// packages/core/src/core/turn.ts
async function* processTurn(message: string) {
  console.time('API Request');
  const response = await geminiClient.sendMessage(message);
  console.timeEnd('API Request');  // 네트워크 + LLM 시간

  if (response.toolCalls) {
    for (const toolCall of response.toolCalls) {
      console.time(`Tool: ${toolCall.tool}`);
      const result = await executeTool(toolCall);
      console.timeEnd(`Tool: ${toolCall.tool}`);  // 도구 실행 시간
    }
  }
}
```

### 명확화

**느린 이유는 상황에 따라 다릅니다:**

- 단순 질문이 느림 → 네트워크 또는 API 서버
- 파일 작업이 느림 → 디스크 I/O
- 여러 작업 후 느림 → 여러 턴 왕복
- 긴 응답이 느림 → LLM 생성 시간

## 오해 5: "에러는 LLM이 내는 것이다"

### 흔한 생각

```
터미널 출력:
❌ Error: ENOENT: no such file or directory

사용자 생각:
"Gemini가 에러를 냈구나"
```

### 실제 동작

**대부분의 에러는 Agent에서 발생합니다.**

#### Agent 에러 (90%)

```typescript
// packages/core/src/tools/read-file.ts
async execute(): Promise<ToolResult> {
  try {
    const content = await fs.promises.readFile(...);
    return { result: content };
  } catch (error) {
    // Agent에서 에러 감지
    if (error.code === 'ENOENT') {
      return {
        error: 'File not found',
        code: 'ENOENT'
      };
    }
    throw error;
  }
}
```

**Agent 에러 종류**:
- 파일 없음 (ENOENT)
- 권한 없음 (EACCES)
- 명령 실패 (exit code !== 0)
- 네트워크 오류
- 샌드박스 위반

#### LLM "에러" (10%)

실제로는 에러가 아니라 **LLM의 판단**입니다:

```
사용자: "/etc/passwd 파일 읽어줘"

LLM 응답:
"⚠️ /etc/passwd는 시스템 파일이므로 읽지 않겠습니다.
보안상 위험할 수 있습니다."
```

**이것은 에러가 아닙니다!** LLM이 안전을 위해 도구를 호출하지 않기로 **결정**한 것입니다.

#### 네트워크 에러

```typescript
// packages/core/src/core/client.ts
try {
  const response = await fetch(API_URL);
} catch (error) {
  // 네트워크 에러 (Agent 환경에서 발생)
  throw new Error('Failed to connect to Gemini API');
}
```

### 명확화

**에러 발생 위치 구분하기:**

| 에러 메시지 | 발생 위치 | 책임 |
|-----------|----------|------|
| "File not found" | Agent | 도구 실행 |
| "Permission denied" | Agent | 시스템/샌드박스 |
| "Command failed with exit code 1" | Agent | 셸 도구 |
| "Network error" | Agent | API 통신 |
| "API key not found" | Agent | 설정 |
| "시스템 파일은 읽을 수 없습니다" | LLM | 안전 판단 |
| "요청을 이해할 수 없습니다" | LLM | 의미 이해 실패 |

## 오해 6: "LLM이 코드를 실행한다"

### 흔한 생각

```
사용자: "npm install 실행해줘"

사용자 생각:
"Gemini가 내 컴퓨터에서 npm을 실행하는구나"
```

### 실제 동작

```
┌─────────────────────────────────────────────┐
│ 1. LLM 판단                                  │
│    "shell 도구로 'npm install' 실행"         │
├─────────────────────────────────────────────┤
│ 2. Agent가 도구 호출 감지                    │
│    ShellInvocation 생성                      │
├─────────────────────────────────────────────┤
│ 3. 정책 엔진 확인 (Agent)                    │
│    "write 작업이므로 사용자 승인 필요"       │
├─────────────────────────────────────────────┤
│ 4. 사용자에게 승인 요청 (Agent)              │
│    "Execute 'npm install'? (y/n)"            │
├─────────────────────────────────────────────┤
│ 5. 사용자 승인 후 실행 (Agent)               │
│    child_process.spawn('npm', ['install'])   │
├─────────────────────────────────────────────┤
│ 6. 출력 캡처 및 LLM에게 전송 (Agent)         │
│    "added 142 packages in 5s"                │
├─────────────────────────────────────────────┤
│ 7. LLM이 결과 해석 및 응답                   │
│    "142개 패키지가 설치되었습니다"           │
└─────────────────────────────────────────────┘
```

### 코드로 보기

```typescript
// packages/core/src/tools/shell.ts
export class ShellInvocation extends ToolInvocation {
  async execute(signal: AbortSignal): Promise<ToolResult> {
    // Agent가 실제로 명령 실행
    const process = spawn(command, args, {
      cwd: this.params.working_directory,
      shell: true
    });

    let stdout = '';
    let stderr = '';

    // Agent가 출력 캡처
    process.stdout.on('data', (data) => {
      stdout += data.toString();
    });

    process.stderr.on('data', (data) => {
      stderr += data.toString();
    });

    await new Promise((resolve) => process.on('close', resolve));

    // Agent가 결과를 LLM에게 전송
    return {
      tool: 'shell',
      result: { stdout, stderr, exit_code: process.exitCode }
    };
  }
}
```

### 명확화

**LLM은 절대 코드를 실행하지 않습니다.**

- **LLM**: "이 명령을 실행하면 되겠다" (판단)
- **Agent**: 실제로 child_process로 실행
- **LLM**: 출력 결과를 해석해서 사용자에게 설명

## 오해 7: "LLM이 내 코드베이스를 모두 알고 있다"

### 흔한 생각

```
사용자: "이 프로젝트의 인증 로직이 어떻게 돼?"

사용자 생각:
"Gemini가 내 프로젝트를 이미 다 알고 있겠지"
```

### 실제 동작

**LLM은 아무것도 모릅니다.** Agent가 필요한 것만 읽어서 보내줍니다.

```
1. LLM: "인증 관련 파일을 찾아야겠다"
   → grep('auth|login|signin')

2. Agent: grep 실행
   → src/auth.ts
   → src/middleware/auth.ts
   → 결과를 LLM에게 전송

3. LLM: "auth.ts를 읽어야겠다"
   → read-file('src/auth.ts')

4. Agent: 파일 읽기
   → 파일 내용을 LLM에게 전송

5. LLM: (이제 파일 내용을 봄)
   → "JWT 기반 인증을 사용합니다..."
```

### 코드로 보기

```typescript
// packages/core/src/core/geminiChat.ts
async sendTurn(userMessage: string) {
  // 현재 턴의 컨텍스트만 전송
  const request = {
    contents: [
      ...this.chatHistory,  // 이전 대화
      { role: 'user', parts: [{ text: userMessage }] }
    ],
    tools: this.toolDefinitions  // 도구 목록만
  };

  // LLM은 chatHistory에 있는 것만 봄
  // 코드베이스는 읽은 것만 포함됨
}
```

### 명확화

**LLM이 보는 것:**
- 사용자 메시지
- 이전 대화 히스토리
- Agent가 읽어서 보내준 파일 내용
- Agent가 실행한 명령의 출력

**LLM이 보지 못하는 것:**
- 읽지 않은 파일
- 실행하지 않은 명령의 결과
- 파일 시스템 구조 (glob/ls 결과 없이는)
- 프로젝트 메타데이터 (package.json 없이는)

### 최적화

```typescript
// packages/core/src/agents/codebase-investigator.ts
export class CodebaseInvestigatorAgent {
  async investigate(query: string): Promise<Investigation> {
    // 1. 먼저 구조 파악
    const structure = await this.analyzeStructure();

    // 2. 관련 파일만 읽기
    const relevantFiles = await this.findRelevantFiles(query);

    // 3. 요약해서 LLM에게 전송
    return {
      summary: '...',
      files: relevantFiles.slice(0, 10)  // 최대 10개만
    };
  }
}
```

**이유**: LLM 컨텍스트 제한을 고려해서 필요한 것만 선별적으로 전송

## 오해 8: "히스토리는 LLM이 저장한다"

### 흔한 생각

```
사용자: "어제 했던 대화 이어서 할게"

사용자 생각:
"Gemini API가 내 대화를 기억하고 있겠지"
```

### 실제 동작

**LLM은 완전히 무상태(stateless)입니다.**

```typescript
// packages/cli/src/services/history.ts
export class ConversationHistory {
  private historyDir = '~/.gemini-cli/history';

  async save(conversation: Conversation): Promise<void> {
    // Agent가 로컬 디스크에 저장
    const filePath = path.join(
      this.historyDir,
      `${conversation.id}.json`
    );

    await fs.promises.writeFile(
      filePath,
      JSON.stringify(conversation, null, 2)
    );
  }

  async load(conversationId: string): Promise<Conversation> {
    // Agent가 로컬 디스크에서 읽기
    const filePath = path.join(
      this.historyDir,
      `${conversationId}.json`
    );

    const content = await fs.promises.readFile(filePath, 'utf-8');
    return JSON.parse(content);
  }
}
```

### API 요청마다 히스토리 전송

```typescript
// packages/core/src/core/geminiChat.ts
async sendMessage(message: string) {
  const request = {
    contents: [
      // 매번 전체 히스토리를 전송!
      { role: 'user', parts: [{ text: '안녕?' }] },
      { role: 'model', parts: [{ text: '안녕하세요!' }] },
      { role: 'user', parts: [{ text: '이름이 뭐야?' }] },
      { role: 'model', parts: [{ text: '제 이름은...' }] },
      // ... (모든 이전 대화)
      { role: 'user', parts: [{ text: message }] }  // 새 메시지
    ]
  };

  // LLM은 매 요청마다 히스토리를 새로 받음
  return await this.api.generateContent(request);
}
```

### 명확화

**히스토리 관리:**

| 작업 | 담당 | 위치 |
|-----|------|------|
| 대화 저장 | Agent | `~/.gemini-cli/history/*.json` |
| 대화 로드 | Agent | 로컬 파일 시스템 |
| 히스토리 압축 | Agent | 오래된 턴 요약 |
| 히스토리 검색 | Agent | 로컬 JSON 파일 검색 |
| 컨텍스트 관리 | Agent | 토큰 제한 감안 |

**LLM은:**
- 각 요청마다 히스토리를 새로 받음
- 요청 사이에 아무것도 기억하지 않음
- 상태를 전혀 저장하지 않음

## 오해 9: "Agent는 단순한 명령 실행기일 뿐이다"

### 흔한 생각

```
"Agent는 그냥 LLM이 시키는 대로만 하는
단순한 스크립트 실행기겠지"
```

### 실제 역할

Agent는 **복잡하고 지능적인 시스템**입니다:

#### 1. 보안 정책 시행

```typescript
// packages/core/src/services/policy-engine.ts
export class PolicyEngine {
  async evaluate(invocation: ToolInvocation): Promise<Decision> {
    // 복잡한 보안 규칙 평가
    if (invocation.tool === 'shell') {
      // 위험한 명령 감지
      if (this.isDangerous(invocation.params.command)) {
        return { allow: false, reason: 'Dangerous command' };
      }
    }

    if (invocation.isWrite()) {
      // Write 작업은 항상 승인 필요
      return { allow: 'ask_user' };
    }

    return { allow: true };
  }
}
```

#### 2. 컨텍스트 최적화

```typescript
// packages/core/src/core/geminiChat.ts
private compressHistory(history: Message[]): Message[] {
  // 오래된 대화 요약해서 토큰 절약
  const oldMessages = history.slice(0, -10);
  const summary = this.summarize(oldMessages);

  return [
    { role: 'system', parts: [{ text: summary }] },
    ...history.slice(-10)  // 최근 10개만 유지
  ];
}
```

#### 3. 스트리밍 처리

```typescript
// packages/core/src/core/turn.ts
async function* streamResponse(response: Response) {
  for await (const chunk of response.stream()) {
    // 실시간으로 파싱 및 변환
    const parsed = parseChunk(chunk);

    // UI에 최적화된 형식으로 변환
    yield transformForUI(parsed);

    // 부분적으로 도구 호출 감지
    if (isToolCall(parsed)) {
      yield* executeToolAndContinue(parsed);
    }
  }
}
```

#### 4. 에러 복구

```typescript
// packages/core/src/core/client.ts
async sendWithRetry(request: Request): Promise<Response> {
  for (let attempt = 0; attempt < 5; attempt++) {
    try {
      return await this.send(request);
    } catch (error) {
      if (isTransientError(error) && attempt < 4) {
        // 지수 백오프로 재시도
        await sleep(Math.pow(2, attempt) * 1000);
        continue;
      }
      throw error;
    }
  }
}
```

#### 5. 도구 체인 관리

```typescript
// packages/core/src/core/turn.ts
async function* executeToolChain(calls: ToolCall[]) {
  const results: ToolResult[] = [];

  for (const call of calls) {
    // 이전 결과를 다음 도구에 전달
    const invocation = await buildWithContext(call, results);
    const result = await invocation.execute(signal);
    results.push(result);

    yield result;
  }

  // 모든 결과를 LLM에게 일괄 전송
  yield* sendResultsToLLM(results);
}
```

### 명확화

**Agent는 단순하지 않습니다:**

- 보안 정책 시행
- 컨텍스트 최적화
- 에러 처리 및 재시도
- 스트리밍 처리
- 파일 시스템 샌드박싱
- 도구 체인 오케스트레이션
- UI 렌더링 및 인터랙션
- 히스토리 관리
- MCP 통합

**LLM은 판단, Agent는 실행**이지만, **Agent의 "실행"은 매우 복잡합니다.**

## 정리: LLM vs Agent 체크리스트

사용자 관점에서 "이게 LLM인가, Agent인가?" 판단하기:

### LLM이 하는 것

✅ 의미 이해 및 판단
- [ ] 사용자 의도 파악
- [ ] 어떤 도구를 사용할지 결정
- [ ] 파라미터 값 결정
- [ ] 결과 해석 및 설명
- [ ] 다음 행동 계획

✅ 텍스트 생성
- [ ] 사용자 응답 작성
- [ ] 요약 및 분석
- [ ] 코드 설명
- [ ] 에러 메시지 해석

✅ 전략적 판단
- [ ] 작업 순서 결정
- [ ] 대안 제시
- [ ] 재시도 여부 판단 (전략적)

### Agent가 하는 것

✅ 시스템 상호작용
- [ ] 파일 시스템 접근
- [ ] 셸 명령 실행
- [ ] 네트워크 요청
- [ ] 프로세스 생성

✅ 데이터 관리
- [ ] 히스토리 저장/로드
- [ ] 캐시 관리
- [ ] 컨텍스트 압축

✅ 보안 및 정책
- [ ] 권한 확인
- [ ] 샌드박스 시행
- [ ] 사용자 승인 요청
- [ ] 위험한 작업 차단

✅ UI 및 표시
- [ ] 터미널 렌더링
- [ ] 색상 및 포맷팅
- [ ] 프로그레스 바
- [ ] 인터랙티브 프롬프트

✅ 인프라
- [ ] API 통신
- [ ] 에러 처리 (시스템 레벨)
- [ ] 재시도 (네트워크)
- [ ] 스트리밍 처리

### 양쪽이 협업하는 것

🤝 파라미터 검증
- LLM: 의미적 검증 ("이게 맞나?")
- Agent: 구문적 검증 ("이게 유효한가?")

🤝 에러 처리
- Agent: 에러 감지 및 분류
- LLM: 에러 해석 및 대응 전략

🤝 출력 포맷팅
- LLM: 내용 구성 ("무엇을 어떤 순서로")
- Agent: 표시 스타일 ("색상, 정렬, 간격")

🤝 재시도
- Agent: 시스템 레벨 자동 재시도
- LLM: 전략적 재시도 (다른 접근 방법)

## 다음 단계

이제 LLM과 Agent의 역할을 명확히 이해했으니, Gemini CLI를 더 효과적으로 사용할 수 있습니다:

1. **성능 문제 디버깅**: 어디가 느린지 정확히 파악
2. **에러 해결**: 에러의 출처를 알면 해결도 쉬움
3. **보안 이해**: 무엇이 로컬이고 무엇이 원격인지
4. **효율적 사용**: LLM의 능력과 한계를 이해하고 활용

---

**3부 완료!** 이제 "LLM과 Agent 사이"의 경계가 명확해졌습니다.
