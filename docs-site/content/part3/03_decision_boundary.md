---
title: "3장. 판단과 실행의 경계"
weight: 03
---

# 3장. 판단과 실행의 경계

## 개요

LLM과 Agent의 역할 구분에서 가장 중요한 개념은 **"판단(Decision)"과 "실행(Execution)"의 분리**입니다. 이 장에서는 이 경계가 어디에 있는지, 그리고 애매한 경우들을 어떻게 처리하는지 살펴봅니다.

## 핵심 원칙: 판단-실행 분리

### 판단 (LLM의 영역)

**정의**: "무엇을", "언제", "어떻게" 할지 결정하는 것

**예시**:
- 어떤 파일을 읽을지
- 어떤 도구를 사용할지
- 어떤 순서로 작업할지
- 어떤 내용을 작성할지
- 에러 발생 시 어떻게 대응할지

### 실행 (Agent의 영역)

**정의**: 결정된 것을 실제로 수행하는 것

**예시**:
- 파일 시스템에 접근하기
- 셸 명령 실행하기
- 네트워크 요청 보내기
- 데이터 저장하기
- 사용자 확인 받기

## 경계가 명확한 경우

### 사례 1: 파일 경로 결정

**사용자**: "설정 파일을 읽어줘"

```
┌─────────────────┬───────────────────────┐
│ 판단 (LLM)      │ 실행 (Agent)          │
├─────────────────┼───────────────────────┤
│ "설정 파일" =   │                       │
│ config.json     │                       │
│ 또는            │                       │
│ config.yaml     │                       │
│                 │                       │
│ → 일반적으로는  │                       │
│   config.json   │                       │
│   선택          │                       │
│                 │                       │
│ read-file       │ fs.readFile(          │
│ 도구 호출 결정  │   "config.json"       │
│                 │ ) 실행                │
└─────────────────┴───────────────────────┘
```

**명확한 이유**:
- 경로 추론은 의미 이해가 필요 → LLM
- 파일 읽기는 시스템 접근이 필요 → Agent

### 사례 2: 에러 메시지 해석

**상황**: 파일을 찾을 수 없음

```
┌─────────────────┬───────────────────────┐
│ 판단 (LLM)      │ 실행 (Agent)          │
├─────────────────┼───────────────────────┤
│                 │ try {                 │
│                 │   fs.readFile(...)    │
│                 │ } catch (e) {         │
│                 │   return {            │
│                 │     error: e,         │
│                 │     code: 'ENOENT'    │
│                 │   }                   │
│                 │ }                     │
│                 │                       │
│ "ENOENT" 해석:  │                       │
│ "파일이         │                       │
│  존재하지       │                       │
│  않습니다"      │                       │
│                 │                       │
│ 대안 제시:      │                       │
│ "유사한 파일명  │                       │
│  검색 또는      │                       │
│  새 파일 생성"  │                       │
└─────────────────┴───────────────────────┘
```

**명확한 이유**:
- 에러 감지 및 분류는 시스템 작업 → Agent
- 에러 의미 해석 및 대안 제시는 언어 이해 → LLM

## 경계가 애매한 경우

### 사례 1: 파라미터 검증

**질문**: 파라미터가 유효한지 확인하는 것은 누구 책임?

**답**: **양쪽 모두** 하지만 다른 수준에서

#### LLM의 검증 (의미적 검증)

```
사용자: "파일 경로가 '/etc/passwd'인데 읽어줘"

LLM 판단:
❌ /etc/passwd는 시스템 파일
❌ 일반 사용자가 접근하면 안 됨
❌ 의도와 맞지 않을 가능성 높음

LLM 응답:
"⚠️ /etc/passwd는 시스템 파일입니다.
정말로 이 파일을 읽으시겠습니까?"
```

#### Agent의 검증 (구문적 검증)

```typescript
// packages/core/src/tools/read-file.ts
async buildCore(params: ReadFileParams): Promise<ReadFileInvocation> {
  // 파라미터 스키마 검증
  if (!params.file_path) {
    throw new Error('file_path is required');
  }

  if (typeof params.file_path !== 'string') {
    throw new Error('file_path must be a string');
  }

  // 경로 정규화
  const normalizedPath = path.resolve(params.file_path);

  // 샌드박스 외부 접근 차단 (Agent 책임)
  if (!isWithinSandbox(normalizedPath)) {
    throw new Error('Access denied: outside sandbox');
  }

  return new ReadFileInvocation(params);
}
```

**구분 기준**:
- **의미적 검증** (LLM): "이게 사용자가 원하는 건가?"
- **구문적 검증** (Agent): "이게 시스템이 허용하는 건가?"

### 사례 2: 재시도 결정

**질문**: 실패한 작업을 재시도할지 결정하는 것은 누구 책임?

**답**: **상황에 따라 다름**

#### Agent의 자동 재시도 (시스템 레벨)

```typescript
// 일시적 네트워크 오류 → Agent가 자동 재시도
async function retryWithBackoff(apiCall, options) {
  for (let attempt = 0; attempt < 5; attempt++) {
    try {
      return await apiCall();
    } catch (error) {
      if (isTransientError(error) && attempt < 4) {
        await sleep(1000 * Math.pow(2, attempt));
        continue;  // 자동 재시도
      }
      throw error;
    }
  }
}
```

**Agent 재시도 조건**:
- 일시적 네트워크 오류
- 429 Rate Limit
- 타임아웃
- 무효한 응답 스트림

#### LLM의 전략적 재시도 (논리적 판단)

```
첫 시도: read-file("config.json")
결과: Error - File not found

LLM 판단:
"파일이 없네요. 다른 시도를 해봐야겠어요."

LLM 재시도:
1. glob("**/ config.json") → 다른 위치 검색
2. read-file("config.example.json") → 예제 파일 확인
3. 또는 새 파일 생성 제안
```

**LLM 재시도 조건**:
- 다른 접근 방법 시도
- 대안 찾기
- 전략 변경

**구분 기준**:
- **시스템 레벨 재시도** (Agent): 같은 작업을 반복
- **전략적 재시도** (LLM): 다른 접근 방법 시도

### 사례 3: 출력 포맷팅

**질문**: 출력을 어떤 형식으로 표시할지 결정하는 것은 누구 책임?

**답**: **협업**

#### LLM의 포맷팅 (내용 구성)

```
LLM이 생성한 응답:
"파일 목록:

1. src/index.ts (142 lines)
2. src/utils.ts (56 lines)
3. src/types.ts (23 lines)

총 3개 파일, 221 줄입니다."
```

#### Agent의 포맷팅 (표시 스타일)

```typescript
// packages/cli/src/ui/ContentDisplay.tsx
<Box flexDirection="column">
  <Text color="cyan">파일 목록:</Text>
  <Text></Text>
  <Text>1. <Text color="green">src/index.ts</Text> (142 lines)</Text>
  <Text>2. <Text color="green">src/utils.ts</Text> (56 lines)</Text>
  <Text>3. <Text color="green">src/types.ts</Text> (23 lines)</Text>
  <Text></Text>
  <Text dimColor>총 3개 파일, 221 줄입니다.</Text>
</Box>
```

**구분 기준**:
- **내용 구성** (LLM): 무엇을 어떤 순서로
- **표시 스타일** (Agent): 색상, 정렬, 간격

## 책임의 레이어

```
┌─────────────────────────────────────┐
│ 사용자 인터페이스                    │ ← Agent
│ (색상, 레이아웃, 인터랙션)           │
├─────────────────────────────────────┤
│ 내용 생성                            │ ← LLM
│ (텍스트, 설명, 요약)                 │
├─────────────────────────────────────┤
│ 전략 및 계획                         │ ← LLM
│ (무엇을, 언제, 어떻게)               │
├─────────────────────────────────────┤
│ 도구 선택                            │ ← LLM
│ (어떤 도구, 어떤 파라미터)           │
├─────────────────────────────────────┤
│ 파라미터 검증                        │ ← 양쪽
│ (의미: LLM, 구문: Agent)             │
├─────────────────────────────────────┤
│ 도구 실행                            │ ← Agent
│ (파일 시스템, 셸, 네트워크)          │
├─────────────────────────────────────┤
│ 에러 처리                            │ ← 양쪽
│ (감지: Agent, 해석: LLM)             │
├─────────────────────────────────────┤
│ 보안 정책                            │ ← Agent
│ (권한 확인, 샌드박스)                │
└─────────────────────────────────────┘
```

## 실전 예제: 복잡한 작업의 책임 분담

### 예제: "프로젝트를 분석해서 보고서를 작성해줘"

#### 1단계: 작업 분해 (LLM)

```
LLM 계획:
1. 프로젝트 구조 파악
   - package.json 읽기
   - 디렉토리 구조 조회
2. 주요 파일 분석
   - 소스 코드 파일 찾기
   - README 읽기
3. 통계 수집
   - 파일 수, 라인 수
   - 의존성 분석
4. 보고서 작성
   - 구조화된 형식으로 정리
   - 마크다운 형식
```

#### 2단계: 도구 호출 (LLM)

```json
[
  { "tool": "read-file", "args": { "file_path": "package.json" } },
  { "tool": "ls", "args": { "directory": ".", "recursive": true } },
  { "tool": "glob", "args": { "pattern": "**/*.{ts,js}" } },
  { "tool": "shell", "args": { "command": "find src -name '*.ts' | xargs wc -l" } }
]
```

#### 3단계: 도구 실행 (Agent)

```typescript
// Agent가 각 도구를 순차적으로 실행
for (const toolCall of toolCalls) {
  const tool = toolRegistry.getTool(toolCall.tool);
  const result = await tool.build(toolCall.args).execute(signal);
  results.push(result);
}
```

#### 4단계: 결과 분석 (LLM)

```
LLM 분석:
- package.json: React, TypeScript, Express 사용
- 디렉토리: src/, tests/, docs/ 구조
- 파일 수: TypeScript 45개, 총 3,245 줄
- 의존성: 12개 (production), 8개 (dev)
```

#### 5단계: 보고서 작성 (LLM)

```markdown
# 프로젝트 분석 보고서

## 개요
이 프로젝트는 TypeScript 기반의 웹 애플리케이션으로...

## 기술 스택
- Frontend: React 18
- Backend: Express.js
- 언어: TypeScript 5.0

## 구조
...

## 통계
- 총 파일: 45개
- 총 코드: 3,245 줄
- 의존성: 20개

## 권장 사항
...
```

#### 6단계: 표시 (Agent)

```typescript
// UI에 마크다운 렌더링
<MarkdownDisplay content={llmResponse} />
```

### 책임 분담 요약

| 단계 | 작업 | 담당 |
|-----|------|------|
| 1 | 작업 분해 | **LLM** |
| 2 | 도구 선택 및 파라미터 결정 | **LLM** |
| 3 | 도구 실행 | **Agent** |
| 4 | 결과 분석 및 해석 | **LLM** |
| 5 | 보고서 내용 작성 | **LLM** |
| 6 | UI 렌더링 | **Agent** |

## 그레이 존 (Gray Zone)

일부 작업은 LLM과 Agent 중 어디서 처리해도 되는 경우가 있습니다.

### 예: 파일 경로 정규화

**옵션 1: LLM이 처리**
```
사용자: "~/ config.json 읽어줘"

LLM:
"~/" = 사용자 홈 디렉토리
→ "/home/user/config.json"
```

**옵션 2: Agent가 처리**
```typescript
const normalizedPath = path.resolve(
  params.file_path.replace('~', os.homedir())
);
```

**Gemini CLI의 선택**: **Agent가 처리**

**이유**:
- 시스템 정보 (홈 디렉토리) 접근 필요
- 플랫폼 의존적 (Windows는 다름)
- LLM 토큰 절약
- 일관성 보장

### 일반 원칙: 그레이 존 처리

```
낮은 수준의 기술적 세부사항 → Agent
높은 수준의 의미적 판단 → LLM

반복적인 단순 작업 → Agent
창의적이고 복잡한 판단 → LLM

시스템 의존적 → Agent
플랫폼 독립적 → LLM

속도가 중요 → Agent
품질이 중요 → LLM
```

## 다음 장에서는

4장에서는 사용자들이 흔히 오해하는 부분들과 그 명확화를 다루겠습니다.
