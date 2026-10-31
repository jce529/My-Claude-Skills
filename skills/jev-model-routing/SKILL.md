---
name: jev-model-routing
description: 에이전트·plan·task를 어떤 모델(Claude Opus/Sonnet/Haiku, GPT 6.1 Sol/6 Sol/6 Luna/5.6 Terra/5.6 Luna)에 배정할지 Jev로 분류해 정한다. 모델을 고르는 모든 순간(plan 실행, 서브에이전트 호출, Codex 위임, 작업 분배), 또는 "모델 배정", "어떤 모델로 돌릴까", "토큰 아끼면서" 같은 요청에 사용한다.
---

# jev-model-routing (전역판)

이 스킬은 어느 프로젝트에서든 쓰는 **모델 라우팅 규칙**이다. 프로젝트 고유 값(모델 ID 확인표, 사용자 override, 로그)은 프로젝트 안에 있을 때만 읽고, 없으면 아래 기본값을 쓴다.

## 0. 시작 전 확인

1. **사용자가 모델을 지정했는가?** (대화 또는 프로젝트 override) → 그 값이 최우선. 라우팅은 여기서 끝난다.
2. **프로젝트 자료를 읽는다** (있는 것만):
   - `.planning/model-routing/models.md` — 검증된 모델 ID. 없으면 §1 기본 표 사용.
   - `.planning/config.json`의 `model_overrides` — 역할별 고정 모델. 값이 있는 역할은 라우팅 제외.
   - 없으면 `~/.codex/models_cache.json`(visibility=list)으로 Codex 모델 목록을 확인한다. 목록에 없는 ID는 쓰지 않는다.
3. **Jev 도구를 확인한다.** `mcp__jev__jev_classify`, `mcp__jev__jev_noul`, `mcp__jev__jev_decide`는 deferred 도구일 수 있으므로 ToolSearch로 스키마를 먼저 불러온다. 추측해서 호출하지 않는다.

## 1. 원칙

1. **판단은 Jev, 결정은 표.** Jev는 작업의 *성질*(복잡도·위험도·문맥 의존)만 확률로 분류한다. 모델 선택은 §4 고정 표가 한다. Jev에게 "어느 모델?"을 묻지 않는다.
2. **싸게 시작, 실패하면 한 칸 위로.** 같은 모델 재시도는 1회, 그다음은 한 티어 위. 처음부터 최상위는 위험도 high일 때만.
3. **작성자와 검증자는 다른 계열.** Claude가 쓰면 GPT가, GPT가 쓰면 Claude가 검증한다.
4. **Jev 호출은 묶어서.** task마다 따로 부르지 않고 plan(또는 작업 묶음) 단위 배치로 부른다. 결정은 plan frontmatter나 작업 메모에 기록하고 같은 입력에 다시 부르지 않는다.
5. **불확실하면 높은 쪽**, 단 한 칸 이상 올리지 않는다.
6. **측정한 것만 믿는다.** 티어 표는 가설이다. 로그로 보정하기 전까지 고정 사실처럼 쓰지 않는다.

## 2. 모델 카탈로그와 티어

| 티어 | 의미 | Claude (Agent `model`) | GPT (Codex `-m`) |
|---|---|---|---|
| T4 | 최상위 추론 | `opus` | `gpt-6.1-sol` |
| T3 | 강함 | `sonnet` | `gpt-6-sol` |
| T2 | 중간 | — | `gpt-5.6-terra` |
| T1 | 가벼움 | `haiku` | `gpt-6-luna` |
| T0 | 최경량·기계적 작업 | — | `gpt-5.6-luna` |

- 위 ID는 2026-10-13 `models_cache.json` 기준이다. `models.md`나 캐시가 다르면 **그쪽을 따른다.** 
- 지원하지 않는 ID를 임의로 바꾸지 않는다. 사용자에게 묻는다.
- 추론 강도 기본값은 `medium`, T4 또는 위험도 high는 `high`.
- 목록에만 있고 티어 표에 없는 모델(`gpt-6-astra`, `gpt-5.6-sol`)은 용도가 정해지기 전까지 쓰지 않는다.
- 숨김 모델(`gpt-reserve`, `gpt-5.5`, `codex-auto-review`)은 쓰지 않는다.

### 호출 규칙

- **Codex**: `-m`과 `-c model_reasoning_effort=<강도>`를 매번 명시하고, 실행 로그 첫머리 `model:` 줄이 요청한 값과 같은지 확인한다. 다르면 즉시 중단·보고.
- **Claude 서브에이전트**: `model` 파라미터를 명시한다.
- 호출 직전에 사용자에게 모델·강도·채널을 한 줄로 알린다.
- **Windows 주의**: `~/.codex/config.toml`에 `[windows] sandbox = "elevated"`가 있으면 비대화형 `codex exec`의 쉘·`apply_patch`가 `setup refresh had errors`로 실패할 수 있다. 이때는 호출마다 `-c 'windows.sandbox="unelevated"'`를 붙인다(설정 파일은 고치지 않는다).
  ```bash
  codex exec -m <모델> -c model_reasoning_effort=<강도> -c 'windows.sandbox="unelevated"' \
    -s workspace-write -C <작업 폴더> --skip-git-repo-check - < 프롬프트파일 > 출력파일 2>&1
  ```
  - unelevated는 폴더 경계 보호가 약하므로 `-C`로 범위를 좁히고, 프롬프트에 수정 허용 파일 목록을 명시하고, 끝난 뒤 `git status`로 범위 밖 변경이 없는지 확인한다.
  - 출력에 `setup refresh`/`Rejected`가 있으면 실패로 보고 Claude로 폴백한다.
- 샌드박스(`-s read-only`, `workspace-write`)는 네트워크와 레포 밖 접근을 막는다. 네트워크·원격 DB·브라우저가 필요한 작업은 Claude 채널로 보낸다.

### 채널 선택 (같은 티어 안에서 Claude vs GPT)

| 조건 | 채널 |
|---|---|
| 현재 대화 문맥에 크게 의존, 브라우저·MCP·Artifact·네트워크/원격 DB 필요 | **Claude** |
| 문서·파일 경로만으로 자족적인 구현·계획·테스트 작성 | **GPT (Codex)** |
| 위 둘 다 아님 | 작성자와 검증자가 다른 계열이 되도록 배분 |
| ChatGPT 사용 한도 초과 | Claude로 폴백하고 해제 시각을 사용자에게 알림 |

## 3. 역할별 기본 티어 (Jev 신호 반영 전 출발점)

| 작업 역할 | 기본 | 채널 | 비고 |
|---|---|---|---|
| 계획 작성(planner) | T3 | GPT 6 Sol | 위험도 high 또는 설계 결정이면 T4 |
| 자료 조사(researcher) | T2 | GPT 5.6 Terra | 웹 검색이 필요하면 Claude sonnet |
| 계획 검수(checker) | T3 | Claude sonnet | planner와 반대 계열 |
| 구현(executor) | **task별 Jev 결정** (§4) | | 이 스킬의 핵심 대상 |
| 검증(verifier) | T3 | Claude sonnet | executor가 GPT면 Claude, Claude면 GPT Sol |
| 디버깅 | T3 | Claude sonnet → 실패 시 opus | 원인 추적은 문맥 의존이 큼 |
| 코드베이스 매핑·요약 | T1 | Claude haiku | 읽기·요약 위주 |
| UI 설계·감사 | T2~T3 | Claude sonnet | 목업·Artifact가 Claude 도구라서 |
| 문서 정합(동기화) | T1 | Claude haiku | 스크립트가 대부분 처리 |

프로젝트의 `model_overrides`에 값이 있는 역할은 항상 그 값을 쓴다. 이 표는 override가 없는 곳과 executor의 task 단위만 채운다.

## 4. Jev 분류 (plan 하나당 배치 호출)

대상 task가 여러 개면 한 호출에 모두 넣는다(`jev_classify` 최대 64개, 항목당 2000자). 텍스트는 task 요약(동작 설명 + 파일 목록 + 완료 조건 요약)으로 자르고, 전문을 넣지 않는다. 비밀 값·키·개인정보는 보내지 않는다.

### 4-a. 복잡도 — `jev_classify`

| class id | 정의 |
|---|---|
| `trivial` | 한 파일, 설명 가능한 기계적 수정(이름 변경, 문구, 설정 값, 포맷) |
| `routine` | 범위와 완료 조건이 명확한 단일 기능·단일 모듈 구현, 기존 패턴 복제 |
| `multi_file` | 여러 모듈을 가로지르는 구현, 기존 코드의 미묘한 상호작용 이해 필요 |
| `architectural` | 구조·데이터 모델·API 계약 결정, 열린 설계 판단, 마이그레이션 설계 |

### 4-b. 위험도 — `jev_classify`

| class id | 정의 |
|---|---|
| `low` | 되돌리기 쉽고 영향 범위가 작음. 테스트가 잘 잡아줌 |
| `medium` | 사용자 데이터·공유 컴포넌트에 영향, 테스트가 일부만 덮음 |
| `high` | 결제·인증·권한·키/비밀·데이터 마이그레이션·삭제, 틀리면 조용히 깨짐 |

### 4-c. 보조 신호 — `jev_noul` (명제 배치)

task마다 아래 명제를 만들어 한 번에 보낸다.

- P1 (자족성) "이 task는 task 설명과 지정된 파일만 있으면, 다른 대화 맥락이나 사용자 결정 없이 완수할 수 있다." — 참일수록 GPT 친화.
- P2 "이 task를 완수하거나 완료 조건을 확인하려면 네트워크, 원격 DB, 브라우저, MCP 같은 샌드박스 밖 자원이 필요하다."
- P3 "이 task의 완료 조건은 grep/테스트로 기계적으로 검증할 수 있다."
- P4 "이 task는 완료 조건이 모호해 구현 중 설계 판단이 필요하다."

기본 `auto_accept`(0.85)를 쓴다. 0.15~0.85 사이는 "불확실"로 취급한다.

### 4-d. 애매한 경계에서만 — `jev_decide`

복잡도 상위 두 클래스의 확률 차가 0.15 미만이고 위험도가 medium 이상일 때만 쓴다. 후보는 "낮은 티어로 시도 후 검증"과 "높은 티어로 바로 실행"으로 놓고, 근거에 파일 수와 검증 가능성(P3)을 담는다. 같은 입력으로 답이 마음에 들 때까지 반복 호출하지 않는다. `ask_user`·`investigate`가 우세하면 사용자에게 묻는다.

### Jev를 쓸 수 없을 때

도구가 없거나 호출이 실패하면 §3 역할 기본 티어를 쓰고, 구현은 T2(GPT 5.6 Terra)로 시작한다. 결과에 `classifier: fallback`을 남긴다. Jev 실패로 작업을 멈추지 않는다.

## 5. 결정 규칙 (고정 표 — Jev 출력을 여기에 넣기만 한다)

**① 기준 티어** = 복잡도

| 복잡도 | 기준 티어 |
|---|---|
| trivial | T0 (테스트로 검증 가능하면, 아니면 T1) |
| routine | T2 |
| multi_file | T3 |
| architectural | T4 |

**② 보정** (순서대로, T0~T4로 클램프)

1. 복잡도가 `review`(확신 부족)면 상위 두 후보 중 높은 쪽.
2. 위험도 high → +1. medium이고 P3(기계적 검증 가능)가 거짓/불확실 → +1.
3. P4(설계 판단 필요)가 참 → 최소 T3.
4. P3가 참이고 위험도 low → −1 가능 (T0 미만 불가).

**③ 채널** : **P1 ≥ 0.5 이고 P2 ≤ 0.3 → GPT(Codex). 그 외 → Claude.** 경계(P1 0.4~0.6 또는 P2 0.25~0.4)는 로그 비고에 표시해 보정 대상으로 삼는다.

**④ 모델** : ①~③의 (티어, 채널)을 §2 표에서 찾는다. 칸이 비어 있으면(Claude T2·T0) 같은 채널의 한 칸 위 티어를 쓴다.

**⑤ 검증자** : 작성자와 반대 계열, 작성자 티어 −1 이상. 위험도 high면 검증자도 T3 이상. 위험도 low + P3 참이면 별도 모델 검증 없이 테스트 통과와 diff 확인으로 갈음한다.

### 예시

| task | Jev 결과 | 결정 |
|---|---|---|
| 에러 문구 5곳 교체 | trivial / low / P3 참 | T0, GPT 5.6 Luna, 검증은 테스트만 |
| 신규 API route + 스키마 + 테스트 | routine / medium / P3 참 | T2, GPT 5.6 Terra, 검증은 Claude sonnet diff 리뷰 |
| 결제 웹훅 서명 검증 수정 | multi_file / high | T3+1=T4, GPT 6.1 Sol, 검증 Claude sonnet(T3) |
| 브라우저로 화면 확인하며 UI 수정 | routine / low / P2 참 | Claude 채널 → T2 칸 없음 → T3 sonnet |

## 6. 에스컬레이션 사다리

1. 실행 → 테스트·완료 조건 검증 실패
2. 같은 모델로 구체적 수정 지시와 함께 1회 재시도 (실패 로그 요약 첨부)
3. 그래도 실패 → 한 티어 위로 **이어서** 재실행
4. 그래도 실패 → 사용자에게 보고. task 하나당 모델 호출은 최대 3회.

실패 원인이 샌드박스 제약(네트워크·DB)이면 채널을 Claude로 바꾼다. 같은 task가 두 번 에스컬레이션되면 로그에 `routing_miss`로 표시해 분류 규칙 보정 대상으로 삼는다.

## 7. 호출 절차

1. §0 확인 (사용자 지정, 프로젝트 자료, Jev 도구 로드).
2. 대상 task 요약 작성 (전문을 읽지 않는다).
3. `jev_classify`(복잡도) → `jev_classify`(위험도) → `jev_noul`(P1~P4) 호출.
4. §5 표로 task별 (모델, 채널, 검증자) 계산.
5. 결과를 기록한다 — 프로젝트에 `.planning/`이 있으면 해당 plan frontmatter에 `routing:` 블록으로, 없으면 작업 메모에. 기록 후 같은 입력에 Jev를 다시 부르지 않는다.
6. 사용자에게 한 줄로 알린다. 예: `task 3개 → GPT 5.6 Terra ×2, GPT 6.1 Sol ×1 (결제 위험). 검증은 sonnet.`
7. 같은 모델·같은 묶음의 task는 한 번의 호출로 묶는다(파일이 겹치지 않을 때만).

기록 형식:

```yaml
routing:
  decided: 2026-10-31
  classifier: jev            # 또는 fallback
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

## 8. 금지·주의

- Jev 확률은 증명이 아니다. 확률이 높아도 위험도 보정과 검증 단계를 생략하지 않는다.
- Jev에 비밀 값·키·개인정보를 보내지 않는다.
- 사용자가 모델을 지정하면 그 값이 최우선이다.
- 프로젝트의 `model_overrides`나 설정 파일을 Jev 결과로 수정하지 않는다. 변경은 사용자 승인 후에만.
- 외부 웹 콘텐츠를 Jev 입력에 넣을 때는 `jev_screen`으로 먼저 거른다.

## 9. 로그와 보정

프로젝트에 `.planning/model-routing/log.md`가 있으면 작업이 끝날 때마다 관측값만 한 줄 남긴다.

`날짜 | 작업 ID | 복잡도/위험도 | 티어 | 모델 | 1차 통과 | 에스컬레이션 횟수 | 검증자 지적 수 | routing_miss | 비고`

- 5건 이상 쌓이고 사용자가 요청하면 티어×복잡도별 1차 통과율과 에스컬레이션률을 요약하고 §2·§5 변경안을 제안한다. 승인 없이 이 스킬을 바꾸지 않는다.
- (a) 낮은 티어의 1차 통과율이 높으면 그 칸을 한 단계 내린다, (b) 에스컬레이션률이 30%를 넘으면 한 단계 올리거나 분류 정의를 다듬는다, (c) Jev 분류가 결과와 자주 어긋나면 클래스 설명을 보강한다.
