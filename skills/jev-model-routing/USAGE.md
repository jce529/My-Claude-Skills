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

- **Jev MCP 서버**: Claude Code 플러그인(claude-jev)의 `mcp__plugin_claude-jev_jev__*`(`jev_ask`, `jev_choice`, `jev_noul`, `jev_score`, `jev_doctor`). 지연 로드되므로 스킬이 실행될 때 ToolSearch로 스키마를 먼저 불러옵니다.
- **Codex CLI** (GPT 채널을 쓸 때만 필요). 설치와 로그인은 `codex --version`으로 확인합니다.
- **Antigravity CLI `agy`** (Gemini 4 Argon 채널을 쓸 때만 필요). Antigravity 계정으로 로그인한 상태에서 `agy models`에 `gemini-4-argon-*`가 보여야 합니다.

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

스킬 본문의 표는 2026-11-02 기준 `~/.codex/models_cache.json` 확인과 실제 호출로 검증한 값입니다(T2는 `gpt-6.1-terra`로 바뀌었습니다). 최신 값이 다르면 캐시를 우선합니다.

**Antigravity 채널 (Gemini 4 Argon, 선택)**

Claude·GPT와 별개의 세 번째 계열입니다. 자동 배정에는 넣지 않았고, 사용자가 지정하거나 호출 템플릿으로 명시적으로 고를 때만 씁니다.

| 티어(가설) | `agy --model` |
|---|---|
| T4 | gemini-4-argon-high |
| T3 | gemini-4-argon-medium |
| T2 | gemini-4-argon-low |

```bash
agy -p "<프롬프트>" --model gemini-4-argon-<강도> --mode plan --output-format text
```

- 기본은 읽기 전용(`--mode plan`)입니다. 파일 수정이 필요할 때만 `--mode accept-edits`를 쓰고 `--add-dir`로 범위를 좁힙니다. `--dangerously-skip-permissions`는 쓰지 않습니다.
- 티어 배치는 공개 벤치마크 보도를 보고 한 가설입니다. 로그로 보정하기 전까지 고정 사실로 쓰지 않습니다.
- Argon이 쓴 결과는 Claude나 GPT가, Argon이 검증하면 작성자는 Claude나 GPT가 맡습니다.
- 구독과 Gemini API 과금은 별개입니다. 목록에 모델이 없거나 호출이 실패하면 Claude·GPT 채널로 폴백합니다.

## 5. 프로젝트별 설정 (선택)

프로젝트에 아래 파일이 있으면 스킬이 읽어 씁니다. 없어도 동작합니다.

| 파일 | 역할 |
|---|---|
| `.planning/model-routing/models.md` | 검증된 모델 ID 기록 |
| `.planning/config.json`의 `model_overrides` | 역할별 고정 모델. 값이 있으면 라우팅에서 제외 |
| `.planning/model-routing/log.md` | 작업마다 결과 한 줄씩 기록 (5건 이상이면 보정 제안) |

## 6. Windows에서 Codex가 실패할 때

2026-11-02 기준 기본값 `[windows] sandbox = "elevated"` 그대로 쉘·`apply_patch`·vitest가 정상 동작합니다(codex-cli 0.166.0 확인). 설정 파일은 고치지 않습니다.

```bash
codex exec -m <모델> -c model_reasoning_effort=medium \
  -s workspace-write -C <작업 폴더> --skip-git-repo-check - < 프롬프트파일 > 출력파일 2>&1
```

- 작성자와 검증자 모두 `-s workspace-write`를 씁니다. `read-only`는 vitest 등이 시스템 Temp에 폴더를 만들다 `EPERM`으로 실패합니다. 대신 `-C`로 범위를 좁히고, 프롬프트에 수정 허용 파일 목록을 적고, 끝난 뒤 `git status`로 범위 밖 변경이 없는지 확인합니다.
- 테스트는 `npx vitest`가 아니라 `node node_modules/vitest/vitest.mjs run <파일>`로 호출하게 합니다(PowerShell에서 `npx.ps1`이 실행 정책에 막힙니다).
- 출력에 `setup refresh`/`Rejected`가 보이면 `-c 'windows.sandbox="unelevated"'`를 붙여 1회 재시도하고, 그래도 실패하면 Claude로 폴백합니다.

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
