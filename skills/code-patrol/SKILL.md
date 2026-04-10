---
name: code-patrol
description: 최근 24시간 변경된 TypeScript/React 코드에서 버그와 취약점을 찾아 Jira 티켓으로 등록한다. 코드를 절대 수정하지 않는다. 퇴근 전이나 코드 리뷰 전에 /code-patrol을 실행하면 된다.
disable-model-invocation: true
---

# Code Patrol

최근 24시간 변경된 TypeScript/React 코드를 2패스로 분석하여 버그와 취약점을 찾고, Jira 티켓으로 등록한다.

**핵심 원칙: 코드를 절대 수정하지 않는다. 이슈를 Jira에 등록만 한다.**

## Prerequisites

- **Git 저장소**: 현재 디렉토리가 git 저장소여야 한다
- **TypeScript/React 프로젝트**: `.ts` 또는 `.tsx` 파일이 존재해야 한다
- **Atlassian 플러그인**: `mcp__plugin_atlassian_atlassian` MCP 도구가 사용 가능해야 한다
- **Serena MCP** (권장): Phase 2 심층 분석에서 시맨틱 도구를 활용한다. 없으면 기본 도구로 대체 가능하다

시작 시 `git rev-parse --is-inside-work-tree`로 git 저장소 여부를 확인한다. 실패하면 사용자에게 안내하고 종료한다.

### Jira 설정

| 항목 | 값 |
|---|---|
| cloudId | `dalbang.atlassian.net` |
| projectKey | `HACKLE` |
| issueTypeName | `하위 작업` |
| parent | `HACKLE-13937` |
| assignee | `712020:d679c4fb-c2d4-4820-890f-726ddffc8094` (wilson) |
| contentFormat | `markdown` |
| label | `code-patrol` |

## Workflow

### Phase 0: 상태 확인 및 초기화

이 스킬은 대량 파일 분석으로 rate limit에 도달할 수 있다. 상태 파일(`.code-patrol-state.json`)로 진행 상황을 저장해서, 중단되더라도 재실행 시 이어서 분석할 수 있도록 한다.

1. `.code-patrol-state.json` 파일이 존재하는지 확인한다:
   - **존재하면**: 재시도 실행이다. `retryCount`를 읽고 1 증가시켜 저장한다.
     - `retryCount`가 0이었으면 (증가 후 1): 첫 번째 재시도. schedule을 재등록하지 않는다. 이전 `progress`를 로드하여 이어서 진행한다.
     - `retryCount`가 1 이상이었으면 (증가 후 2+): 재시도를 이미 했는데 또 상태 파일이 남아있는 상황. 기존 `issues`로 즉시 Jira 티켓을 등록(Phase 3)하고 종료한다.
     - `progress.pass1Done`이 true면 Phase 1을 건너뛰고 Phase 2부터 이어서 진행한다.
   - **존재하지 않으면**: 첫 실행이다. 아래 단계를 수행한다.

2. 상태 파일을 생성한다:
   ```json
   {
     "status": "running",
     "retryCount": 0,
     "startedAt": "<현재 ISO 시각>",
     "scheduledJobId": null,
     "progress": {
       "pass1Done": false,
       "pass1Results": [],
       "analyzedFiles": [],
       "issues": []
     }
   }
   ```

3. CronCreate로 5시간 후 1회 실행 schedule을 등록한다 (rate limit 안전망):
   - 현재 시각에 5시간을 더한 시각으로 cron 표현식을 계산한다 (예: 22:00 시작이면 `"0 3 {내일날짜} {월} *"`)
   - `prompt`: `/code-patrol`
   - `recurring`: false
   - 반환된 job ID를 상태 파일의 `scheduledJobId`에 저장한다

정상 완료 시 Phase 4에서 이 schedule을 취소하므로, rate limit으로 세션이 끊긴 경우에만 자동 재실행된다.

### Phase 1: 변경 파일 스캔

메인 컨텍스트에서 가볍게 수행한다. 파일 내용을 읽지 않는다.

1. `git log --since="24 hours ago" --name-only --pretty=format:""` 실행하여 최근 변경 파일 목록을 수집한다. 중복을 제거한다.

2. `.ts`, `.tsx` 파일만 필터링한다. 다음은 제외한다:
   - `*.test.ts`, `*.test.tsx`, `*.spec.ts`, `*.spec.tsx`
   - `*.stories.ts`, `*.stories.tsx`
   - `*.d.ts`
   - `node_modules/` 하위

3. 변경 파일이 없으면 "최근 24시간 변경 없음"을 사용자에게 알리고, 상태 파일 삭제 + schedule 취소(`scheduledJobId`가 있으면 CronDelete) 후 종료한다.

4. `search_for_pattern`(Serena) 또는 Grep으로 빠른 사전 필터링 — 변경된 파일들에 대해 다음 패턴을 검색하여 태그를 부여한다:

   | 패턴 (정규식) | 태그 |
   |---|---|
   | `: any\b` 또는 `as any\b` | `[any-type]` |
   | `@ts-ignore` 또는 `@ts-expect-error` | `[ts-suppress]` |
   | `eslint-disable` | `[eslint-disable]` |
   | `dangerouslySetInnerHTML` | `[dangerous-html]` |
   | `catch\s*\([^)]*\)\s*\{\s*\}` | `[empty-catch]` |
   | `.then\(` 이 있고 같은 Promise 체인에 `.catch(`가 없는 경우 | `[unhandled-promise]` |
   | `sk-\|api_key\|password\s*=` | `[hardcoded-secret]` |

5. 태그가 하나도 부여되지 않은 파일은 의심 목록에서 제외한다.

6. 의심 파일이 하나도 없으면 "이슈 없음"을 사용자에게 알리고, 상태 파일 삭제 + schedule 취소 후 종료한다.

7. 상태 파일 업데이트: `pass1Done: true`, `pass1Results`에 `[{"file": "경로", "tags": ["태그1", ...]}]` 저장

### Phase 2: 심층 분석 (파일별 서브에이전트)

**메인 컨텍스트에서 파일을 직접 분석하지 않는다.** 각 파일을 독립된 서브에이전트(Agent 도구)에 위임하여 컨텍스트를 작게 유지한다. 이렇게 하면 파일 수가 많아도 분석 정확도가 일정하게 유지된다.

재시도 시에는 상태 파일의 `analyzedFiles`에 이미 있는 파일은 건너뛴다.

#### 서브에이전트 실행

Phase 1의 의심 파일 목록에서 각 파일에 대해 Agent 도구로 서브에이전트를 실행한다. 독립적인 파일들은 병렬로 실행한다.

각 서브에이전트에 전달할 프롬프트:

```
다음 파일을 분석하여 버그와 취약점을 찾아라. 코드를 수정하지 말고 이슈 목록만 JSON으로 반환하라.

파일: {파일경로}
태그: {Phase 1에서 부여된 태그들}

Serena MCP 사용 시 먼저 activate_project("{프로젝트 경로}")를 호출하라.

분석 순서:
1. git diff로 최근 24시간 변경 부분을 확인한다
2. Serena의 get_symbols_overview로 파일 구조를 파악한다
3. Serena의 read_file로 파일을 읽고 태그된 패턴이 실제 문제인지 판단한다
4. find_symbol, find_referencing_symbols로 타입 정의와 사용처를 추적하여 영향 범위를 파악한다

False positive 필터링 — 다음은 이슈로 보고하지 않는다:
- any 타입: 주석으로 사유가 명시된 의도적 사용
- 빈 catch: 주석으로 의도적 무시가 명시된 경우
- eslint-disable: 정당한 사유 주석이 있는 경우
- @ts-expect-error: 테스트에서 의도적 타입 오류를 만드는 경우

추가 분석 항목:
- React 무한 리렌더링: useEffect 내 setState가 의존성 배열의 값을 변경
- Stale closure: useCallback/useMemo에서 의존성 배열 누락
- 메모리 누수: addEventListener 후 cleanup 없음
- 레이스 컨디션: 비동기 호출 후 unmount 미처리
- null/undefined 미처리: optional 필드에 직접 접근

심각도:
- critical: 런타임 에러 확실, 보안 취약점, 데이터 손실
- warning: 특정 조건에서 버그 가능, 불안정한 패턴
- info: 코드 스멜, 개선 권장

민감 정보(API 키, 토큰 등)는 마스킹하라 (예: sk-...xxxx).

결과를 다음 JSON 형식으로 반환하라:
[{"file": "경로", "line": 번호, "severity": "critical|warning|info", "title": "제목", "description": "설명", "evidence": "코드 스니펫", "suggestion": "제안", "commit": "hash"}]

이슈가 없으면 빈 배열 []을 반환하라.
```

#### 결과 취합

각 서브에이전트가 반환한 JSON 이슈 목록을 취합한다. 각 파일 분석 완료 시마다 상태 파일의 `analyzedFiles`와 `issues`를 업데이트한다.

### Phase 3: Jira 티켓 등록

이슈가 하나도 없으면 (모두 false positive로 필터링됨) "분석 완료 — 이슈 없음"을 알리고, 상태 파일 삭제 + schedule 취소 후 종료한다.

#### 중복 확인

각 이슈를 등록하기 전에 `searchJiraIssuesUsingJql`로 기존 티켓을 검색하여 중복을 방지한다:

```
jql: project = HACKLE AND summary ~ "[Code Patrol]" AND description ~ "{파일명}:{라인}" AND status != Done
cloudId: dalbang.atlassian.net
```

이미 동일한 이슈가 있으면 건너뛰고 사용자에게 "이미 등록됨: HACKLE-123" 형태로 알린다.

#### 티켓 생성

상태 파일의 `issues`를 심각도 순으로 정렬한다: critical -> warning -> info.

각 이슈마다 `createJiraIssue`로 HACKLE-13937의 **하위 작업**을 생성한다:

```
cloudId: dalbang.atlassian.net
projectKey: HACKLE
issueTypeName: 하위 작업
parent: HACKLE-13937
assignee_account_id: 712020:d679c4fb-c2d4-4820-890f-726ddffc8094
summary: {아래 티켓 제목 형식}
description: {아래 티켓 설명 형식}
contentFormat: markdown
additional_fields: {"labels": ["code-patrol"]}
```

#### 티켓 제목 형식

제목만 보고도 어디에서 무슨 문제인지 바로 파악할 수 있어야 한다:

```
[Code Patrol] {심각도 이모지} {파일명}:{라인} — {핵심 문제 한 줄}
```

심각도 이모지: critical = `🔴`, warning = `🟡`, info = `🔵`

예시:
- `[Code Patrol] 🔴 UserProfile.tsx:44 — dangerouslySetInnerHTML에 사용자 데이터 직접 렌더링`
- `[Code Patrol] 🔴 ApiService.ts:17 — 하드코딩된 API 키 노출`
- `[Code Patrol] 🟡 UserProfile.tsx:19 — useEffect 의존성 배열에 userId 누락`
- `[Code Patrol] 🔵 ApiService.ts:20 — @ts-ignore 사유 미명시`

#### 티켓 설명 형식

````markdown
### 문제
{구체적 설명}

### 근거
```
{코드 스니펫}
```

### 파일
{파일경로}:{라인번호}

### 변경 커밋
{short hash} — {커밋 메시지}

### 제안
{수정 방향}

---
_이 티켓은 Code Patrol이 {YYYY-MM-DD} 분석에서 자동 생성했습니다._
````

### Phase 4: 정리

정상 완료 시:
1. 상태 파일의 `scheduledJobId`가 있으면 CronDelete로 schedule을 취소한다
2. `.code-patrol-state.json` 파일을 삭제한다
3. 사용자에게 결과 요약을 출력한다:
   - 스캔 파일 수, 심층 분석 파일 수, 발견 이슈 수 (심각도별)
   - 생성된 Jira 티켓 목록 (티켓 키 + 제목)
   - 중복으로 건너뛴 이슈 수

## Notes

- **읽기 전용**: 소스 코드, 설정 파일, 패키지 파일 등 프로젝트 파일을 절대 수정하지 않는다. 유일한 쓰기 작업은 `.code-patrol-state.json` 상태 파일과 Jira 티켓 생성이다.
- **민감 정보**: 코드에서 발견한 하드코딩된 시크릿, API 키, 토큰 등은 Jira 티켓에 전체 값을 기록하지 않고 마스킹한다 (예: `sk-...xxxx`).
- **상태 파일**: `.code-patrol-state.json`은 `.gitignore`에 추가를 권장한다. 이 파일이 실행 후에도 남아있다면 이전 실행이 비정상 종료된 것이다.
- **Atlassian 플러그인 필수**: `mcp__plugin_atlassian_atlassian` 도구가 사용 가능해야 한다. 연동되지 않았으면 사용자에게 안내하고 종료한다.
