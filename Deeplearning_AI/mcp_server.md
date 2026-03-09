# MCP : Model Context Protocol

## 1. MCP 정의

LLM이 외부 리소스와 연결하는 **표준 통신 규격**

MCP 이전에는 각 리소스마다 연결 코드를 개별 구현해야 했지만, MCP 표준으로 MCP Client 하나로 여러 외부 리소스에 동일한 방식으로 연결한다.

```
Claude (Host)
└── MCP Client
    ├── MCP Server A (Gmail)   →  실제 Gmail
    ├── MCP Server B (내 DB)   →  실제 DB
    └── MCP Server C (GitHub)  →  실제 GitHub
```

| 구성 요소 | 설명 |
|---|---|
| **Host** | LLM 애플리케이션 (예: Claude) |
| **MCP Client** | Host 안에 내장. MCP Server와 통신 담당 |
| **MCP Server** | 외부 리소스별로 존재. 직접 만들거나 가져다 씀 |
| **External Resource** | 실제 DB, API 등. MCP를 모름 |






---

## 2. MCP Server

단순한 외부 리소스 통로가 아니라, **해당 리소스를 어떻게 쓸 수 있는지까지 정의**한 서버.

MCP Server를 통해 LLM이 사용 가능한 것:

| 종류 | 설명 |
|---|---|
| **Tools** | LLM이 호출하는 함수 |
| **Resources** | LLM이 읽는 데이터 |
| **Prompts** | 재사용 가능한 프롬프트 템플릿 |

---

## 3. Transport 방식

MCP Client ↔ MCP Server 간 통신 방식 2가지.

| 방식 | 용도 |
|---|---|
| **stdio** | 로컬 프로세스. 개발·테스트용 |
| **SSE (HTTP)** | 원격 서버. 프로덕션용 |

---

## 4. 실습: PDF 인보이스 데이터 추출 시스템

### 기존 방식 vs MCP Server 방식

| 단계 | 기존 방식 | MCP Server 방식 |
|---|---|---|
| 데이터 저장 | 로컬에 PDF 저장 필요 | 클라우드 외부 DB에 저장 |
| PDF 처리 | pdfreader로 직접 텍스트 추출 | MCP Server Tools 사용 |
| 정보 추출 | extract_invoice_fields 직접 구현 | Tools 호출로 대체 |
| DB 저장 | cursor 쿼리 | cursor 쿼리 (동일) |
| 리포트 생성 | cursor 쿼리 | cursor 쿼리 (동일) |

### 핵심 차이

**특징 1 — 데이터 접근**
기존은 로컬에 데이터가 있어야 하지만, MCP Server는 외부 DB에 곧바로 접근 가능. 데이터 용량이 클 때 유리.

**특징 2 — 구현 방식**
기존은 PDF 파싱·추출 로직을 직접 구현하지만, MCP Server는 서버에 선언된 Tools를 호출하는 것으로 대체.

### MCP Server 방식 흐름

```
클라우드 PDF
    ↓
StdioServerParameters  ← MCP Server 선언
    ↓
Tools 로드
    ↓
PDF 텍스트 추출 + 정보 추출  (extract_invoice_fields)
    ↓
DB 저장  (cursor 쿼리)
    ↓
리포트 생성  (cursor 쿼리)
```

---

## 5. 용어 정리

| 용어 | 풀네임 | 설명 |
|---|---|---|
| **SDK** | Software Development Kit | 특정 플랫폼·서비스를 코드로 다루기 위한 라이브러리 모음 |
| **ADK** | Agent Development Kit | AI Agent 특화 SDK. 멀티에이전트·Tool 연동·메모리 등 포함 |
| **IDE** | Integrated Development Environment | 통합 개발 환경. 코드 에디터·디버거·터미널·Git 연동 등 포함 |
