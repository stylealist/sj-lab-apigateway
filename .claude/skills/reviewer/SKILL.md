---
name: reviewer
description: 이 저장소(sj-lab-apigateway)의 현재 변경사항(git diff)을 프로젝트 전용 관점으로 리뷰한다. "리뷰해줘", "코드 리뷰", "review" 같은 요청이 이 저장소에서 들어왔을 때 사용한다.
---

# reviewer

이 저장소 전용 코드 리뷰 스킬입니다. 범용 `/code-review`보다 이 프로젝트(Spring Cloud Gateway API 게이트웨이)의 구조에 특화되어 있습니다.

## 실행 방법

1. 리뷰 대상이 인자로 주어지지 않았다면 `git diff` (staged + unstaged)와 `git status`로 변경 파일을 확인한다.
2. `.claude/agents/reviewer.md`에 정의된 `reviewer` 서브에이전트를 Agent 도구로 호출하여 실제 리뷰를 수행시킨다. 프롬프트에는 리뷰 대상 범위(커밋되지 않은 변경 전체, 특정 파일, 또는 PR)를 명시해서 전달한다.
3. `reviewer` 에이전트가 반환한 결과를 그대로 사용자에게 정리해서 보여준다 — 발견된 문제가 없으면 "발견된 문제 없음"이라고 짧게 안내한다.

## 참고

- 저장소 구조와 필터/라우팅 규칙은 `CLAUDE.md`를 참고한다.
- 파일 저장 직후 자동으로 도는 가벼운 리뷰는 `.claude/settings.json`의 `PostToolUse` hook(Edit|Write)이 이미 담당하므로, 이 스킬은 사용자가 명시적으로 리뷰를 요청했을 때 더 깊이 있게 점검하는 용도로 사용한다.
