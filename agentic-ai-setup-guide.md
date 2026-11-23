---
title: "agentic-ai-public 완전 가이드 - Windows에서 AI Research Agent 실행하기"
date: 2026-11-23 15:00:00 +0900
categories: [개발, AI]
tags: [agentic-ai, docker, fastapi, openai, python]
---

> DeepLearning.AI의 "Agentic Workflow" 강의용 AI 연구 리포트 자동 생성 프로젝트를 Windows 환경에서 실행하는 방법을 정리한 문서입니다.

## 목차

- [프로젝트 소개](#프로젝트-소개)
- [전체 작동 흐름](#전체-작동-흐름)
- [프로젝트 구조 및 파일 설명](#프로젝트-구조-및-파일-설명)
- [실행 방법 (Windows 기준)](#실행-방법-windows-기준)
  - [Step 0: 사전 준비물](#step-0-사전-준비물)
  - [Step 1: 소스 코드 다운로드](#step-1-소스-코드-다운로드)
  - [Step 2: .env 파일 생성](#step-2-env-파일-생성)
  - [Step 3: Docker 이미지 빌드](#step-3-docker-이미지-빌드)
  - [Step 4: 컨테이너 실행](#step-4-컨테이너-실행)
  - [Step 5: 브라우저에서 사용](#step-5-브라우저에서-사용)
  - [Step 6: 종료](#step-6-종료)
- [Windows에서 발생하는 오류와 해결법](#windows에서-발생하는-오류와-해결법)
- [문제 해결 가이드](#문제-해결-가이드)
- [참고 사항](#참고-사항)

---

## 프로젝트 소개

**저장소**: https://github.com/https-deeplearning-ai/agentic-ai-public

사용자가 연구 주제를 입력하면, **여러 AI 에이전트가 팀처럼 협업**하여 학술 수준의 리포트를 자동으로 작성하는 서비스입니다. FastAPI(웹 서버) + PostgreSQL(데이터베이스) 기반이며, Docker로 한번에 실행합니다.

---

## 전체 작동 흐름

```
사용자: "양자 컴퓨팅의 최신 동향을 조사해줘"
         │
         ▼
   ┌─────────────┐
   │ Planner Agent│  ← "팀장": 연구 계획을 세운다 (최대 7단계)
   └──────┬──────┘
          │  계획서 (예: 1.웹검색 → 2.논문검색 → 3.초안작성 → 4.편집)
          ▼
   ┌─────────────┐
   │Research Agent│  ← "조사원": Tavily(웹), arXiv(논문), Wikipedia를 검색
   └──────┬──────┘
          │  조사 결과
          ▼
   ┌─────────────┐
   │ Writer Agent │  ← "작가": 조사 결과를 바탕으로 학술 리포트 초안 작성
   └──────┬──────┘
          │  초안
          ▼
   ┌─────────────┐
   │ Editor Agent │  ← "편집자": 초안을 검토하고 다듬음
   └──────┬──────┘
          │
          ▼
     최종 리포트 (Markdown/HTML)
```

---

## 프로젝트 구조 및 파일 설명

```
agentic-ai-public/
├── main.py                  ← 메인 웹 서버
├── requirements.txt         ← 필요한 Python 패키지 목록
├── Dockerfile               ← Docker 컨테이너 설정
├── .env                     ← (직접 만들어야 함) API 키 저장
├── docker/
│   └── entrypoint.sh        ← 컨테이너 시작 스크립트
├── src/
│   ├── planning_agent.py    ← 계획 수립 에이전트
│   ├── agents.py            ← 연구/작성/편집 에이전트들
│   └── research_tools.py    ← 검색 도구들 (Tavily, arXiv, Wikipedia)
├── templates/
│   └── index.html           ← 웹 UI 화면
└── static/                  ← CSS/JS 등 정적 파일
```

### main.py - 메인 웹 서버

모든 것을 연결하는 "컨트롤 타워". FastAPI 기반.

- **데이터베이스 연결**: PostgreSQL에 연결하여 작업(Task) 상태를 저장
- **웹 페이지 제공**: `GET /` → 사용자가 주제를 입력하는 웹 화면
- **리포트 생성 시작**: `POST /generate_report` → 주제를 받아 계획 수립 → 백그라운드에서 순차 실행
- **진행 상황 조회**: `GET /task_progress/{task_id}` → 현재 몇 단계까지 완료했는지
- **최종 결과 조회**: `GET /task_status/{task_id}` → 완성된 리포트 확인

### src/planning_agent.py - 계획 수립 에이전트

"팀장" 역할. 연구 주제를 받아서 실행 계획(최대 7단계)을 만듦.

- OpenAI 모델(기본: `gpt-4o-mini`)에게 프롬프트를 보내서 계획 생성
- 필수 규칙: 1단계=Tavily 웹검색, 2단계=arXiv 논문검색, 마지막=리포트 작성
- `executor_agent_step()`: 각 단계의 키워드에 따라 적절한 에이전트를 호출
  - "research" → Research Agent
  - "draft" / "write" → Writer Agent
  - "revise" / "edit" / "feedback" → Editor Agent

### src/agents.py - 실행 에이전트 3인방

aisuite 라이브러리를 사용하여 OpenAI 모델(gpt-4o-mini)과 통신.

| 에이전트 | 역할 | 특징 |
|----------|------|------|
| **Research Agent** | 외부 자료 검색 | Tavily, arXiv, Wikipedia 도구 사용. `tool_choice="auto"`, 최대 5번 반복 |
| **Writer Agent** | 학술 리포트 초안 작성 | 필수 9개 섹션, 숫자 인용, 최소 2,400단어 목표 |
| **Editor Agent** | 초안 편집 및 다듬기 | 인용(citation)을 보존하면서 학술적 품질로 편집 |

### src/research_tools.py - 검색 도구 3종

AI 에이전트가 실제로 외부 정보를 가져오는 도구들.

| 도구 | 소스 | API 키 필요 | 설명 |
|------|------|-------------|------|
| `tavily_search_tool` | 웹 전체 | 필요 (TAVILY_API_KEY) | 일반 웹 검색, 제목/내용/URL 반환 |
| `arxiv_search_tool` | arXiv 논문 | 불필요 | 논문 검색 + PDF 다운로드 + 텍스트 추출 (최대 6페이지) |
| `wikipedia_search_tool` | Wikipedia | 불필요 | 위키 문서 요약 검색 (기본 5문장) |

### Dockerfile - Docker 컨테이너 설계도

Python 3.11 기반 이미지에 PostgreSQL과 모든 의존 패키지를 설치하는 설계도. 한 컨테이너에 웹 서버 + DB가 모두 포함됨 (개발/학습용 구성).

### docker/entrypoint.sh - 컨테이너 시작 스크립트

컨테이너 가동 시 자동 실행: PostgreSQL 시작 → DB 준비 대기 → 사용자/DB 생성 → FastAPI 서버 시작.

### requirements.txt - Python 패키지 목록

| 패키지 | 용도 |
|--------|------|
| fastapi, uvicorn | 웹 서버 |
| sqlalchemy, psycopg2-binary | 데이터베이스 |
| python-dotenv | 환경변수 로드 |
| jinja2 | HTML 템플릿 |
| openai, aisuite | AI 모델 호출 |
| tavily-python | 웹 검색 API |
| wikipedia | Wikipedia API |
| pdfminer.six, pymupdf | PDF 텍스트 추출 |

---

## 실행 방법 (Windows 기준)

### Step 0: 사전 준비물

#### (A) Docker Desktop 설치

**목적**: 웹 서버 + 데이터베이스를 한번에 묶어서 가상 환경에서 실행하기 위해 필요합니다.

1. https://www.docker.com/products/docker-desktop/ 에서 다운로드 및 설치
2. 설치 중 "Use WSL 2" 옵션 체크
3. 설치 후 컴퓨터 재부팅
4. Docker Desktop 실행 → 작업 표시줄에 고래 아이콘이 "Engine running" 상태가 되면 준비 완료

확인:
```powershell
docker --version
# Docker version 27.x.x 같은 출력이 나오면 성공
```

#### (B) OpenAI API Key 발급

**목적**: 모든 AI 에이전트가 OpenAI GPT 모델을 사용합니다. 이 키가 없으면 AI가 작동하지 않습니다.

1. https://platform.openai.com 에서 회원가입/로그인
2. 좌측 메뉴 → **API Keys** → **"Create new secret key"**
3. 생성된 `sk-proj-...` 키를 복사하여 보관 (창을 닫으면 다시 볼 수 없음)
4. Settings → Billing에서 **$5~10 충전** (리포트 1건당 약 $0.05~0.20 소요)

#### (C) Tavily API Key 발급

**목적**: Research Agent의 웹 검색에 사용됩니다. 이 키가 없으면 웹 검색이 실패합니다.

1. https://tavily.com 에서 회원가입
2. 대시보드에서 `tvly-...` 형태의 API Key 복사
3. 무료 플랜으로 월 1,000회 검색 가능

---

### Step 1: 소스 코드 다운로드

**목적**: GitHub의 프로젝트 코드를 내 컴퓨터로 가져옵니다.

PowerShell을 열고:

```powershell
cd ~/Desktop
git clone https://github.com/https-deeplearning-ai/agentic-ai-public.git
cd agentic-ai-public
```

> Git이 없으면: GitHub 페이지 → 초록색 `<> Code` 버튼 → `Download ZIP` → 압축 해제

확인:
```powershell
dir
# Dockerfile, main.py, requirements.txt, src/ 등이 보여야 함
```

---

### Step 2: .env 파일 생성

**목적**: API 키를 소스 코드와 분리하여 안전하게 관리합니다. `.gitignore`에 의해 GitHub에 업로드되지 않습니다.

프로젝트 폴더 안에서:

```powershell
notepad .env
```

메모장에 아래 내용 입력 후 저장:

```
OPENAI_API_KEY=sk-proj-여기에-실제키
TAVILY_API_KEY=tvly-여기에-실제키
```

> 등호 양쪽에 공백 넣지 않기. 따옴표 넣지 않기.

---

### Step 3: Docker 이미지 빌드

**목적**: Dockerfile을 읽고 "Python + PostgreSQL + 모든 패키지가 설치된 가상 환경"을 만듭니다. 처음 빌드에 5~10분 소요됩니다.

```powershell
docker build -t fastapi-postgres-service .
```

확인:
```powershell
docker images
# fastapi-postgres-service 이미지가 목록에 보이면 성공
```

---

### Step 4: 컨테이너 실행

**목적**: 빌드한 이미지를 실제로 가동합니다. PostgreSQL이 시작되고, 그 위에 FastAPI 웹 서버가 올라갑니다.

```powershell
docker run --rm -it -p 8000:8000 -p 5432:5432 --name fpsvc --env-file .env fastapi-postgres-service
```

| 옵션 | 의미 |
|------|------|
| `--rm` | 종료 시 컨테이너 자동 삭제 |
| `-it` | 로그를 실시간으로 보여주고 Ctrl+C로 종료 가능 |
| `-p 8000:8000` | 내 컴퓨터의 8000번 포트를 컨테이너에 연결 |
| `--name fpsvc` | 컨테이너 이름 지정 |
| `--env-file .env` | API 키를 컨테이너에 전달 |

성공 시 출력:
```
Starting Postgres cluster 17/main...
Postgres is ready
DATABASE_URL=postgresql://app:local@127.0.0.1:5432/appdb
INFO:     Uvicorn running on http://0.0.0.0:8000
```

> 이 터미널은 닫지 마세요!

---

### Step 5: 브라우저에서 사용

브라우저에서 **http://localhost:8000** 에 접속합니다.

1. 연구 주제를 입력 (영어 권장, 예: `Recent advances in large language models`)
2. 제출 버튼 클릭
3. AI 에이전트들이 순차적으로 작업 수행 (1~5분 소요)
4. 완료되면 학술 리포트 표시

추가 페이지:
- **진행 상황**: `http://localhost:8000/task_progress/{task_id}`
- **최종 결과**: `http://localhost:8000/task_status/{task_id}`
- **API 문서**: `http://localhost:8000/docs`

---

### Step 6: 종료

터미널에서 `Ctrl + C`를 누르면 서버가 종료됩니다.

- `--rm` 옵션 덕분에 컨테이너 자동 삭제
- Docker 이미지는 남아 있으므로 다음에는 Step 4부터 바로 실행 가능

---

## Windows에서 발생하는 오류와 해결법

### 오류 1: `env: 'bash\r': No such file or directory`

**원인**: Windows 줄바꿈(CRLF)과 Linux 줄바꿈(LF)의 차이. `entrypoint.sh`에 `\r`이 섞여서 Linux가 `bash\r`라는 존재하지 않는 프로그램을 찾음.

**해결**: PowerShell에서 줄바꿈을 LF로 변환 후 BOM 없이 저장:

```powershell
$path = "docker/entrypoint.sh"
$text = [System.IO.File]::ReadAllText((Resolve-Path $path))
$text = $text -replace "`r", ""
[System.IO.File]::WriteAllBytes((Resolve-Path $path), [System.Text.UTF8Encoding]::new($false).GetBytes($text))
```

> 이후 반드시 `docker build`를 다시 실행해야 합니다.

### 오류 2: `ModuleNotFoundError: No module named 'psycopg'`

**원인**: `requirements.txt`에 `psycopg2-binary`만 있는데, 최신 SQLAlchemy가 `psycopg` (v3)를 찾으려 함.

**해결**: `requirements.txt`에 패키지 추가:

```powershell
Add-Content -Path requirements.txt -Value "psycopg[binary]"
```

> 이후 반드시 `docker build`를 다시 실행해야 합니다.

### 오류 3: `TypeError: unhashable type: 'dict'` (TemplateResponse)

**원인**: 최신 FastAPI/Starlette에서 `TemplateResponse` 호출 방식이 변경됨.

**해결**: `main.py`의 76번 줄을 수정:

```python
# 변경 전
return templates.TemplateResponse("index.html", {"request": request})

# 변경 후
return templates.TemplateResponse(request=request, name="index.html")
```

> 이후 반드시 `docker build`를 다시 실행해야 합니다.

### 오류 4: `container name "/fpsvc" is already in use`

**원인**: 같은 이름의 컨테이너가 이미 존재함.

**해결**:

```powershell
docker rm -f fpsvc
```

### 오류 5: `The model 'o4-mini' has been deprecated`

**원인**: 원본 코드에서 사용하는 OpenAI 모델(`o4-mini`, `gpt-4.1-mini`)이 더 이상 지원되지 않음.

**해결**: 두 파일의 모델명을 `gpt-4o-mini`로 변경:

`src/planning_agent.py` 27번 줄:
```python
# 변경 전
def planner_agent(topic: str, model: str = "openai:o4-mini") -> List[str]:
# 변경 후
def planner_agent(topic: str, model: str = "openai:gpt-4o-mini") -> List[str]:
```

`src/agents.py`에서 `openai:gpt-4.1-mini`를 모두 `openai:gpt-4o-mini`로 변경 (3군데).

> 이후 반드시 `docker build`를 다시 실행해야 합니다.

### 오류 6: `open //./pipe/dockerDesktopLinuxEngine: The system cannot find the file specified`

**원인**: Docker Desktop이 실행되고 있지 않음.

**해결**: 시작 메뉴에서 Docker Desktop을 실행하고, 고래 아이콘이 "Engine running" 상태가 될 때까지 대기 (1~2분).

---

## 문제 해결 가이드

| 증상 | 원인 | 해결 |
|------|------|------|
| `docker: command not found` | Docker 미설치 | Docker Desktop 설치 |
| 빌드 중 `failed to fetch` | 인터넷 연결 문제 | 네트워크 확인 후 재시도 |
| `port is already allocated` | 8000 포트를 다른 프로그램이 사용 중 | `-p 8001:8000`으로 변경 후 `localhost:8001`로 접속 |
| 서버는 뜨는데 리포트 생성 실패 | API 키 오류 | `.env` 파일의 키 확인 |
| `Insufficient quota` | OpenAI 크레딧 부족 | platform.openai.com에서 충전 |
| 브라우저에 빈 페이지 | `templates/index.html` 누락 | 소스 코드 재다운로드 |
| 이전 작업이 사라짐 | 매 시작마다 DB 초기화됨 | 의도된 동작 (학습용) |
| cmd에서 `ls` 안 됨 | cmd는 `ls` 미지원 | `dir` 사용 또는 `powershell` 입력하여 전환 |

---

## 참고 사항

- **API 키 보안**: `.env` 파일은 로컬에만 존재하며, Docker 컨테이너 → OpenAI/Tavily 서버로만 직접 전송됩니다 (HTTPS 암호화). 외부에 공개되지 않습니다. 단, `.env` 파일을 GitHub에 올리지 않도록 주의하세요.
- **비용**: OpenAI API 호출마다 비용 발생 (gpt-4o-mini 사용, 리포트 1건당 약 $0.05~0.20 추정).
- **영어 권장**: 프롬프트와 도구들이 영어 기준으로 설계되어 있어 영어 주제가 가장 잘 작동합니다.
- **네트워크 필요**: arXiv, Tavily, Wikipedia, OpenAI 모두 인터넷 연결이 필요합니다.
- **두 번째 실행부터는** Docker 이미지가 이미 있으므로 Step 4(`docker run`)만 실행하면 됩니다.
