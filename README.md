# Skills

Claude Code 커스텀 스킬 모음.

## 포함된 스킬

| 스킬 | 설명 |
|------|------|
| `bug-investigate` | Chrome DevTools MCP를 이용한 프론트엔드 버그 재현 및 root cause 분석. CDP로 자동 재현하고, 콘솔 에러·네트워크 데이터를 수집하여 원인 분석 문서를 작성한다. |

## 설치

Claude Code 플러그인으로 설치한다.

```bash
claude plugins add wilson-hackle/skills
```

설치 후 Claude Code를 재시작하면 스킬이 자동으로 인식된다.

## 사용법

Claude Code 대화에서 슬래시 커맨드로 호출한다.

```
/bug-investigate
```

또는 자연어로 요청하면 Claude가 관련 스킬을 자동으로 인식하여 사용한다.

## 스킬 구조

```
skills/
└── {skill-name}/
    └── SKILL.md        # 스킬 정의 (frontmatter + 프롬프트)
```

각 스킬은 `SKILL.md` 파일 하나로 구성된다. frontmatter에 이름, 설명, 옵션을 정의하고, 본문에 워크플로우를 작성한다.

## 라이선스

MIT
