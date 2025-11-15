# 5장. 사용 시나리오 (Use Case Scenarios)

## 개요

이 장에서는 다양한 실제 사용 시나리오를 엔드투엔드로 추적합니다. 각 시나리오는 사용자 입력부터 최종 응답까지의 전체 흐름을 API 요청/응답, 이벤트, 그리고 코드와 함께 상세히 분석합니다.

## 시나리오 1: 단순 질문-응답

### 사용자 요청

```
"TypeScript에서 제네릭이란 무엇인가?"
```

### 전체 흐름

#### 1. 요청 구성

```typescript
// API 요청
{
  "contents": [
    {
      "role": "user",
      "parts": [{ "text": "TypeScript에서 제네릭이란 무엇인가?" }]
    }
  ],
  "systemInstruction": {
    "parts": [{ "text": "You are an interactive CLI agent..." }]
  },
  "generationConfig": {
    "temperature": 1,
    "topP": 0.95,
    "topK": 64
  },
  "tools": [{ "functionDeclarations": [...] }]
}
```

#### 2. API 응답

```json
// 청크 1
{
  "candidates": [{
    "content": {
      "parts": [{ "text": "TypeScript의 제네릭은 " }]
    }
  }]
}

// 청크 2
{
  "candidates": [{
    "content": {
      "parts": [{ "text": "재사용 가능한 컴포넌트를 만들기 위한 도구입니다. " }]
    }
  }]
}

// 청크 3
{
  "candidates": [{
    "content": {
      "parts": [{ "text": "타입을 파라미터로 받아서 다양한 타입에서 작동하는 코드를 작성할 수 있게 해줍니다." }]
    },
    "finishReason": "STOP"
  }],
  "usageMetadata": {
    "promptTokenCount": 850,
    "candidatesTokenCount": 45,
    "totalTokenCount": 895
  }
}
```

#### 3. 생성된 이벤트

```typescript
{ type: 'content', value: "TypeScript의 제네릭은 " }
{ type: 'content', value: "재사용 가능한 컴포넌트를 만들기 위한 도구입니다. " }
{ type: 'content', value: "타입을 파라미터로 받아서 다양한 타입에서 작동하는 코드를 작성할 수 있게 해줍니다." }
{ type: 'finished', value: { reason: "STOP", usageMetadata: {...} } }
```

#### 4. UI 표시

```
TypeScript의 제네릭은 재사용 가능한 컴포넌트를 만들기 위한 도구입니다. 타입을 파라미터로 받아서 다양한 타입에서 작동하는 코드를 작성할 수 있게 해줍니다.

[895 tokens]
```

---

## 시나리오 2: 파일 읽기

### 사용자 요청

```
"README.md 파일을 읽어줘"
```

### 전체 흐름

#### 1. 요청 구성 및 API 호출

```typescript
{
  "contents": [
    {
      "role": "user",
      "parts": [{ "text": "README.md 파일을 읽어줘" }]
    }
  ],
  // ... systemInstruction, tools, etc.
}
```

#### 2. LLM Thought 응답

```json
{
  "candidates": [{
    "content": {
      "parts": [{
        "thought": true,
        "text": "**파일 읽기 작업**\n사용자가 README.md 파일을 읽기를 요청했습니다. read-file 도구를 사용하여 파일 내용을 가져와야 합니다."
      }]
    }
  }]
}
```

**이벤트**:
```typescript
{
  type: 'thought',
  value: {
    subject: "파일 읽기 작업",
    description: "사용자가 README.md 파일을 읽기를 요청했습니다. read-file 도구를 사용하여 파일 내용을 가져와야 합니다."
  }
}
```

#### 3. LLM Function Call

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

**이벤트**:
```typescript
{
  type: 'tool_call_request',
  value: {
    callId: "read-file-1701234567890-abc",
    name: "read-file",
    args: { file_path: "README.md" }
  }
}
```

#### 4. 도구 실행

**정책 결정**: ALLOW (read-file은 안전한 읽기 전용 도구)

**실행**:
```typescript
const result = await readFileInvocation.execute(signal);

// 결과
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

#### 5. FunctionResponse 생성 및 다음 턴

```typescript
// FunctionResponse
{
  "contents": [
    // 이전 메시지들...
    {
      "role": "user",
      "parts": [{
        "functionResponse": {
          "name": "read-file",
          "response": {
            "content": "1\t# My Project\n2\t\n3\tThis is a sample README file...",
            "resultDisplay": "Read 9 lines from README.md"
          }
        }
      }]
    }
  ]
}
```

#### 6. LLM 최종 응답

```json
{
  "candidates": [{
    "content": {
      "parts": [{
        "text": "README.md 파일의 내용은 다음과 같습니다:\n\n이 프로젝트는 'My Project'라는 이름의 샘플 프로젝트입니다. 설치는 `npm install` 명령으로 할 수 있습니다."
      }]
    },
    "finishReason": "STOP"
  }]
}
```

#### 7. 최종 UI 표시

```
💭 파일 읽기 작업
   사용자가 README.md 파일을 읽기를 요청했습니다.

🔧 read-file
   ✓ Read 9 lines from README.md

README.md 파일의 내용은 다음과 같습니다:

이 프로젝트는 'My Project'라는 이름의 샘플 프로젝트입니다. 설치는 `npm install` 명령으로 할 수 있습니다.
```

---

## 시나리오 3: 코드 분석 및 수정

### 사용자 요청

```
"src/utils.ts 파일에서 'processData' 함수를 찾아서 에러 처리를 추가해줘"
```

### 전체 흐름

#### Turn 1: 파일 읽기

**LLM 응답**:
```json
{
  "functionCall": {
    "name": "read-file",
    "args": { "file_path": "src/utils.ts" }
  }
}
```

**도구 실행 결과**:
```typescript
1\texport function processData(input: string) {
2\t  const result = input.toUpperCase();
3\t  return result;
4\t}
```

#### Turn 2: 함수 분석 및 수정 계획

**LLM 응답 (with Function Call)**:
```json
{
  "candidates": [{
    "content": {
      "parts": [
        {
          "text": "processData 함수를 찾았습니다. 현재 에러 처리가 없으므로 입력 검증과 try-catch를 추가하겠습니다."
        },
        {
          "functionCall": {
            "name": "edit",
            "args": {
              "file_path": "src/utils.ts",
              "old_string": "export function processData(input: string) {\n  const result = input.toUpperCase();\n  return result;\n}",
              "new_string": "export function processData(input: string) {\n  if (!input) {\n    throw new Error('Input cannot be empty');\n  }\n  \n  try {\n    const result = input.toUpperCase();\n    return result;\n  } catch (error) {\n    console.error('Error processing data:', error);\n    throw error;\n  }\n}"
            }
          }
        }
      ]
    }
  }]
}
```

**이벤트**:
```typescript
{ type: 'content', value: "processData 함수를 찾았습니다. 현재 에러 처리가 없으므로 입력 검증과 try-catch를 추가하겠습니다." }
{
  type: 'tool_call_request',
  value: {
    name: "edit",
    args: {
      file_path: "src/utils.ts",
      old_string: "export function processData(input: string) {...}",
      new_string: "export function processData(input: string) {...에러 처리 추가...}"
    }
  }
}
```

#### 사용자 확인

```
┌─────────────────────────────────────────┐
│ Edit File: src/utils.ts                │
│                                         │
│ Replace:                                │
│ export function processData(input: st…  │
│                                         │
│ With:                                   │
│ export function processData(input: st…  │
│ + 입력 검증 및 try-catch 추가            │
│                                         │
│ [Proceed] [Deny] [Proceed Always]      │
└─────────────────────────────────────────┘

→ Proceed
```

#### 도구 실행

```typescript
const result = await editInvocation.execute(signal);

// 결과
{
  responseParts: [{ text: "File edited successfully" }],
  resultDisplay: "Edited src/utils.ts"
}
```

#### Turn 3: 확인

**LLM 최종 응답**:
```
에러 처리를 추가했습니다. 이제 processData 함수는:
1. 입력이 비어있는지 검증합니다
2. try-catch 블록으로 예외를 처리합니다
3. 에러 발생 시 로그를 남기고 재발생시킵니다
```

---

## 시나리오 4: 다중 도구 사용

### 사용자 요청

```
"프로젝트의 TypeScript 파일들을 찾아서 각 파일의 라인 수를 알려줘"
```

### 전체 흐름

#### Turn 1: 파일 찾기

**LLM Function Call**:
```json
{
  "functionCall": {
    "name": "glob",
    "args": {
      "pattern": "**/*.ts"
    }
  }
}
```

**도구 실행 결과**:
```
src/index.ts
src/utils.ts
src/types.ts
tests/index.test.ts
```

#### Turn 2: 각 파일 라인 수 확인

**LLM Function Call** (다중 호출):
```json
{
  "candidates": [{
    "content": {
      "parts": [
        {
          "functionCall": {
            "name": "shell",
            "args": {
              "command": "wc -l src/index.ts",
              "description": "Count lines in src/index.ts"
            }
          }
        },
        {
          "functionCall": {
            "name": "shell",
            "args": {
              "command": "wc -l src/utils.ts",
              "description": "Count lines in src/utils.ts"
            }
          }
        },
        {
          "functionCall": {
            "name": "shell",
            "args": {
              "command": "wc -l src/types.ts",
              "description": "Count lines in src/types.ts"
            }
          }
        },
        {
          "functionCall": {
            "name": "shell",
            "args": {
              "command": "wc -l tests/index.test.ts",
              "description": "Count lines in tests/index.test.ts"
            }
          }
        }
      ]
    }
  }]
}
```

**병렬 도구 실행**:
```typescript
const results = await Promise.all([
  executeShell("wc -l src/index.ts"),
  executeShell("wc -l src/utils.ts"),
  executeShell("wc -l src/types.ts"),
  executeShell("wc -l tests/index.test.ts"),
]);

// 결과
[
  { content: "142 src/index.ts" },
  { content: "56 src/utils.ts" },
  { content: "23 src/types.ts" },
  { content: "89 tests/index.test.ts" }
]
```

#### Turn 3: 결과 요약

**LLM 최종 응답**:
```
프로젝트의 TypeScript 파일들과 라인 수는 다음과 같습니다:

- src/index.ts: 142 라인
- src/utils.ts: 56 라인
- src/types.ts: 23 라인
- tests/index.test.ts: 89 라인

총 4개 파일, 310 라인입니다.
```

---

## 시나리오 5: 에이전트 호출

### 사용자 요청

```
"인증 시스템이 어떻게 작동하는지 자세히 분석해줘"
```

### 전체 흐름

#### Turn 1: 에이전트 호출

**LLM Function Call**:
```json
{
  "functionCall": {
    "name": "codebase-investigator",
    "args": {
      "question": "인증 시스템이 어떻게 작동하는지 분석",
      "context": "JWT 토큰 처리, 미들웨어, 사용자 인증 흐름 포함"
    }
  }
}
```

#### 에이전트 실행

**에이전트 내부 Turn 1**: 파일 검색
```json
{
  "functionCall": {
    "name": "grep",
    "args": {
      "pattern": "auth|jwt|token",
      "glob": "**/*.ts"
    }
  }
}
```

**에이전트 내부 Turn 2**: 파일 읽기
```json
{
  "functionCall": {
    "name": "read-file",
    "args": { "file_path": "src/auth/jwt.ts" }
  }
}
```

**에이전트 내부 Turn 3**: 미들웨어 분석
```json
{
  "functionCall": {
    "name": "read-file",
    "args": { "file_path": "src/middleware/auth.ts" }
  }
}
```

**에이전트 내부 Turn 4**: 작업 완료
```json
{
  "functionCall": {
    "name": "complete_task",
    "args": {
      "findings": "인증 시스템은 JWT 기반으로 작동합니다...",
      "relevantFiles": [
        "src/auth/jwt.ts",
        "src/middleware/auth.ts",
        "src/routes/auth.ts"
      ],
      "explorationTrace": [
        { "action": "grep auth", "result": "Found 15 matches" },
        { "action": "read src/auth/jwt.ts", "result": "JWT 토큰 생성/검증 함수 발견" },
        { "action": "read src/middleware/auth.ts", "result": "인증 미들웨어 발견" }
      ]
    }
  }
}
```

#### Turn 2: 에이전트 결과 처리

**FunctionResponse**:
```typescript
{
  "functionResponse": {
    "name": "codebase-investigator",
    "response": {
      "findings": "인증 시스템은 JWT 기반으로 작동합니다. 주요 컴포넌트는:\n1. JWT 토큰 생성 (src/auth/jwt.ts)\n2. 인증 미들웨어 (src/middleware/auth.ts)\n3. 인증 라우트 (src/routes/auth.ts)...",
      "relevantFiles": [...],
      "explorationTrace": [...]
    }
  }
}
```

#### Turn 3: LLM 최종 응답

```
코드베이스를 분석한 결과, 인증 시스템은 다음과 같이 작동합니다:

## 구조
1. **JWT 토큰 관리** (src/auth/jwt.ts)
   - `generateToken()`: 사용자 정보로부터 JWT 생성
   - `verifyToken()`: JWT 검증 및 디코딩

2. **인증 미들웨어** (src/middleware/auth.ts)
   - 요청 헤더에서 토큰 추출
   - 토큰 검증
   - 사용자 정보를 req.user에 설정

3. **인증 라우트** (src/routes/auth.ts)
   - POST /login: 로그인 및 토큰 발급
   - POST /refresh: 토큰 갱신
   - POST /logout: 토큰 무효화

## 흐름
1. 사용자 로그인 요청
2. 자격 증명 검증
3. JWT 토큰 생성 및 반환
4. 클라이언트가 토큰을 저장
5. 이후 요청마다 Authorization 헤더에 토큰 포함
6. 미들웨어가 토큰 검증
7. 유효하면 요청 처리, 무효하면 401 응답
```

---

## 시나리오 6: 에러 처리 및 복구

### 사용자 요청

```
"nonexistent.txt 파일을 읽어줘"
```

### 전체 흐름

#### Turn 1: 파일 읽기 시도

**LLM Function Call**:
```json
{
  "functionCall": {
    "name": "read-file",
    "args": { "file_path": "nonexistent.txt" }
  }
}
```

**도구 실행 실패**:
```typescript
{
  error: Error("ENOENT: no such file or directory, open 'nonexistent.txt'"),
  errorType: ToolErrorType.ExecutionError,
  responseParts: [
    { text: "Error: ENOENT: no such file or directory, open 'nonexistent.txt'" }
  ]
}
```

**FunctionResponse**:
```typescript
{
  "functionResponse": {
    "name": "read-file",
    "response": {
      "error": "ENOENT: no such file or directory",
      "content": "Error: ENOENT: no such file or directory, open 'nonexistent.txt'"
    }
  }
}
```

#### Turn 2: LLM의 에러 처리

**LLM 응답**:
```
죄송합니다. 'nonexistent.txt' 파일을 찾을 수 없습니다. 현재 디렉토리의 파일 목록을 확인해 드릴까요?
```

**사용자**:
```
네, 확인해줘
```

#### Turn 3: 디렉토리 조회

**LLM Function Call**:
```json
{
  "functionCall": {
    "name": "ls",
    "args": { "directory": "." }
  }
}
```

**도구 실행 성공**:
```
README.md
package.json
src/
tests/
```

**LLM 최종 응답**:
```
현재 디렉토리에는 다음 파일과 폴더가 있습니다:

- README.md
- package.json
- src/ (디렉토리)
- tests/ (디렉토리)

찾으시는 'nonexistent.txt' 파일은 존재하지 않습니다. 혹시 다른 파일명을 의미하셨나요?
```

---

## 시나리오 요약

### 시나리오별 턴 수

| 시나리오 | 턴 수 | 도구 호출 수 | 특징 |
|---------|------|------------|------|
| 단순 Q&A | 1 | 0 | 도구 불필요 |
| 파일 읽기 | 2 | 1 | 기본 도구 사용 |
| 코드 수정 | 3 | 2 | 확인 필요 도구 |
| 다중 도구 | 3 | 5 | 병렬 실행 |
| 에이전트 호출 | 3 | 1 (에이전트 내부 4+) | 복잡한 분석 |
| 에러 복구 | 4 | 2 | 에러 처리 및 복구 |

### 공통 패턴

1. **사고 → 행동 → 결과**
   - LLM이 먼저 생각 (Thought)
   - 도구 호출로 행동
   - 결과를 바탕으로 최종 응답

2. **반복적 개선**
   - 초기 계획 수립
   - 단계별 실행
   - 결과 확인 및 조정

3. **에러 회복력**
   - 에러 감지
   - 대안 제시
   - 사용자 가이드

4. **사용자 제어**
   - 위험한 작업 확인
   - 진행 상황 표시
   - 취소 가능

## 마무리

Gemini CLI는 강력한 이벤트 기반 아키텍처와 도구 시스템을 통해 다양한 소프트웨어 엔지니어링 작업을 자율적으로 수행할 수 있습니다. 이 전자책에서 다룬 내용:

### 1부: 주요 구성 요소
- **아키텍처**: 계층 분리, 이벤트 스트리밍
- **GeminiClient**: 클라이언트 관리, 컨텍스트 압축
- **GeminiChat**: 세션 관리, 응답 검증
- **Turn**: 이벤트 생성, 턴 관리
- **Tool System**: 20+ 도구, MCP 통합, 정책 엔진
- **Agent System**: 특수 작업용 서브 에이전트
- **Prompt System**: 동적 프롬프트 생성

### 2부: LLM 인터페이스
- **요청 흐름**: 프롬프트 구성, 도구 정의, 히스토리 관리
- **응답 처리**: 스트림 파싱, 이벤트 변환, 검증
- **도구 실행**: 확인 프롬프트, 실행, FunctionResponse
- **시나리오**: 실제 사용 사례와 패턴

이 지식을 바탕으로 Gemini CLI를 확장하거나, 유사한 LLM 기반 CLI 도구를 개발하거나, 프로젝트에 기여할 수 있을 것입니다.
