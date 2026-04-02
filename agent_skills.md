# Agent Skills with Anthropic

## 스킬(Skill)이란?

재사용 가능한 워크플로우, 도메인 지식, 특정 기능을 폴더 단위로 패키징한 지침 모음.

동일한 프롬프트를 반복적으로 사용한다면 Skill로 만들 적합한 후보다.

**장점**
- 반복 작업 자동화
- 컨텍스트 효율 개선
- 재사용 및 팀 공유 가능
- 플랫폼 간 호환 가능 (Codex, Gemini CLI 등)
- 모듈형 구조

---

## Skills vs Tools, MCP, Subagents

### 에이전트 핵심 구성 요소
```
User Prompt
    ↓
Main Agent
    ├── Tools        — 단일 기능 수행 도구
    ├── MCP          — 외부 데이터/시스템 연결
    ├── Skills       — 반복 가능한 워크플로우 묶음
    └── Subagents    — 독립적인 작업 수행자
            └── Skills + 제한된 Tools 사용
```

### 언제 무엇을 쓸까

| 구성 요소 | 사용 시점 |
|---|---|
| Skills | 반복적인 절차, 팀 표준 프로세스, 예측 가능한 결과 도출 |
| Subagents | 병렬 처리, 복잡한 작업 분리, 격리된 권한 필요 |
| MCP | 외부 데이터 접근, 내부 시스템 연동, 실시간 데이터 |

---

## 기본 제공 Skills (Pre-Built)

| Skill | 설명 |
|---|---|
| Document Skill | Excel, PowerPoint, Word, PDF |
| skill-creator | 스킬 폴더 및 파일 구조 자동 생성 |

---

## Custom Skills

### YAML 메타데이터
Claude가 언제 이 스킬을 사용할지 판단하는 기준 — 중요.

- `name`: 소문자, 숫자, 하이픈만 가능. 공백 금지. 동사+ing 형태 권장 (예: `generating-practice-questions`)
- `description`: 언제 사용하는지, 무엇을 하는지, 어떤 키워드에 트리거되는지 명시

### 본문 작성 원칙
- 단계별 명확한 지시 (Step-by-Step)
- Edge case 정의
- 단계를 건너뛰는 경우 이유 명시
- 500줄 이하 유지. 길어지면 `scripts/` 또는 `references/`로 분리
- 경로 표기는 항상 `/` 사용

### 자유도 설계
- 높은 자유도: 다양한 스타일, 창의적 출력 허용
- 낮은 자유도: 순서 고정, 엄격한 출력 형식

### 디렉토리 구조
```
skill-name/
├── SKILL.md         (필수)
├── scripts/         (선택 — 실행 코드, 에러 처리 포함)
├── references/      (선택 — 도메인 지식, 참고 문서)
└── assets/          (선택 — 출력 템플릿, 스키마, 예시 파일)
```

### 실행 흐름
```
SKILL.md (정의)
    ↓
scripts/ (실행)
    ↓
assets/ (출력 템플릿)
    ↓
references/ (지식)
    ↓
skill-creator (품질 평가)
    ↓
unit test harness (동작 검증)
```

---

## Skills with Claude API

Skills 동작에는 파일 시스템 + 코드 실행 환경이 필요.
API는 이 두 가지를 자동으로 제공하지 않으므로 별도 연결 필요.

**셋팅 순서**

1. 스킬 폴더를 Files API로 업로드 → `file_id` 획득
2. 메시지 요청 전송
3. Claude가 샌드박스 안에서 `SKILL.md` 읽고 파일 생성
4. 생성된 파일을 Files API로 다운로드

**요청 구조**
```python
# Beta
client.beta.messages.create(
    model=...,
    max_tokens=...,
    betas=["skills", "code-execution-2025-05-22", "files-api-2025-04-14"],
    container={"skills": [skill_id1, skill_id2]},
    tools=[{"type": "code-execution-xxx", "name": "code-execution"}],
    messages=[{"role": "user", "content": ...}]
)

# Production
client.messages.create(
    model=...,
    max_tokens=...,
    container={"skills": [skill_id1, skill_id2]},
    tools=[{"type": "code-execution", "name": "code-execution"}],
    messages=[{"role": "user", "content": ...}]
)
```

**Beta 파라미터 설명**
- `skills` — 스킬 인식 기능 + `container` 파라미터 활성화
- `code-execution-xxx` — 샌드박스 실행 환경
- `files-api-xxx` — 파일 업로드/다운로드 기능

---

## Skills with Claude Code

Claude Code는 파일 시스템과 코드 실행 환경을 자동으로 제공.

**고정 디렉토리**
```
프로젝트/.claude/skills/스킬이름/SKILL.md
```

---

## Skills with Agent SDK

Claude Code와 동일한 폴더 구조. 단, UI 대신 코드로 구성.

**고정 디렉토리**
```
프로젝트/.claude/skills/스킬이름/SKILL.md
```

**구조**
```python
# 서브에이전트
agents = {
    "subagent1": AgentDefinition(
        description=...,
        prompt=...,
        tools=...
    )
}

# 메인에이전트
agent = ClaudeAgent(
    system_prompt=...,
    allowed_tools=...,
    agents=agents
)
```

**Anthropic 빌트인 Tools**

| Tool | 설명 |
|---|---|
| `Bash` | 터미널 명령 실행 |
| `Write` | 파일 쓰기 |
| `Read` | 파일 읽기 |
| `WebSearch` | 웹 검색 |
| `WebFetch` | 웹 fetch |
| `Task` | 서브에이전트 디스패치 |
| `Skill` | 스킬 파일 읽기 |

`setting_sources = ["users", "projects"]` — 스킬 위치를 에이전트에게 알려주는 설정

---

## Claude API vs Claude Code vs Agent SDK 비교

| | Claude API | Claude Code | Agent SDK |
|---|---|---|---|
| 파일 시스템 | 수동 설정 필요 | 자동 제공 | 자동 제공 |
| 코드 실행 환경 | 수동 설정 필요 | 자동 제공 | 자동 제공 |
| 스킬 디렉토리 | 어디든 가능 (Files API로 업로드) | `프로젝트/.claude/skills/` | `프로젝트/.claude/skills/` |
| 구성 방식 | 코드 | UI + 폴더 | 코드 |
