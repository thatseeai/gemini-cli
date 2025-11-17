---
title: "1부: 주요 구성 요소"
weight: 01
---

# 1부: 주요 구성 요소

## 1장. 개요

### Gemini CLI란?

Gemini CLI는 Google Gemini API를 터미널 환경에서 활용할 수 있도록 하는 오픈소스 AI 에이전트입니다. 단순한 채팅 인터페이스를 넘어서, 파일 시스템 조작, 셸 명령 실행, 웹 검색 등 다양한 도구를 활용하여 실제 작업을 수행할 수 있는 강력한 에이전트 시스템을 제공합니다.

### 프로젝트 구조

Gemini CLI는 TypeScript 모노레포로 구성되어 있으며, npm workspaces를 사용하여 여러 패키지를 관리합니다.

```
packages/
├── cli/                    # Frontend: 사용자 인터페이스
│   └── src/
│       ├── ui/            # React 컴포넌트 (Ink 기반)
│       ├── commands/      # CLI 명령 핸들러
│       ├── services/      # CLI 전용 서비스
│       └── gemini.tsx     # 메인 진입점
│
├── core/                   # Backend: 핵심 로직
│   └── src/
│       ├── tools/         # 내장 도구 (20+ 도구)
│       ├── agents/        # 에이전트 구현
│       ├── services/      # 핵심 서비스
│       ├── prompts/       # 프롬프트 구성
│       ├── mcp/           # Model Context Protocol 통합
│       └── core/          # 핵심 클라이언트 로직
│
├── a2a-server/            # 실험적: Agent-to-Agent 서버
├── test-utils/            # 테스트 유틸리티
└── vscode-ide-companion/  # VS Code 확장
```

### 책임의 분리 (Separation of Concerns)

Gemini CLI의 아키텍처는 명확한 책임 분리 원칙을 따릅니다:

#### packages/cli - Frontend
- **사용자 경험 (UX)** 담당
- React/Ink를 사용한 터미널 UI 렌더링
- 사용자 입력 처리 및 출력 표시
- 테마 및 스타일링
- 대화 히스토리 관리 (UI 측면)

#### packages/core - Backend
- **비즈니스 로직** 담당
- Gemini API와의 통신
- 도구 등록, 관리, 실행
- 에이전트 프레임워크
- 상태 관리 (대화, 컨텍스트)
- 텔레메트리 및 정책 엔진

### 핵심 컴포넌트 미리보기

1부에서 다룰 주요 컴포넌트들을 간략히 소개합니다:

#### GeminiClient
- Gemini API와의 모든 상호작용을 관리하는 최상위 클래스
- 대화 세션 시작, 메시지 스트리밍, 도구 설정 등을 담당
- 파일 위치: `packages/core/src/core/client.ts`

#### GeminiChat
- 개별 채팅 세션을 관리
- 대화 히스토리 유지 및 검증
- API 요청/응답 처리
- 파일 위치: `packages/core/src/core/geminiChat.ts`

#### Turn
- 에이전트의 턴 기반 대화를 관리
- API 응답을 이벤트 스트림으로 변환
- 도구 호출 요청 처리
- 파일 위치: `packages/core/src/core/turn.ts`

#### Tool System
- 20개 이상의 내장 도구 제공
- MCP (Model Context Protocol) 통합
- 플러그인 형태로 확장 가능한 구조
- 파일 위치: `packages/core/src/tools/`

#### Agent System
- 특수 작업을 위한 전문 에이전트
- CodebaseInvestigatorAgent 등 내장 에이전트
- 사용자 정의 에이전트 생성 가능
- 파일 위치: `packages/core/src/agents/`

#### Prompt System
- 동적 시스템 프롬프트 생성
- 도구 설명, 워크스페이스 컨텍스트 포함
- 사용자 메모리 통합
- 파일 위치: `packages/core/src/core/prompts.ts`

### 기술 스택

- **언어**: TypeScript
- **런타임**: Node.js (>=20)
- **UI 프레임워크**: React (Ink - 터미널용 React)
- **API**: Google Generative AI SDK (@google/genai)
- **테스팅**: Vitest
- **빌드 도구**: npm workspaces

### 주요 설계 패턴

Gemini CLI에서 사용되는 주요 디자인 패턴들:

1. **빌더 패턴 (Builder Pattern)**
   - Tool (빌더) → ToolInvocation (실행 가능 객체)
   - 도구를 선언적으로 정의하고 실행 인스턴스를 생성

2. **전략 패턴 (Strategy Pattern)**
   - ContentGenerator 인터페이스와 다양한 구현체
   - 다양한 API 클라이언트 전략 지원

3. **옵저버 패턴 (Observer Pattern)**
   - 이벤트 제너레이터로 ServerGeminiStreamEvent 생성
   - 실시간 스트리밍 업데이트

4. **어댑터 패턴 (Adapter Pattern)**
   - MCP 서버 → Tool 변환
   - 외부 프로토콜을 내부 인터페이스로 적응

5. **비동기 제너레이터 (Async Generator)**
   - 스트리밍 응답을 제너레이터로 처리
   - 실시간 업데이트 및 백프레셔 제어

### 다음 장에서는

2장에서는 이러한 컴포넌트들이 어떻게 상호작용하는지 전체 아키텍처를 살펴보겠습니다.
