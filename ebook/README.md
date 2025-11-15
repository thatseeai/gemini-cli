# Gemini CLI Internal

Gemini CLI의 내부 구조와 동작 원리를 깊이 있게 다루는 전자책입니다.

## 목차

### 1부: 주요 구성 요소

Gemini CLI를 구성하는 핵심 컴포넌트들의 구조와 역할을 상세히 설명합니다.

1. [개요](part1/01_overview.md)
2. [전체 아키텍처](part1/02_architecture.md)
3. [GeminiClient - 클라이언트 관리자](part1/03_client.md)
4. [GeminiChat - 채팅 세션 관리](part1/04_chat.md)
5. [Turn - 턴 기반 대화 처리](part1/05_turn.md)
6. [도구 시스템 (Tool System)](part1/06_tools.md)
7. [에이전트 시스템 (Agent System)](part1/07_agents.md)
8. [프롬프트 시스템 (Prompt System)](part1/08_prompts.md)

### 2부: LLM 인터페이스

Gemini API와의 통신 메커니즘과 다양한 사용 시나리오를 실제 코드와 함께 분석합니다.

1. [개요](part2/01_overview.md)
2. [요청 흐름 (Request Flow)](part2/02_request_flow.md)
3. [응답 처리 (Response Processing)](part2/03_response_processing.md)
4. [도구 실행 (Tool Execution)](part2/04_tool_execution.md)
5. [사용 시나리오](part2/05_scenarios.md)

## 이 책에 대하여

이 전자책은 Gemini CLI 프로젝트의 실제 소스 코드를 기반으로 작성되었습니다. 각 챕터는 해당 기능을 구현하는 실제 코드 예제와 함께 제공되어, 이론과 실무를 연결하여 이해할 수 있도록 구성되었습니다.

## 대상 독자

- Gemini CLI 프로젝트에 기여하고자 하는 개발자
- LLM 기반 CLI 도구의 내부 동작을 이해하고자 하는 개발자
- TypeScript와 Node.js를 사용한 대규모 프로젝트 아키텍처에 관심 있는 개발자

## 프로젝트 정보

- **프로젝트 이름**: Gemini CLI
- **설명**: Google Gemini API를 활용한 터미널 기반 AI 에이전트
- **기술 스택**: TypeScript, Node.js, React (Ink), Google Gemini API
- **아키텍처**: 모노레포 (npm workspaces)
