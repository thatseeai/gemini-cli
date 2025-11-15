# CLAUDE_KO.md

이 파일은 Claude Code(claude.ai/code)가 이 저장소에서 작업할 때 참고할 수 있는 한글 가이드입니다.

## 프로젝트 개요

Gemini CLI는 터미널에서 직접 Gemini의 강력한 기능을 사용할 수 있게 해주는 오픈소스 AI 에이전트입니다. npm 워크스페이스를 사용하는 TypeScript 모노레포로, 프론트엔드는 React 기반의 터미널 UI(Ink)를, 백엔드는 Google Gemini API와 통신하는 Node.js로 구성되어 있습니다.

## 필수 빌드 및 테스트 명령어

**커밋 전 필수 실행:**
```bash
npm run preflight
```
빌드, 테스트, 타입 체크, 린팅을 포함한 전체 검증 과정을 실행합니다. 모든 변경사항은 제출 전에 이 명령을 통과해야 합니다.

**개발 워크플로우:**
```bash
npm install              # 의존성 설치
npm run build            # 모든 패키지 빌드
npm start                # 소스에서 CLI 실행
npm run build:all        # 샌드박스 컨테이너 포함 빌드
```

**테스트:**
```bash
npm run test                                 # 유닛 테스트만 실행
npm run test:e2e                            # 통합 테스트 (샌드박스 없음)
npm run test:integration:sandbox:docker     # Docker 샌드박스 통합 테스트
npm run test:integration:sandbox:podman     # Podman 샌드박스 통합 테스트
npm run test:scripts                        # scripts 디렉토리 테스트
```

**단일 테스트 파일 실행:**
```bash
cd packages/cli  # 또는 packages/core
npx vitest run path/to/test-file.test.ts
```

**코드 품질:**
```bash
npm run lint         # ESLint 실행
npm run lint:fix     # ESLint 자동 수정 및 Prettier 포맷팅
npm run format       # Prettier 포맷팅만 실행
npm run typecheck    # 모든 패키지의 TypeScript 타입 검사
```

**디버깅:**
```bash
npm run debug        # Node 디버거로 CLI 시작 (--inspect-brk)
DEV=true npm start   # React DevTools 지원 활성화
DEBUG=1 gemini       # 샌드박스 컨테이너 디버그
```

## 저장소 구조

```
packages/
├── cli/                    # 프론트엔드: 사용자 대면 CLI, UI 렌더링, 히스토리
│   └── src/
│       ├── ui/            # React 컴포넌트 (Ink 기반 터미널 UI)
│       ├── commands/      # CLI 명령어 핸들러
│       ├── services/      # CLI 전용 서비스
│       └── gemini.tsx     # 메인 CLI 진입점
│
├── core/                   # 백엔드: Gemini API 클라이언트, 도구 실행, 오케스트레이션
│   └── src/
│       ├── tools/         # 내장 도구 (파일 시스템, 셸, 웹 가져오기 등)
│       ├── agents/        # 에이전트 구현
│       ├── services/      # 코어 서비스
│       ├── prompts/       # 프롬프트 구성
│       ├── mcp/           # Model Context Protocol 통합
│       └── index.ts       # 코어 API 내보내기
│
├── a2a-server/            # 실험적: Agent-to-Agent 서버
├── test-utils/            # 임시 파일 시스템용 테스팅 유틸리티
└── vscode-ide-companion/  # Gemini CLI를 위한 VS Code 확장

docs/                       # 모든 프로젝트 문서
scripts/                    # 빌드, 테스트 및 개발 자동화 스크립트
integration-tests/          # 종단 간 통합 테스트
```

## 아키텍처 흐름

**사용자 상호작용 흐름:**
1. **사용자 입력** → `packages/cli`가 터미널 입력 처리 및 UI 표시
2. **CLI → Core** → CLI가 사용자 프롬프트를 코어 패키지로 전송
3. **Core → Gemini API** → 코어가 컨텍스트, 히스토리, 도구 정의를 포함한 프롬프트 구성
4. **Gemini 응답** → API가 텍스트 응답 또는 도구 실행 요청 반환
5. **도구 실행** → 코어가 도구 실행 (쓰기 작업은 사용자 승인 필요)
6. **도구 결과 → API** → 결과를 Gemini로 다시 전송하여 최종 응답 생성
7. **Core → CLI** → 최종 응답을 CLI로 반환
8. **사용자에게 표시** → CLI가 포맷된 출력을 터미널에 렌더링

**주요 역할 분리:**
- `packages/cli`: 사용자 경험, 렌더링, 입출력, 테마 담당
- `packages/core`: API 통신, 도구 등록/실행, 상태 관리 담당
- 도구들은 `packages/core/src/tools/`의 모듈식 확장

## TypeScript 및 React 컨벤션

**타입 안전성:**
- `any` 타입 사용 금지—대신 `unknown`과 타입 좁히기 사용
- 타입 단언(`as Type`) 절대적으로 필요한 경우 외에는 피하기
- switch 문의 default 절에서 `checkExhaustive()` 헬퍼 사용 (`packages/cli/src/utils/checks.ts`에서 가져오기)

**코드 구조:**
- 클래스보다 TypeScript 인터페이스를 가진 일반 객체 선호
- 공개 API는 ES 모듈 export 사용; 내보내지 않은 것 = private
- 불변성을 위해 배열 연산자(`.map()`, `.filter()`, `.reduce()`) 활용

**React (Ink UI):**
- Hooks를 사용하는 함수형 컴포넌트만 사용
- 렌더 로직은 순수하게 유지—컴포넌트 본문에 부수효과 금지
- `useEffect`는 동기화 목적으로만 사용, 상태 업데이트용으로 사용 금지
- `useEffect` 내부에서 `setState` 호출 금지 (성능 저하)
- `useEffect` 의존성 배열에 모든 의존성 포함
- `useMemo`, `useCallback`, `React.memo` 생략—React 컴파일러가 최적화 처리
- 이전 상태 기반 상태 업데이트 시 함수형 업데이트 사용: `setState(prev => prev + 1)`

**테스팅 (Vitest):**
- 테스트 파일은 소스와 함께 위치: `*.test.ts` (로직) 또는 `*.test.tsx` (React 컴포넌트)
- ES 모듈용 `vi.mock()` 사용; 중요한 mock은 파일 최상단에 배치
- `vi.mock` 팩토리 전에 필요한 mock 함수는 `vi.hoisted()` 사용
- React 컴포넌트는 `ink-testing-library`의 `render()` 사용하고 `lastFrame()`으로 검증
- 내부 모듈과 외부 SDK를 자주 mock: `@google/genai`, `@modelcontextprotocol/sdk`, `fs`, `os`, `child_process`
- 비동기 타이머 테스트는 `vi.useFakeTimers()` 및 `vi.advanceTimersByTimeAsync()` 사용

## 중요 참고사항

**메인 브랜치:** `main`

**Node.js 버전:**
- 개발: Node.js `~20.19.0` 사용 (개발 의존성 이슈로 특정 버전 필요; nvm 사용 권장)
- 프로덕션: `>=20` 모든 버전 허용

**샌드박싱:**
- macOS: 기본적으로 Seatbelt(`sandbox-exec`) 사용
- 크로스 플랫폼: `GEMINI_SANDBOX=docker` 또는 `GEMINI_SANDBOX=podman` 설정
- 샌드박스 빌드: `npm run build:sandbox` (`build:all`에 포함)

**Import 제약사항:**
- ESLint가 패키지 경계 강제—패키지 간 상대 import 제약 준수

**플래그 네이밍:**
- 플래그 이름에 하이픈 사용: `--my-flag` (`--my_flag` 아님)

**주석:**
- 고가치 주석만 작성; 주석으로 사용자와 소통하지 않기

**문서화:**
- 사용자 대면 변경사항(새 명령어, 수정된 동작, 플래그)은 `/docs` 업데이트 필수
- [Google Developer Documentation Style Guide](https://developers.google.com/style) 준수

## 추가 컨텍스트

**GEMINI.md vs CLAUDE.md:**
- 이 코드베이스는 상세한 코딩 컨벤션과 테스팅 패턴을 위해 `GEMINI.md`를 사용합니다. 다음 내용 참조:
  - 상세한 TypeScript/React 가이드라인
  - 포괄적인 테스팅 패턴 및 mocking 전략
  - ES 모듈 모범 사례
  - 타입 좁히기 및 완전성 검사
- `CLAUDE.md` (또는 이 파일)는 생산성을 위한 고수준 아키텍처와 일반 명령어에 초점

**기여하기:**
- 모든 PR은 기존 이슈에 연결되어야 함
- PR은 작고 단일 변경에 집중
- 제출 전 `npm run preflight` 실행
- 커밋 메시지는 [Conventional Commits](https://www.conventionalcommits.org/) 준수
- 프론트엔드 자동 리뷰를 위해 `/review-frontend <PR_NUMBER>` 사용 (실험적)
