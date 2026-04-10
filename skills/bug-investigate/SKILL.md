---
name: bug-investigate
description: Chrome DevTools MCP를 이용한 프론트엔드 버그 재현 및 root cause 분석. 버그 제보를 받으면 로컬 개발 서버에서 CDP로 자동 재현하고, 콘솔 에러·네트워크 데이터를 수집하여 원인을 분석한 문서를 ./docs/issues/에 작성한다. "버그가 있어요", "에러가 나요", "왜 안 되지", "재현해봐", "이슈 조사해줘", "프론트 디버깅", "화면이 깨져요", "동작이 이상해요" 등 프론트엔드 버그 관련 요청에 반드시 이 스킬을 사용하라.
---

# Bug Investigate

React 프로젝트의 버그 제보를 Chrome DevTools MCP로 자동 재현하고, 수집 데이터와 소스 코드를 교차 분석하여 root cause 문서를 작성한다.

## Prerequisites

- **chrome-devtools-mcp**: Claude Code MCP 서버로 등록되어 있어야 한다
- **Chrome 브라우저**: 실행 중이어야 한다 (MCP가 CDP로 연결)
- **로컬 개발 서버**: 재현 대상 앱이 실행 중이어야 한다

시작 시 `list_pages`를 호출하여 MCP 연결과 Chrome 상태를 확인한다. 실패하면 사용자에게 Chrome 실행 및 MCP 설정을 안내한다. 로컬 개발 서버 실행 여부는 `navigate_page`로 대상 URL에 접근하여 확인한다.

## Workflow

### Phase 1: 버그 정보 수집

사용자에게 **버그 설명**과 **재현 URL**만 확인한다. 이 두 가지가 있으면 시작할 수 있다.

기대 동작, 재현 조건 등은 사용자가 자발적으로 제공하면 활용하고, 아니면 Phase 2 코드 탐색에서 유추한다. 사용자에게 질문 폭격을 하지 않는다.

### Phase 2: 코드베이스 탐색

버그 재현 전에 관련 코드를 파악한다. 목적은 두 가지:
1. **재현에 필요한 정보** — UI 요소의 셀렉터, 인터랙션 순서, 폼 필드명
2. **root cause 분석 후보** — 상태 관리, API 호출, 에러 핸들링 지점

탐색 대상:
- 해당 URL의 라우팅 → 페이지 컴포넌트 매핑
- 컴포넌트 트리 (부모-자식 관계, props 흐름)
- 상태 관리 코드 (useState, useReducer, Context, Redux/Zustand 등)
- API 호출 코드 (fetch, axios, react-query/tanstack-query 등)
- 에러 바운더리, try-catch, 에러 상태 처리

Phase 1에서 기대 동작이 불분명했다면, 코드의 정상 흐름을 읽어서 기대 동작을 유추한다.

### Phase 3: 버그 재현 및 데이터 수집

Chrome DevTools MCP 도구로 버그를 재현하면서 동시에 디버깅 데이터를 수집한다. 재현과 수집은 분리된 단계가 아니다 — 재현 과정 자체가 콘솔 에러와 네트워크 요청을 발생시키므로, 재현 완료 후 바로 수집한다.

#### 재현 실행

1. `navigate_page`로 대상 페이지 이동
2. `take_snapshot`으로 페이지 요소와 uid를 확인 — 모든 인터랙션 도구는 CSS 셀렉터가 아닌 snapshot의 `uid`를 사용한다
3. 재현 단계를 순차 실행:
   - `click` — 버튼, 링크, 요소 클릭 (`dblClick: true`로 더블클릭 가능)
   - `fill` — 입력 필드에 값 입력 또는 `<select>` 옵션 선택 (기존 값 대체)
   - `fill_form` — 여러 폼 필드를 한 번에 입력
   - `hover` — 마우스 호버 (툴팁, 드롭다운 트리거)
   - `wait_for` — 특정 조건 대기 (요소 출현, 네트워크 idle 등)
   - `handle_dialog` — 브라우저 alert/confirm/prompt 대응
   - `evaluate_script` — 키보드 이벤트 디스패치 등 도구로 직접 지원되지 않는 인터랙션 처리
4. 인터랙션 후 DOM이 변경되면 `take_snapshot`으로 최신 uid를 다시 확인한다
5. 각 주요 단계 후 `take_screenshot`으로 시각적 상태 기록

**요소 식별**: `take_snapshot`이 반환하는 텍스트 스냅샷에서 요소의 `uid`를 찾는다. Phase 2에서 파악한 `data-testid`, ARIA role, `id` 등은 스냅샷에서 올바른 요소를 식별하는 단서로 활용한다.

**요소를 찾을 수 없을 때:**
1. `take_snapshot`으로 최신 DOM 구조 재확인 — 이전 인터랙션으로 DOM이 변경되었을 수 있다
2. `take_screenshot`으로 시각적 상태 확인 — 로딩 중이거나 모달이 가려진 경우
3. `wait_for`로 요소 출현 대기 후 재시도
4. Phase 2의 코드를 다시 참조하여 인터랙션 순서 수정

#### 데이터 수집

재현 단계를 실행한 직후 수집한다:

**콘솔 로그:**
- `list_console_messages`로 전체 콘솔 메시지 조회 (에러, 경고, 스택 트레이스 포함)
- 추가 디버깅이 필요하면 `evaluate_script`로 특정 값이나 상태를 직접 조회

**네트워크:**
- `list_network_requests`로 전체 요청 목록 조회
- 실패한 요청(4xx, 5xx)이나 의심스러운 요청에 `get_network_request`로 상세 확인 (요청/응답 body 포함)

**React 상태 (선택):**
`evaluate_script`로 React 내부 상태를 조회할 수 있다:
```javascript
// Error Boundary가 에러를 잡았는지
document.querySelectorAll('[data-error-boundary], [role="alert"]')

// Redux store 상태 (있다면)
window.__REDUX_DEVTOOLS_EXTENSION__?.store?.getState()
```

**스크린샷:**
- 버그가 재현된 상태에서 `take_screenshot`
- 에러 메시지, UI 깨짐 등 시각적 증거 확보

#### CDP 재현이 안 되는 경우

자동화로 재현이 어려운 상황이 있다 — 특정 유저 세션/인증 상태, 미묘한 타이밍 의존, 드래그 앤 드롭 같은 복잡한 인터랙션 등.

이 경우 다음 순서로 대응한다:

1. **사용자에게 수동 재현 요청**: 사용자가 직접 브라우저에서 버그를 재현하도록 안내하고, 그 상태에서 `list_console_messages`와 `list_network_requests`로 데이터만 수집한다
2. **코드 분석 중심 전환**: 수집 가능한 데이터가 부족하면, Phase 2에서 파악한 코드를 기반으로 정적 분석에 집중한다. 버그 설명에서 추론 가능한 실행 경로를 따라가며 의심 지점을 찾는다
3. **문서에 재현 방식 명시**: 자동 재현이 아닌 수동 재현 또는 코드 분석 기반임을 문서에 기록한다

### Phase 4: 원인 분석

수집 데이터와 소스 코드를 교차 분석하여 root cause를 찾는다:

1. **에러 메시지 추적**: 콘솔 에러 → 소스 코드에서 해당 에러를 throw/log하는 지점 추적
2. **네트워크 실패 분석**: 실패한 API → 호출 코드 → 에러 핸들링 확인 → 서버 응답 body 확인
3. **상태 흐름 추적**: 버그 발생 시점의 상태가 어떻게 잘못되었는지 역추적
4. **타이밍 이슈**: 레이스 컨디션, useEffect 의존성 누락, 비동기 처리 순서

root cause는 "어떤 조건에서 어떤 코드가 잘못된 동작을 하는지"에 대한 기술적 답변이다. 단순히 "에러가 발생했다"가 아니라 원인 코드와 조건을 특정한다.

### Phase 5: 문서 작성

`./docs/issues/`에 Markdown 문서를 작성한다. 디렉토리가 없으면 생성한다.

**파일명**: `YYYY-MM-DD-{영문-kebab-case-제목}.md` (예: `2026-04-09-login-submit-error.md`)

**스크린샷**: `./docs/issues/assets/` 디렉토리에 저장하고, 문서에서 상대 경로로 참조한다 (예: `![에러 화면](assets/2026-04-09-login-error-01.png)`).

## Output Format

```markdown
# {버그 제목}

> 조사일: {날짜}

## 버그 설명

{사용자 제보 원문 또는 요약}

- **기대 동작**: {정상이라면 어떻게 되어야 하는지}
- **실제 동작**: {실제로 발생하는 현상}

## 재현 단계

1. `{URL}`에 접속
2. ...
3. ...

**재현 방식**: {CDP 자동 재현 / 수동 재현 / 코드 분석}
**재현 환경**: Chrome, {로컬 서버 주소}

## 수집된 증거

### 콘솔
{관련 에러/경고 — 핵심만 발췌}

### 네트워크
{실패하거나 비정상적인 요청 — URL, status, 응답 요약}

### 스크린샷
{에러 상태를 보여주는 핵심 스크린샷}

## 원인 분석

### Root Cause
{기술적 원인 — 어떤 코드가 어떤 조건에서 잘못 동작하는지}

### 관련 코드
{파일 경로:라인 번호와 핵심 코드 스니펫}

### 영향 범위
{이 버그가 다른 기능에 미치는 영향 — 없으면 생략}
```

## Notes

- 민감한 데이터(토큰, 비밀번호 등)가 네트워크 로그에 포함될 수 있다. 문서 기록 시 마스킹한다.
- Chrome MCP 연결이 끊기면 사용자에게 Chrome 상태 확인을 요청한다.
