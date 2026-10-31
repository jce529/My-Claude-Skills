# phase-session-dispatch 사용 설명서

GSD 매니저(총괄 세션)에서 **승인한 phase마다 전용 세션을 포크**하고, 각 세션이 **jev-model-routing으로 모델을 고르며** 작업을 이어 가게 하는 전역 스킬입니다. 총괄은 진행도만 관리하고, 하위 세션의 보고는 사용자가 검증한 뒤에만 반영합니다.

## 1. 흐름

```
총괄(매니저)
 ├─ 승인된 phase 목록 확인 (예: 12, 13)
 ├─ phase 12 세션 포크 ──► plan·execute 전 jev-model-routing으로 모델 배정
 ├─ phase 13 세션 포크 ──► 같은 방식
 └─ 보고 수신 → 사용자 "검증 끝" 확인 → 진행 기록 반영 → 다음 승인 대기
```

## 2. 설치

### Claude Code

```bash
mkdir -p ~/.claude/skills/phase-session-dispatch
cp skills/phase-session-dispatch/SKILL.md ~/.claude/skills/phase-session-dispatch/SKILL.md
```

### Codex CLI

```bash
mkdir -p ~/.codex/skills/phase-session-dispatch
cp skills/phase-session-dispatch/SKILL.md ~/.codex/skills/phase-session-dispatch/SKILL.md
```

모델 배정 단계가 제대로 동작하려면 [jev-model-routing](../jev-model-routing/USAGE.md)도 함께 설치하세요.

## 3. 사용 예시

- "phase 12, 13 세션으로 돌려줘" → 두 phase만 세션을 만듭니다.
- "매니저에서 다음 phase dispatch" → 대상 phase를 한 줄로 보여주고 승인을 받은 뒤에 만듭니다.
- "phase 12 세션 보고 반영해" → 사용자가 검증을 끝냈는지 확인한 뒤 진행 기록을 맞춥니다.

## 4. 주의

- 세션은 worktree에서 돌아가므로 마지막 커밋 기준으로 시작합니다. 필요한 변경은 먼저 커밋하세요.
- phase 브랜치는 자동으로 병합되지 않습니다. 병합은 직접 지시할 때만 합니다.
- 클라우드 세션은 보고가 돌아오지 않을 수 있어 기본 대상이 아닙니다.
