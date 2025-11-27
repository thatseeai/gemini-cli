---
title: "2장. 기능별 역할 분담"
weight: 02
---

## 개요

이 장에서는 Gemini CLI의 주요 기능을 하나씩 분석하며, 각 기능에서 LLM과 Agent가 어떤 역할을 담당하는지 상세히 살펴봅니다.

## 1. 파일 읽기/쓰기

### 파일 읽기

#### LLM의 역할

**1. 의도 파악**
```
사용자: "README 파일을 확인해줘"

LLM 판단:
- "README 파일" = README.md 또는 README.txt
- "확인해줘" = 내용을 읽고 보여주기
→ read-file 도구 사용 결정
```

**2. 파일 경로 추론**
```
LLM 추론:
- 일반적으로 README는 프로젝트 루트에 위치
- 확장자는 .md가 가장 일반적
→ file_path = "README.md" 결정
```

**3. 내용 해석 (읽기 후)**
```
LLM이 파일 내용을 분석하여:
- 프로젝트 목적 파악
- 주요 기능 요약
- 설치 방법 추출
- 사용법 정리
```

#### Agent의 역할

**1. 실제 파일 읽기**
```typescript
// packages/core/src/tools/read-file.ts
async execute(signal: AbortSignal): Promise<ToolResult> {
  const { file_path } = this.params;

  // 실제 파일 시스템 접근
  const content = await fs.promises.readFile(file_path, {
    encoding: 'utf-8',
  });

  return {
    responseParts: [{ text: content }],
    resultDisplay: `Read ${file_path}`,
  };
}
```

**2. 에러 처리**
```typescript
try {
  const content = await fs.promises.readFile(file_path);
} catch (error) {
  if (error.code === 'ENOENT') {
    return {
      error: new Error('File not found'),
      responseParts: [{ text: 'File does not exist' }],
    };
  }
}
```

**3. 결과 포맷팅**
```typescript
// 라인 번호 추가
const numberedLines = lines
  .map((line, idx) => `${idx + 1}\t${line}`)
  .join('\n');
```

#### 역할 분담 요약

| 작업 | 담당 | 이유 |
|-----|------|------|
| 파일 경로 결정 | **LLM** | 사용자 의도와 컨텍스트 이해 필요 |
| 파일 읽기 | **Agent** | 파일 시스템 접근 권한 필요 |
| 내용 요약 | **LLM** | 자연어 이해 및 생성 필요 |
| 에러 처리 | **Agent** | 시스템 레벨 에러 처리 |

### 파일 쓰기

#### LLM의 역할

**1. 내용 생성**
```
사용자: "README.md에 설치 섹션을 추가해줘"

LLM이 생성:
## Installation

```bash
npm install
```

To install dependencies, run the command above.
```

**2. 기존 내용과의 조화**
```
LLM이 고려:
- 기존 README 스타일 (마크다운 형식, 헤더 레벨)
- 섹션 순서 (Installation은 보통 Usage 전에 위치)
- 톤앤매너 (기술적/친근함 등)
```

#### Agent의 역할

**1. 파일 쓰기 실행**
```typescript
// packages/core/src/tools/write-file.ts
async execute(signal: AbortSignal): Promise<ToolResult> {
  const { file_path, content } = this.params;

  await fs.promises.writeFile(file_path, content, {
    encoding: 'utf-8',
  });

  return {
    responseParts: [{ text: 'File written successfully' }],
    resultDisplay: `Wrote to ${file_path}`,
  };
}
```

**2. 백업 생성 (선택적)**
```typescript
// 기존 파일이 있으면 백업
if (await fs.promises.exists(file_path)) {
  await fs.promises.copyFile(
    file_path,
    `${file_path}.backup`,
  );
}
```

**3. 권한 확인**
```typescript
// 정책 엔진에서 확인
const decision = await policyEngine.evaluate('write-file', file_path);

if (decision === 'ASK_USER') {
  // 사용자 확인 프롬프트 표시
  const confirmed = await askUser({
    title: 'Write File',
    prompt: `Write to ${file_path}?`,
  });

  if (!confirmed) {
    throw new Error('User denied file write');
  }
}
```

## 2. 코드 분석

### 코드 이해

#### LLM의 역할

**1. 코드 구조 파악**
```typescript
// src/auth/jwt.ts 파일을 읽은 후

LLM 분석:
- generateToken 함수: JWT 토큰 생성
- verifyToken 함수: JWT 검증
- refreshToken 함수: 토큰 갱신
- 의존성: jsonwebtoken 라이브러리 사용
- 보안: SECRET_KEY 환경 변수 사용
```

**2. 로직 추론**
```typescript
LLM 추론:
"이 코드는 JWT 기반 인증을 구현합니다:
1. 로그인 시 generateToken으로 토큰 생성
2. 요청마다 verifyToken으로 검증
3. 만료 전 refreshToken으로 갱신"
```

**3. 잠재적 문제 발견**
```typescript
LLM이 발견:
"⚠️ SECRET_KEY가 하드코딩되면 보안 위험
⚠️ 토큰 만료 시간이 너무 긺 (7일)
⚠️ refresh token 재사용 방지 로직 없음"
```

#### Agent의 역할

**1. 파일 검색**
```typescript
// grep으로 인증 관련 파일 찾기
const files = await grepTool.execute({
  pattern: 'jwt|auth|token',
  glob: '**/*.ts',
});
```

**2. 여러 파일 읽기**
```typescript
// 발견된 파일들을 병렬로 읽기
const contents = await Promise.all(
  files.map(file => readFile(file)),
);
```

**3. 의존성 확인**
```typescript
// package.json 읽어서 설치된 라이브러리 확인
const packageJson = await readFile('package.json');
const dependencies = JSON.parse(packageJson).dependencies;
```

### 코드 수정

#### LLM의 역할

**1. 수정 계획 수립**
```
사용자: "JWT 만료 시간을 1시간으로 줄여줘"

LLM 계획:
1. jwt.ts 파일 읽기
2. expiresIn 설정 찾기
3. '7d' → '1h'로 변경
4. edit 도구로 수정
```

**2. 정확한 코드 작성**
```typescript
LLM이 생성한 old_string:
const token = jwt.sign(payload, SECRET_KEY, {
  expiresIn: '7d'
});

LLM이 생성한 new_string:
const token = jwt.sign(payload, SECRET_KEY, {
  expiresIn: '1h'
});
```

#### Agent의 역할

**1. 문자열 치환 실행**
```typescript
// packages/core/src/tools/edit.ts
async execute(signal: AbortSignal): Promise<ToolResult> {
  const { file_path, old_string, new_string } = this.params;

  let content = await fs.promises.readFile(file_path, 'utf-8');

  if (!content.includes(old_string)) {
    throw new Error('old_string not found in file');
  }

  content = content.replace(old_string, new_string);

  await fs.promises.writeFile(file_path, content);

  return {
    responseParts: [{ text: 'File edited successfully' }],
    resultDisplay: `Edited ${file_path}`,
  };
}
```

**2. 검증**
```typescript
// 수정 후 파일을 다시 읽어서 확인
const updatedContent = await fs.promises.readFile(file_path);
if (!updatedContent.includes(new_string)) {
  throw new Error('Edit verification failed');
}
```

## 3. 도구 선택

### LLM의 역할 (거의 모든 것)

**1. 상황 판단**
```
사용자: "프로젝트의 TypeScript 파일들을 찾아줘"

LLM 판단:
- "찾아줘" = 파일 검색
- "TypeScript 파일" = *.ts 또는 *.tsx
- 적합한 도구 = glob 또는 shell (find)
→ glob 선택 (더 간단하고 안전)
```

**2. 도구 비교 및 선택**
```
사용 가능한 도구:
- glob: 패턴 매칭으로 파일 찾기
- shell: find 명령 실행
- ls: 디렉토리 목록 조회

LLM 선택 이유:
✓ glob: 패턴 매칭에 최적화, 안전
✗ shell: 과도하게 복잡, 확인 필요
✗ ls: 재귀 검색 불가
→ glob 선택
```

**3. 파라미터 최적화**
```
LLM이 결정한 파라미터:
{
  "pattern": "**/*.ts",  // 재귀 검색
  "path": "."            // 현재 디렉토리부터
}

고려사항:
- node_modules 제외 (glob 도구가 자동 처리)
- .ts와 .tsx 모두 포함 필요 없음 (사용자가 .ts만 요청)
```

### Agent의 역할

**1. 도구 목록 제공**
```typescript
// packages/core/src/tools/tool-registry.ts
getFunctionDeclarations(): FunctionDeclaration[] {
  return [
    {
      name: 'read-file',
      description: 'Read the contents of a file',
      parameters: {...}
    },
    {
      name: 'write-file',
      description: 'Write content to a file',
      parameters: {...}
    },
    // ... 20개 이상의 도구
  ];
}
```

이 목록이 LLM에 전달되어 LLM이 선택할 수 있게 됨.

**2. 선택된 도구 실행**
```typescript
// LLM이 glob을 선택한 후
const tool = toolRegistry.getTool('glob');
const invocation = await tool.build(params);
const result = await invocation.execute(signal);
```

## 4. 컨텍스트 관리

### LLM의 역할

**거의 없음** - LLM은 상태를 저장하지 않습니다.

LLM은 매번 받는 히스토리만 볼 수 있음:
```json
{
  "contents": [
    { "role": "user", "parts": [{ "text": "첫 번째 질문" }] },
    { "role": "model", "parts": [{ "text": "첫 번째 답변" }] },
    { "role": "user", "parts": [{ "text": "두 번째 질문" }] }
  ]
}
```

### Agent의 역할 (거의 모든 것)

**1. 히스토리 저장**
```typescript
// packages/core/src/core/geminiChat.ts
private history: Content[] = [];

async sendMessageStream(...) {
  // 사용자 메시지 추가
  this.history.push(userContent);

  // API 호출 후 모델 응답 추가
  this.history.push(modelResponse);
}
```

**2. 히스토리 관리**
```typescript
// 유효한 히스토리만 추출
getHistory(curated: boolean = false): Content[] {
  const history = curated
    ? extractCuratedHistory(this.history)  // 유효한 것만
    : this.history;                         // 전부

  return structuredClone(history);  // Deep copy
}
```

**3. 히스토리 압축**
```typescript
// packages/core/src/services/chatCompressionService.ts
async compress(chat: GeminiChat, ...): Promise<CompressedHistory> {
  const history = chat.getHistory();

  // LLM에게 요약 요청
  const summary = await this.summarizeHistory(history);

  // 압축된 히스토리로 교체
  return {
    newHistory: [summary],
    compressionRatio: 0.2,  // 80% 감소
  };
}
```

**4. 토큰 카운팅**
```typescript
// 매 응답마다 토큰 수 기록
if (chunk.usageMetadata) {
  this.lastPromptTokenCount = chunk.usageMetadata.promptTokenCount;
}

// 컨텍스트 윈도우 초과 예방
if (estimatedTokens > tokenLimit * 0.95) {
  await this.tryCompressChat();
}
```

## 5. 에러 처리

### LLM의 역할

**1. 에러 해석**
```
Agent가 반환한 에러:
"Error: ENOENT: no such file or directory, open 'config.json'"

LLM 해석:
"config.json 파일이 존재하지 않습니다."
```

**2. 대안 제시**
```
LLM의 대응:
"config.json 파일을 찾을 수 없습니다.
대신 config.example.json을 확인해 드릴까요?
아니면 새로운 config.json을 생성해 드릴까요?"
```

**3. 재시도 전략**
```
LLM 판단:
- 파일명 오타 가능성 → 유사한 파일명 검색
- 경로 문제 가능성 → 다른 경로 시도
- 파일 미생성 → 생성 제안
```

### Agent의 역할

**1. 에러 감지 및 분류**
```typescript
try {
  const result = await toolInvocation.execute(signal);
} catch (error) {
  // 에러 타입 분류
  if (error.code === 'ENOENT') {
    return {
      errorType: ToolErrorType.NotFound,
      error: error,
      responseParts: [{ text: 'File not found' }],
    };
  } else if (error.code === 'EACCES') {
    return {
      errorType: ToolErrorType.PermissionDenied,
      error: error,
      responseParts: [{ text: 'Permission denied' }],
    };
  }
}
```

**2. 자동 재시도**
```typescript
// packages/core/src/core/geminiChat.ts
for (let attempt = 0; attempt < maxAttempts; attempt++) {
  try {
    const stream = await this.makeApiCallAndProcessStream(...);
    // 성공
    break;
  } catch (error) {
    if (error instanceof InvalidStreamError && attempt < maxAttempts - 1) {
      // 재시도
      await sleep(500 * (attempt + 1));
      continue;
    }
    throw error;
  }
}
```

**3. 폴백 처리**
```typescript
// 429 에러 시 더 작은 모델로 전환
if (is429Error(error)) {
  await handleFallback(config, currentModel);
  // gemini-2.5-pro → gemini-2.5-flash로 전환
  return await apiCall();  // 재시도
}
```

## 6. 보안 및 정책

### LLM의 역할

**없음** - LLM은 보안 결정을 내리지 않습니다.

LLM은 단순히 필요한 도구를 요청할 뿐:
```json
{
  "functionCall": {
    "name": "shell",
    "args": {
      "command": "rm -rf important-folder"
    }
  }
}
```

### Agent의 역할 (전부)

**1. 정책 평가**
```typescript
// packages/core/src/tools/tools.ts
async shouldConfirmExecute(abortSignal): Promise<boolean> {
  const decision = await this.policyEngine.evaluate(
    this._toolName,
    this.params,
  );

  if (decision === 'ALLOW') {
    return false;  // 확인 불필요
  } else if (decision === 'DENY') {
    throw new Error('Denied by policy');
  } else if (decision === 'ASK_USER') {
    return this.getConfirmationDetails();
  }
}
```

**2. 사용자 확인**
```typescript
const confirmed = await askUser({
  type: 'warning',  // 위험한 작업
  title: 'Shell Command',
  prompt: `Execute: rm -rf important-folder?`,
  options: ['Proceed', 'Deny', 'Proceed Always'],
});

if (confirmed === 'Deny') {
  throw new Error('User denied operation');
}
```

**3. 샌드박스 적용**
```bash
# macOS Seatbelt
sandbox-exec -p profile.sb gemini

# Docker
docker run --rm -it gemini-cli-sandbox

# Podman
podman run --security-opt label=type:container_t gemini-cli
```

## 7. 프롬프트 생성

### LLM의 역할

**없음** - 시스템 프롬프트는 LLM에게 주어지는 것입니다.

### Agent의 역할 (전부)

**1. 시스템 프롬프트 구성**
```typescript
// packages/core/src/core/prompts.ts
export function getCoreSystemPrompt(config: Config): string {
  let prompt = `You are an interactive CLI agent...`;

  // 도구 목록 추가
  prompt += `\nAvailable tools:\n`;
  for (const tool of tools) {
    prompt += `- ${tool.name}: ${tool.description}\n`;
  }

  // Git 컨텍스트 추가
  if (isGitRepo()) {
    prompt += `\n# Git Repository\n...`;
  }

  // 사용자 메모리 추가
  if (userMemory) {
    prompt += `\n---\n${userMemory}`;
  }

  return prompt;
}
```

**2. 동적 컨텍스트 추가**
```typescript
// IDE 컨텍스트
if (config.getIdeMode()) {
  const ideContext = {
    activeFile: editor.activeFile,
    cursor: editor.cursorPosition,
    selectedText: editor.selection,
  };

  chat.addHistory({
    role: 'user',
    parts: [{ text: JSON.stringify(ideContext) }],
  });
}
```

## 역할 분담 요약표

| 기능 | LLM | Agent |
|-----|-----|-------|
| **파일 읽기** | 경로 결정, 내용 이해 | 실제 읽기, 에러 처리 |
| **파일 쓰기** | 내용 생성, 스타일 결정 | 실제 쓰기, 백업, 권한 확인 |
| **코드 분석** | 로직 이해, 문제 발견 | 파일 검색, 내용 수집 |
| **코드 수정** | 수정 계획, 코드 작성 | 문자열 치환, 검증 |
| **도구 선택** | 도구 비교 및 선택 | 도구 목록 제공, 실행 |
| **컨텍스트 관리** | (없음) | 히스토리 저장, 압축, 관리 |
| **에러 처리** | 해석, 대안 제시 | 감지, 분류, 재시도, 폴백 |
| **보안/정책** | (없음) | 정책 평가, 사용자 확인, 샌드박스 |
| **프롬프트 생성** | (없음) | 시스템 프롬프트 구성, 컨텍스트 추가 |

## 다음 장에서는

3장에서는 "판단"과 "실행"의 경계를 더 명확히 하고, 애매한 경우들을 어떻게 처리하는지 살펴보겠습니다.
