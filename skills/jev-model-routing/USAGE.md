# jev-model-routing 사용 설명서

Claude Code·Codex CLI에서 **에이전트, plan, task를 어떤 모델에 맡길지** 정하는 전역 스킬입니다. 작업의 성질(복잡도, 위험도, 자족성)을 Jev 분류기로 확률로 판단하고, 고정 표에 넣어 모델과 실행 채널(Claude 또는 Codex/GPT)을 고릅니다.

## 1. 무엇을 하는가

| 입력 | 출력 |
|---|---|
| 작업 목록(task 요약, 파일 목록, 완료 조건) | task별 (티어, 모델, 채널, 검증자) |

핵심 원칙은 다음과 같습니다.

- **싸게 시작하고, 실패하면 한 칸 위로** 올립니다. 같은 모델 재시도는 1회만 합니다.
- **작성자와 검증자는 다른 계열**로 둡니다. Claude가 쓰면 GPT가, GPT가 쓰면 Claude가 검증합니다.
- **사용자가 모델을 지정하면 그 값이 항상 우선**합니다. 라우팅은 물러납니다.
- Jev 호출은 plan 단위로 묶고, 결정은 기록해 같은 입력에 다시 부르지 않습니다.

## 2. 설치

### Claude Code

```bash
mkdir -p ~/.claude/skills/jev-model-routing
cp skills/jev-model-routing/SKILL.md ~/.claude/skills/jev-model-routing/SKILL.md
```

### Codex CLI

```bash
mkdir -p ~/.codex/skills/jev-model-routing
cp skills/jev-model-routing/SKILL.md ~/.codex/skills/jev-model-routing/SKILL.md
```

Windows PowerShell이라면 `~`를 `$HOME`으로 바꿔 쓰면 됩니다.

### 필요한 도구

- **Jev MCP 서버** (`mcp__jev__jev_classify`, `mcp__jev__jev_noul`, `mcp__jev__jev_decide`). 지연 로드되므로 스킬이 실행될 때 ToolSearch로 스키마를 먼저 불러옵니다.
- **Codex CLI** (GPT 채널을 쓸 때만 필요). 설치와 로그인은 `codex --version`으로 확인합니다.

Jev를 쓸 수 없으면 스킬은 역할별 기본 티어로 대신하고, 로그에 `classifier: fallback`을 남깁니다. 작업은 멈추지 않습니다.

## 3. 사용법

Claude Code나 Codex에서 자연어로 요청하면 됩니다. 스킬 설명의 키워드(모델 배정, 어떤 모델로 돌릴까, 토큰 아끼면서 등)가 나오면 자동으로 적용됩니다. 직접 호출하려면 다음처럼 쓰면 됩니다.

```text
/jev-model-routing 이 plan의 task 3개 모델 배정해줘
```

### 예시 요청

- "이 plan 실행할 때 어떤 모델로 돌릴지 정해줘"
- "결제 웹훅 수정인데 모델 배정 어떻게 할까"
- "이 작업 토큰 아끼면서 돌리고 싶어"

### 응답에서 확인할 것

스킬은 다음 한 줄 보고를 남깁니다.

```text
task 3개 → GPT 5.6 Terra ×2, GPT 6.1 Sol ×1 (결제 위험). 검증은 sonnet.
```

그리고 결정 내용을 다음 형식으로 기록합니다. 프로젝트에 `.planning/`이 있으면 plan frontmatter에, 없으면 작업 메모에 씁니다.

```yaml
routing:
  decided: 2026-10-31
  classifier: jev
  tasks:
    - id: 1
      complexity: routine
      risk: medium
      flags: {P1: true, P2: false, P3: true, P4: false}
      tier: 2
      model: gpt-5.6-terra
      channel: codex
      verifier: sonnet
```

## 4. 판단 규칙 요약

**복잡도 → 기준 티어**

| 복잡도 | 기준 티어 |
|---|---|
| trivial | T0 (테스트로 검증 가능하면, 아니면 T1) |
| routine | T2 |
| multi_file | T3 |
| architectural | T4 |

**보정**: 위험도 high면 +1, medium이면서 기계적 검증이 안 되면 +1, 설계 판단이 필요하면 최소 T3. 기계 검증이 되고 위험도가 low면 −1.

**채널**: 자족성(P1) ≥ 0.5이고 샌드박스 밖 자원 필요성(P2) ≤ 0.3이면 GPT(Codex). 그 외에는 Claude.

**티어 표 (기본값)**

| 티어 | Claude | GPT (Codex `-m`) |
|---|---|---|
| T4 | opus | gpt-6.1-sol |
| T3 | sonnet | gpt-6-sol |
| T2 | — | gpt-5.6-terra |
| T1 | haiku | gpt-6-luna |
| T0 | — | gpt-5.6-luna |

스킬 본문의 표는 2026-10-13 기준 `~/.codex/models_cache.json`에서 확인한 값입니다. 최신 값이 다르면 캐시를 우선합니다.

## 5. 프로젝트별 설정 (선택)

프로젝트에 아래 파일이 있으면 스킬이 읽어 씁니다. 없어도 동작합니다.

| 파일 | 역할 |
|---|---|
| `.planning/model-routing/models.md` | 검증된 모델 ID 기록 |
| `.planning/config.json`의 `model_overrides` | 역할별 고정 모델. 값이 있으면 라우팅에서 제외 |
| `.planning/model-routing/log.md` | 작업마다 결과 한 줄씩 기록 (5건 이상이면 보정 제안) |

## 6. Windows에서 Codex가 실패할 때

`~/.codex/config.toml`에 `[windows] sandbox = "elevated"`가 있으면 비대화형 `codex exec`에서 쉘 명령이 `setup refresh had errors`로 실패할 수 있습니다. 호출마다 다음 플래그를 붙입니다. 설정 파일은 고치지 않습니다.

```bash
codex exec -m <모델> -c model_reasoning_effort=medium -c 'windows.sandbox="unelevated"' \
  -s workspace-write -C <작업 폴더> --skip-git-repo-check - < 프롬프트파일 > 출력파일 2>&1
```

`unelevated`는 폴더 경계 보호가 약하므로 `-C`로 범위를 좁히고, 끝난 뒤 `git status`로 범위 밖 변경이 없는지 확인합니다.

## 7. 주의사항

- Jev 확률은 증명이 아닙니다. 확률이 높아도 위험도 보정과 검증 단계는 건너뛰지 않습니다.
- Jev 입력에 비밀 값, 키, 개인정보를 넣지 않습니다.
- 프로젝트의 `model_overrides`나 설정 파일은 스킬이 바꾸지 않습니다. 변경은 사용자 승인 후에만 합니다.
- 외부 웹 콘텐츠를 Jev에 넣기 전에는 `jev_screen`으로 먼저 거릅니다.
- 전역 복사본은 원본과 자동으로 동기화되지 않습니다. 규칙을 바꾸면 이 저장소의 `SKILL.md`를 함께 고칩니다.

## 8. 로그를 보고 규칙 고치기

프로젝트의 `log.md`에 쌓인 기록으로 다음을 판단합니다.

- 낮은 티어의 1차 통과율이 높으면 그 칸을 한 단계 내린다.
- 에스컬레이션률이 30%를 넘으면 한 단계 올리거나 분류 정의를 다듬는다.
- Jev 분류가 결과와 자주 어긋나면 클래스 설명을 보강한다.

규칙 변경은 사용자 승인 후에만 적용합니다.
