---
title: 스킬 작성하기
summary: SKILL.md 형식과 모범 사례
---

스킬은 에이전트가 하트비트 중에 호출할 수 있는 재사용 가능한 지침입니다. 에이전트에게 특정 작업 수행 방법을 가르치는 마크다운 파일입니다.

## 스킬 구조

스킬은 YAML 프론트매터가 포함된 `SKILL.md` 파일을 가진 디렉토리입니다:

```
skills/
└── my-skill/
    ├── SKILL.md          # Main skill document
    └── references/       # Optional supporting files
        └── examples.md
```

## SKILL.md 형식

```markdown
---
name: my-skill
description: >
  Short description of what this skill does and when to use it.
  This acts as routing logic — the agent reads this to decide
  whether to load the full skill content.
---

# My Skill

Detailed instructions for the agent...
```

### 프론트매터 필드

- **name** -- 스킬의 고유 식별자 (kebab-case)
- **description** -- 에이전트에게 이 스킬을 언제 사용해야 하는지 알려주는 라우팅 설명. 마케팅 문구가 아닌 결정 로직으로 작성하세요.

## 런타임에서의 스킬 작동 방식

1. 에이전트가 컨텍스트에서 스킬 메타데이터(이름 + 설명)를 확인합니다
2. 에이전트가 현재 작업에 해당 스킬이 관련 있는지 판단합니다
3. 관련이 있으면 전체 SKILL.md 내용을 로드합니다
4. 에이전트가 스킬의 지침을 따릅니다

이를 통해 기본 프롬프트를 작게 유지합니다 -- 전체 스킬 내용은 필요할 때만 로드됩니다.

## 모범 사례

- **설명을 라우팅 로직으로 작성하세요** -- "사용할 때"와 "사용하지 말아야 할 때" 안내를 포함하세요
- **구체적이고 실행 가능하게** -- 에이전트가 모호함 없이 스킬을 따를 수 있어야 합니다
- **코드 예시를 포함하세요** -- 구체적인 API 호출과 명령 예시가 산문보다 더 신뢰성 있습니다
- **스킬을 집중적으로 유지하세요** -- 관심사당 하나의 스킬; 관련 없는 절차를 결합하지 마세요
- **참조 파일은 절제하여 사용하세요** -- 메인 SKILL.md를 부풀리지 말고 지원 세부 사항은 `references/`에 넣으세요

## 스킬 주입

어댑터는 에이전트 런타임에서 스킬을 발견할 수 있게 하는 역할을 합니다. `claude_local` 어댑터는 심볼릭 링크가 있는 임시 디렉토리와 `--add-dir`을 사용합니다. `codex_local` 어댑터는 전역 스킬 디렉토리를 사용합니다. 자세한 내용은 [Creating an Adapter](/adapters/creating-an-adapter) 가이드를 참조하세요.
