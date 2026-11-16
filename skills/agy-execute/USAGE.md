# agy-execute 사용 설명서

GSD 사이클에서 **설계·검증은 Claude Code가, 실제 코딩은 agy CLI가** 맡도록 역할을 나누는 스킬입니다. `/gsd:plan-phase`로 승인된 PLAN.md를 agy에게 넘기고, 끝나면 Claude Code가 diff를 리뷰합니다.

## 1. 흐름

```
Claude Code
 ├─ discuss-phase / plan-phase 로 PLAN.md 작성·승인
 ├─ PLAN.md의 @경로를 펼쳐 agy 프롬프트 준비 (context handoff)
 ├─ agy -p "<프롬프트>" --mode accept-edits ──► agy가 코드 작성·커밋
 └─ git diff 리뷰 → 빌드 확인 → STATE.md 갱신 → verify-work
```

## 2. 준비물

- agy CLI가 설치되어 PATH에 있고 계정에 로그인되어 있어야 합니다. `agy --version`, `agy models`로 확인하세요.
- GSD(`.planning/` 디렉터리와 `/gsd:*` 명령)를 쓰는 프로젝트를 전제로 합니다.

## 3. 설치

```bash
mkdir -p ~/.claude/skills/agy-execute
cp skills/agy-execute/SKILL.md ~/.claude/skills/agy-execute/SKILL.md
```

특정 프로젝트에서만 쓰려면 `<프로젝트>/.claude/skills/agy-execute/SKILL.md`에 두면 됩니다.

### 권한 허용 (선택)

agy 호출마다 승인 창이 뜨는 것을 없애려면 `.claude/settings.local.json`의 `permissions.allow`에 추가합니다.

```json
"Bash(agy *)"
```

이 규칙은 `--dangerously-skip-permissions`가 붙은 호출도 통과시킵니다. 막으려면 `permissions.deny`에 `"Bash(agy *--dangerously-skip-permissions*)"`를 함께 넣으세요.

## 4. 프로젝트에 맞게 바꿀 곳

- `SKILL.md`의 `<빌드 명령>`을 프로젝트의 컴파일 확인 명령으로 바꿉니다 (예: `dotnet build "<솔루션>.sln" --nologo -v quiet`, `npx tsc --noEmit`).
- 모델 ID는 `agy models` 출력에 있는 값만 씁니다. 추론 강도는 ID 접미사(`-low`/`-medium`/`-high`)에 들어 있습니다.

## 5. 사용 예시

- "agy로 실행해줘" → 승인된 PLAN.md를 agy에게 넘겨 구현합니다.
- "실행은 agy한테 넘겨" → 같은 동작입니다.
- "이건 네가 직접 고쳐줘" → 예외로 Claude Code가 직접 수정합니다.

## 6. 주의

- agy는 GSD의 `@경로` 문법을 모릅니다. 스킬 0번 단계(context handoff)를 생략하면 참고 파일 내용이 전달되지 않습니다.
- `--dangerously-skip-permissions`는 사용자 확인 없이 쓰지 않습니다.
- 모델을 직접 고르려면 [jev-model-routing](../jev-model-routing/USAGE.md)의 Antigravity 채널 표를 참고하세요.
