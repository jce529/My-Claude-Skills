# My Claude Skills

개인 개발 환경에서 쓰는 Claude Code · Codex CLI 전역 스킬 모음입니다. 여러 디바이스에서 같은 스킬을 쓰려고 정리해 둡니다.

## 스킬 목록

| 스킬 | 한 줄 설명 | 문서 |
|---|---|---|
| `jev-model-routing` | 에이전트·plan·task를 어떤 모델(Claude / GPT)에 맡길지 Jev 분류로 정한다 | [USAGE.md](skills/jev-model-routing/USAGE.md) |

## 설치 방법 (공통)

각 스킬 폴더의 `SKILL.md`를 아래 위치에 복사합니다.

- Claude Code: `~/.claude/skills/<스킬 이름>/SKILL.md`
- Codex CLI: `~/.codex/skills/<스킬 이름>/SKILL.md`

예시 (`jev-model-routing`):

```bash
mkdir -p ~/.claude/skills/jev-model-routing
cp skills/jev-model-routing/SKILL.md ~/.claude/skills/jev-model-routing/SKILL.md
```

## 폴더 구조

```text
skills/
└── <스킬 이름>/
    ├── SKILL.md   # 스킬 본문 (에이전트가 읽는 파일)
    └── USAGE.md   # 사람을 위한 사용 설명서
```

`SKILL.md`는 프런트매터(`name`, `description`)로 시작해야 Claude Code와 Codex가 스킬로 인식합니다.
