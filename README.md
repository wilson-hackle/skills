# Skills

Claude Code 커스텀 스킬 모음.

## 포함된 스킬

| 스킬              | 설명                                                                                                                                                       |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bug-investigate` | Chrome DevTools MCP를 이용한 프론트엔드 버그 재현 및 root cause 분석. CDP로 자동 재현하고, 콘솔 에러·네트워크 데이터를 수집하여 원인 분석 문서를 작성한다. |

## 설치

### 전체 설치

```sh
npx skills add wilson-hackle/skills -a claude-code
```

### 개별 설치

```sh
npx skills add wilson-hackle/skills --skills bug-investigate a claude-code
```
