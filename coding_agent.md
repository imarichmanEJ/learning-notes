# Build full-stack Coding Agent

## demo
<div align="left">
  <video src="https://github.com/user-attachments/assets/466eb72b-9001-44a9-9267-48608d3cf6b4" width="300" controls>
  </video>
</div>

## 1. Coding Agent

목적: LLM이 사용자 쿼리에 따라 스스로 코드실행/파일읽기/파일생성 등 도구를 호출하여 작업을 수행하는 에이전트 생성

순서: LLM 선언 → tool schema 정의 → tool def 정의 → execute_tool def 정의 → coding_agent def 정의

**(1) LLM 정의**
- 실습에서는 OpenAI SDK 사용

**(2) tool schema 정의**
- LLM한테 어떤 도구가 있는지 알려주는 json 명세
- LLM 모델(openai, claude, gemini 등등)에 따라 스키마 구성 요소 다르니 주의

**(3) tool def 정의**
- 각 도구 함수 정의
- 실습에선 execute_code(코드 실행), read_file(파일읽기), write_file(파일생성) 도구 정의함

**(4) tool 실행 함수 정의**
- 호출된 도구 실행하는 함수
- try-except로 에러 케이스 처리

**(5) coding agent 함수 정의**
- 사용자 메세지에 대한 llm 응답이 'message'인 경우와 'function_call'인 경우로 나누어 처리
- has_function_call 플래그로 function_call 여부 확인: True이면 멀티턴, False이면 단일 턴
- while steps < max_steps 루프로 tool 실행 결과를 messages에 누적해서 다시 LLM에 전달

---

## 2. 실행 환경

**(1) 실행 환경 중요성**
- 민감 데이터 포함
- 악의적인 코드 삽입 가능성
- 개인 API키 노출 가능성
- 기존 앱과 충돌
- 확장/축소 어려움

**(2) 실행 환경 옵션**

| 환경 | 특징 |
|---|---|
| Local Execution | 시스템 격리 없음, 리소스 제한, 패키지 충돌, 확장성 문제 |
| Docker | 컨테이너 기반 격리 |
| Sandbox | 프로세스 격리, 파일/네트워크/디바이스 접근 차단, 엄격한 리소스 제한 → **가장 권장** |

---

## 3. Pagination

클라이언트가 서버에 데이터를 요청해서 서버가 클라이언트에 응답할 때, 데이터 **전체를 한 번에 주지 않고 나눠서 주는 방식**

**목적**

한번에 많은 양의 데이터를 전달하면:
- 서버에서 클라이언트로 전송 시간 느림
- 메모리 과부하
- 클라이언트가 렌더링 못함

**방법1. Offset-based**

```
offset=0,  limit=10  →  1~10번째
offset=10, limit=10  →  11~20번째
```

- 구현 쉬움
- 단점: 데이터가 중간에 추가/삭제되면 **중복 or 누락** 발생

**방법2. Cursor-based**

```
cursor=null      →  첫 10개 반환 + next_cursor="abc123"
cursor="abc123"  →  다음 10개 반환 + next_cursor="def456"
```

- 구현 복잡
- 데이터 변동에 안전 → **실무에서 더 많이 씀**

Offset은 "몇 번째부터", Cursor는 "어디까지 봤는지 기억"하는 방식

실시간 데이터가 많은 서비스일수록 Cursor가 적합

가장 효율적인 방법은 DB 쿼리 단계에서 `LIMIT/OFFSET`으로 필요한 것만 가져오기

---

## 4. 에이전트 실습

| 실습 | 내용 |
|---|---|
| 실습 1 | 폴더 생성 후 txt 파일 생성하기 |
| 실습 2 | 파일 생성하고 내용 읽어오기 |
| 실습 3 | 시저 사이퍼 함수 만들기 |
| 실습 4 | 랜덤 좌표 그리기 (matplotlib.pyplot) |
| 실습 5 | 웹사이트 만들기 (index.html) |
| 실습 6 | pokemon.csv, unknown.csv 데이터 분석 (자료 설명 + type별 합계) |
| 실습 7 | 하위 폴더 재귀적으로 탐색해 파일명에 'sbx' 찾기 |
| 실습 8 | full-stack agent로 할일목록 앱 생성하기 |

---

## 5. 참고

### (1) ReAct : Reasoning + Action

프롬프트에서 ReAct 구조로 지시할 때:
- `<scratchpad>` → 내부 추론 공간 (사용자한테 안 보임). LLM이 좀 더 자유롭게 추론하도록 함
- tool 호출 → 실제 행동

**ReAct 패턴 구조**

```
Reason  →  <scratchpad>에서 생각
Act     →  tool 호출
Observe →  tool 결과 받기
Reason  →  다시 <scratchpad>에서 생각
Act     →  다음 tool 호출
...
```

---

### (2) 컨텍스트 윈도우 압축 전담 프롬프트 (SYSTEM_PROMPT_COMPRESS_MESSAGES)

**포함 내용**

| 태그 | 설명 |
|---|---|
| `<overall_goal>` | 사용자가 뭘 원하는지 한 줄 요약 |
| `<key_knowledge>` | 기억해야 할 핵심 사실 (빌드 명령어, API 주소 등) |
| `<file_system_state>` | 어떤 파일을 읽었고, 수정했고, 만들었는지 |
| `<recent_actions>` | 최근에 뭘 했고 결과가 어땠는지 |
| `<current_plan>` | 할 일 목록 (완료/진행중/예정) |

- 압축 자체에도 추론이 필요하기 때문에 여기서도 `<scratchpad>` 생성

---

### (3) 글로브 패턴 (Glob Pattern)

파일 경로를 와일드카드로 표현하는 규칙

| 패턴 | 의미 | 예시 |
|---|---|---|
| `*` | 같은 폴더 내 아무 문자열 | `*.py` → 현재 폴더의 모든 .py 파일 |
| `**` | 하위 폴더 전체 재귀 | `**/*.py` → 모든 하위 폴더의 .py 파일 |
| `?` | 문자 딱 1개 | `file?.py` → file1.py, fileA.py |

```
*.py            →  현재 폴더에서 모든 py 파일 검색
**/*.py         →  재귀적으로 모든 py 파일 검색
test/**/*.py    →  test 폴더 내 모든 py 파일 검색
```

---

### (4) 파일 내용에서 패턴 검색 방법

| 방법 | 설명 |
|---|---|
| 정확 매칭 | 대소문자 포함해 정확히 똑같은 문자열만 검색 |
| 정규식 매칭 | Regex로 패턴 기반 검색 |
| 퍼지 매칭 | 유사도 점수(Levenshtein Distance 기반) 계산해 비슷한 문자열 검색. `fuzzy_threshold`로 임계값 지정 |

---

### (5) Full-Stack Agent

**구성 파일**

| 파일 | 역할 |
|---|---|
| `prompt.py` | 시스템 프롬프트 모음 |
| `tool_schema.py` | LLM에 넘길 툴 명세 (json) |
| `tools.py` | 실제 툴 실행 로직 |
| `coding_agent.py` | Agentic loop + 스트리밍 + 메시지 압축 |
| `server.py` | FastAPI + SSE 엔드포인트 |
| `index.html` | 채팅 UI |

---

**prompt.py**

| 프롬프트 | 설명 |
|---|---|
| `SYSTEM_PROMPT_COMPRESS_MESSAGES` | 대화 히스토리가 너무 길어졌을 때, XML 형식으로 압축 요약해서 에이전트의 메모리를 교체 |
| `SYSTEM_PROMPT_GET_NEXT_SPEAKER` | 에이전트 응답이 끝난건지 아닌지 판단해서, 다음 발화자가 user인지 assistant인지 결정 |
| `SYSTEM_PROMPT_WEB_DEV` | Next.js 시니어 개발자 역할로, scratchpad로 먼저 생각하고 툴을 써서 코드를 작성/수정 |
| `SYSTEM_PROMPT_PYTHON_DEV` | SYSTEM_PROMPT_WEB_DEV에서 Next.js 특화 내용 제거하고 Python 환경에 맞게 수정한 버전 |

---

**tool_schema.py**

LLM 모델마다 스키마 형식 다르니 주의

OpenAI Responses API vs Chat Completions API 형식 차이:

```python
# Responses API
{"type": "function", "name": "read_file", "description": "...", "parameters": {...}}

# Chat Completions API → function 키로 한 번 더 감싸야 함
{"type": "function", "function": {"name": "read_file", "description": "...", "parameters": {...}}}
```

---

**tools.py**

| 함수 | 설명 | 입력 | 출력 | 비고 |
|---|---|---|---|---|
| `_paginate_results` | 리스트 결과를 페이지 단위로 잘라서 반환하는 내부 유틸리티 | `results`, `offset`(기본 0), `limit`(기본 16) | `{pagination: {total, offset, limit, has_more}, results: [...]}` | `results` 키 이중 슬라이싱 버그 있음 (`page[start:end]` → `page`로 수정 필요) |
| `secure_path` | 요청 경로가 작업 디렉토리를 벗어나지 않는지 검증 | `requested_path` | 절대경로 문자열 / 벗어나면 `ToolError` | |
| `execute_code` | Python 코드 실행 | `code` | `{results: [...], errors: [...]}` | `exec()` 기반. Python 전용, Bash 불가 |
| `execute_bash` | Bash 명령어 실행 | `code` | `{results: [...], errors: [...]}` | `subprocess` 기반. Python 코드 실행 불가 |
| `list_directory` | 특정 디렉토리 안의 파일/폴더 목록을 페이지네이션으로 반환 | `path`, `ignore`, `offset`, `limit` | `{pagination: {...}, results: [{name, type, size, modified}], path}` | 디렉토리 먼저, 파일 알파벳순 정렬 |
| `read_file` | 파일 내용을 읽되, offset/limit으로 일부만 읽기 가능 | `file_path`, `limit`, `offset` | `{content: "...", size: int}` | UTF-8 디코딩 실패 시 대체 문자로 처리 |
| `write_file` | 지정한 경로에 파일 쓰기, 디렉토리 없으면 자동 생성 | `content`, `file_path` | `{message: "Written N bytes to ...", size: int}` | |
| `replace_in_file` | 파일 내 특정 텍스트를 찾아서 교체 | `file_path`, `old_string`, `new_string`, `expected_replacements`(기본 1) | `{replacements: int, message: "Replaced N occurrences"}` | 실제 횟수가 expected와 다르면 `ToolError`. 의도치 않은 다중 교체 방지용 |
| `search_file_content` | 파일 내용에서 패턴 검색 (정확/정규식/퍼지 매칭 지원) | `pattern`, `include`, `path`, `use_regex`, `fuzzy_threshold`, `offset`, `limit` | `{pagination: {...}, results: [{file, line, content, similarity?}], files_searched: int}` | 퍼지 모드일 때 유사도 내림차순 정렬. `similarity`는 퍼지 모드일 때만 포함 |
| `glob` | 글로브 패턴으로 파일 경로 검색 | `pattern`, `path`, `ignore`, `offset`, `limit` | `{pagination: {...}, results: [{path, relative_path, size, modified}], pattern}` | `ignore=None`이면 `.gitignore` 자동 로드. 수정시간 내림차순 정렬 |

---

**coding_agent.py**

A. Agentic loop 흐름:

```
유저 입력
    ↓
메시지 압축 체크 (토큰 임계값 초과 시)
    ↓
LLM 호출 (스트리밍)
    ├── 텍스트 응답  →  yield  →  종료
    └── tool_call   →  툴 실행  →  결과 messages에 추가  →  다시 LLM 호출

while steps < max_steps 로 무한루프 방지 (기본값 3)
```

B. 스트리밍 + 툴 호출 동시 처리:
- 스트리밍 시 툴 인자가 청크 단위로 쪼개져서 옴
- 텍스트 누적(`full_content`)과 툴 인자 누적(`arguments`)을 동시에 처리
- 스트림 끝난 후 `json.loads()`로 완성된 인자 파싱

C. 메시지 압축:

```
토큰 수 체크
    ↓
임계값 초과 시
    ↓
전체 메시지 앞 70%  →  LLM으로 압축  →  <state_snapshot> XML
나머지 메세지 30%  →  그대로 유지
    ↓
[압축 스냅샷] + [유지 메시지] 로 교체
```

D. edge case: 압축 경계가 `function_call`로 끝나면 `function_call_output`까지 포함시켜야 함

---

**server.py**

- FastAPI로 `/chat`, `/reset`, `/history` 엔드포인트 제공
- `/chat`: SSE(Server-Sent Events)로 `coding_agent` yield 값 실시간 스트리밍
- 대화 상태(`messages`, `usage`)는 서버 메모리에 저장 → 서버 재시작 시 초기화됨

SSE(Server-Sent Events):
- HTTP 연결 1번으로 서버 → 클라이언트 단방향 스트리밍
- WebSocket보다 단순. 채팅 스트리밍 용도로 충분

```
data: {"type": "text_delta", "content": "안"}
data: {"type": "text_delta", "content": "녕"}
data: [DONE]
```

실행:

```bash
uvicorn server:app --reload --port 8000
```

---

**index.html**

SSE 수신 시 라인이 네트워크 버퍼 단위로 잘릴 수 있음 → 미완성 라인은 다음 청크와 이어붙여야 함

파트 타입별 렌더링:

| 타입 | 렌더링 |
|---|---|
| `text_delta` | 채팅 버블에 텍스트 스트리밍 |
| `function_call` | 🛠️ Using {tool명} 접힌 블록 |
| `function_call_output` | ✅ Tool completed 접힌 블록 |
| `role: user` | 유저 버블 |

---

**실습 시 겪은 오류**

1. **tool schema 형식 불일치** → Responses API 스키마를 Chat Completions API에 그대로 쓰면 `400 Bad Request`. `function` 키로 감싸야 함
2. **시스템 프롬프트 미스매치** → `SYSTEM_PROMPT_WEB_DEV`가 Next.js 전제라 존재하지 않는 폴더 탐색. `SYSTEM_PROMPT_PYTHON_DEV`로 교체 필요
