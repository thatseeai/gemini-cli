---
title: "2부: LLM 인터페이스"
weight: 01
---

## 1장. 개요

2부에서는 Gemini CLI가 Gemini API와 어떻게 통신하는지, 요청을 어떻게 구성하고 응답을 어떻게 처리하는지, 그리고 다양한 사용 시나리오에서 어떻게 동작하는지 실제 코드와 함께 살펴봅니다.

## LLM 인터페이스란?

LLM 인터페이스는 사용자의 요청을 LLM이 이해할 수 있는 형식으로 변환하고, LLM의 응답을 다시 사용자에게 유용한 형태로 변환하는 계층입니다. Gemini CLI의 경우, 다음과 같은 주요 기능을 포함합니다:

1. **요청 구성**: 사용자 메시지 + 시스템 프롬프트 + 도구 정의 + 히스토리
2. **스트리밍 처리**: 실시간 응답 스트리밍
3. **도구 실행**: LLM이 요청한 도구 호출 처리
4. **에러 처리**: 재시도, 폴백, 검증
5. **컨텍스트 관리**: 히스토리 압축, 토큰 제한

## 2부 구성

### 2장: 요청 흐름 (Request Flow)

사용자 메시지가 어떻게 Gemini API 요청으로 변환되는지 단계별로 살펴봅니다.

**주요 내용**:
- 요청 구성 요소
- 프롬프트 조합
- 도구 정의 포함
- 실제 API 요청 예제

### 3장: 응답 처리 (Response Processing)

Gemini API의 스트리밍 응답을 어떻게 파싱하고 이벤트로 변환하는지 알아봅니다.

**주요 내용**:
- 스트림 청크 처리
- 이벤트 타입별 처리
- Thought, Content, Function Call 파싱
- 응답 검증 및 재시도

### 4장: 도구 실행 (Tool Execution)

LLM이 요청한 도구 호출을 어떻게 실행하고 결과를 다시 LLM에 전달하는지 살펴봅니다.

**주요 내용**:
- 도구 호출 요청 처리
- 파라미터 검증
- 도구 실행
- 결과를 FunctionResponse로 변환
- 다음 턴 시작

### 5장: 사용 시나리오 (Use Case Scenarios)

다양한 실제 사용 시나리오에서의 전체 흐름을 추적합니다.

**주요 내용**:
- 단순 질문-응답
- 파일 읽기/쓰기
- 코드 분석 및 수정
- 다중 도구 사용
- 에이전트 호출
- 에러 처리 및 복구

## 실습 중심 접근

2부의 각 장에는 실제 코드 예제와 함께 API 요청/응답의 실제 페이로드를 보여줍니다. 이를 통해 이론과 실무를 연결하여 이해할 수 있습니다.

## API 통신 스택

Gemini CLI의 API 통신 스택을 간략히 살펴보면:

```
사용자 입력
    ↓
[Presentation Layer]
useGeminiStream
    ↓
[Business Logic Layer]
GeminiClient.sendMessageStream()
    ├─ 프롬프트 구성
    ├─ 컨텍스트 압축
    ├─ 모델 라우팅
    └─ Turn 실행
        ↓
GeminiChat.sendMessageStream()
    ├─ 히스토리 관리
    ├─ 재시도 로직
    └─ 스트림 검증
        ↓
[Data Access Layer]
ContentGenerator.generateContentStream()
    ↓
[External API]
Google Generative AI SDK
    ↓
Gemini API
```

## 통신 프로토콜

Gemini CLI는 Google Generative AI SDK (`@google/genai`)를 사용하여 Gemini API와 통신합니다.

### 주요 API 메서드

```typescript
// 스트리밍 콘텐츠 생성
generateContentStream(request: {
  model: string;
  contents: Content[];
  config: GenerateContentConfig;
}): AsyncGenerator<GenerateContentResponse>

// 단일 콘텐츠 생성 (비스트리밍)
generateContent(request: {
  model: string;
  contents: Content[];
  config: GenerateContentConfig;
}): Promise<GenerateContentResponse>
```

### Content 구조

```typescript
interface Content {
  role: 'user' | 'model';
  parts: Part[];
}

type Part =
  | { text: string }
  | { functionCall: FunctionCall }
  | { functionResponse: FunctionResponse }
  | { inlineData: InlineData }
  | { fileData: FileData }
  | { thought: boolean; text: string };
```

### GenerateContentConfig

```typescript
interface GenerateContentConfig {
  temperature?: number;
  topP?: number;
  topK?: number;
  maxOutputTokens?: number;
  systemInstruction?: string;
  tools?: Tool[];
  thinkingConfig?: {
    includeThoughts: boolean;
    thinkingBudget?: number;
  };
  abortSignal?: AbortSignal;
}
```

## 이벤트 기반 아키텍처

Gemini CLI는 이벤트 기반 아키텍처를 사용하여 비동기 스트리밍을 처리합니다.

### 이벤트 흐름

```
API 스트림 청크
    ↓
GeminiChat (검증 및 병합)
    ↓
Turn (파싱 및 이벤트 생성)
    ↓
ServerGeminiStreamEvent
    ↓
GeminiClient (루프 감지, 텔레메트리)
    ↓
useGeminiStream (UI 업데이트)
    ↓
React 컴포넌트 (렌더링)
```

### 주요 이벤트 타입

1. **Content**: 텍스트 콘텐츠
2. **Thought**: 모델의 사고 과정
3. **ToolCallRequest**: 도구 호출 요청
4. **ToolCallResponse**: 도구 실행 결과
5. **Finished**: 턴 완료
6. **Error**: 에러 발생
7. **Retry**: 재시도

## 비동기 제너레이터의 활용

Gemini CLI는 비동기 제너레이터를 광범위하게 사용하여 스트리밍을 처리합니다.

### 비동기 제너레이터 예제

```typescript
async function* sendMessageStream(
  request: PartListUnion,
  signal: AbortSignal,
): AsyncGenerator<ServerGeminiStreamEvent, Turn> {
  // 이벤트 생성 및 yield
  yield { type: GeminiEventType.ModelInfo, value: modelName };

  // 중첩된 제너레이터 위임
  yield* turn.run(model, request, signal);

  // 최종 값 반환
  return turn;
}

// 사용
const stream = client.sendMessageStream(request, signal);
for await (const event of stream) {
  console.log(event);
}
const finalTurn = await stream.return().value;
```

**장점**:
- 실시간 스트리밍
- 백프레셔 제어
- 자연스러운 에러 처리
- 취소 지원 (AbortSignal)

## 다음 장에서는

2장에서는 요청 흐름을 자세히 살펴보며, 사용자 메시지가 어떻게 API 요청으로 변환되는지 실제 예제와 함께 분석하겠습니다.
