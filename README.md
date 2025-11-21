# AI Agent 사전과제 – ReAct + Reflection + ReasoningBank (라이트 버전)

## 1. 개요 및 구현 목표

이 프로젝트는 OpenAI Python SDK와 ReAct(Reasoning and Action) 프레임워크를 사용하여, 다음 두 가지 모드의 에이전트를 구현한 사전과제 제출용 코드입니다.

1. **Baseline Mode (ReAct + Reflection)**  
   - 단일 에이전트가 ReAct 루프(Thought → Action → Observation)를 따르며,  
   - 실패 또는 불확실한 상황에서 최대 2회까지 Reflection을 수행해 자기 피드백을 반영합니다.

2. **Enhanced Mode (ReasoningBank + ReAct + Reflection)**  
   - ReasoningBank 논문(ReasoningBank: Scaling Agent Self-Evolving with Reasoning Memory)을 라이트 버전으로 구현하여,  
   - 성공/실패 Trajectory에서 재사용 가능한 추론 전략(Reasoning Rules)을 추출해 JSON 스키마로 저장하고,  
   - 이후 유사한 태스크에서 태그 기반으로 규칙을 검색·주입하여 에이전트 추론에 활용합니다.

3. **도구 구현**  
   - `python_exec(path)`: 외부 파이썬 스크립트를 실행하고 stdout에서 숫자를 정규식으로 추출해 반환합니다.  
   - `xlsx_query(path, query)`: 엑셀 파일의 시트를 자동 탐색하고, 쿼리에 따라 행 필터링 및 합계/집계를 수행합니다.  
   - 두 도구 모두 ReAct 에이전트의 Action으로 호출되며, Observation은 Trajectory JSON에 그대로 기록됩니다.

---

## 2. 프로젝트 구조

```text
.
├── agent_baseline.py         # Baseline 에이전트 (ReAct + Reflection)
├── agent_enhanced.py         # Enhanced 에이전트 (ReasoningBank + ReAct + Reflection)
├── reasoning_bank.py         # ReasoningBank 클래스 (규칙 저장/검색 및 메모리 스키마)
├── prompt_templates.py       # ReAct 및 Enhanced 모드 프롬프트 템플릿
├── tools.py                  # python_exec, xlsx_query 도구 구현
├── run_baseline.py           # Baseline 모드 실행 스크립트 (answers_baseline.json 생성)
├── run_enhanced.py           # Enhanced 모드 실행 스크립트 (answers_enhanced.json, bank.json 생성)
├── requirements.txt          # 필요 라이브러리 목록
├── README.md                 # 실행 및 구현 설명
├── test/
│   ├── 사전과제.json
│   ├── f918266a-b3e0-4914-865d-4faa564f1a1ef.py
│   ├── 7cc4acfa-63fd-4acc-a1a1-e8e529e0a97f.xlsx
│   └── 4d0aa727-86b1-406b-9b33-f870dd14a4a5.xlsx
├── memory/
│   └── bank.json             # Enhanced 모드 실행 후 생성되는 ReasoningBank 규칙 모음
└── runs/
    └── {task_id}/
        ├── baseline_{run_id}.json
        └── enhanced_{run_id}.json
```

- `task_id`는 1~3 (사전과제.json에 정의된 세 문제)
- `mode`는 `baseline` 또는 `enhanced`
- `run_id`는 기본 0 (필요시 여러 번 실행 시 구분용)

---

## 3. 환경 설정 및 종속성

### 3.1 파이썬 패키지 설치

```bash
pip install -r requirements.txt
```

`requirements.txt` 예시:

```text
openai>=1.0.0
pandas
openpyxl
```

- `openai`: OpenAI Python SDK (Chat Completions 사용)
- `pandas`, `openpyxl`: 엑셀 파일 로딩 및 집계용

### 3.2 OpenAI API 키 설정 (선택)

실제 OpenAI API를 사용하려면 유효한 API 키와 과금 크레딧이 필요합니다.  
다음 중 한 가지 방식으로 설정할 수 있습니다.

1. **환경 변수 사용 (권장)**

```bash
export OPENAI_API_KEY="YOUR_API_KEY_HERE"
```

2. **커맨드라인 인자로 전달**

```bash
python run_baseline.py --api_key YOUR_API_KEY
python run_enhanced.py --api_key YOUR_API_KEY
```

### 3.3 Mock 모드 (API 비용 없이 테스트)

API 키 없이 로직과 로깅 구조만 확인하고 싶다면, `--mock` 플래그를 사용합니다.

```bash
python run_baseline.py --mock
python run_enhanced.py --mock
```

Mock 모드에서는 OpenAI API를 호출하지 않고, 내부에서 고정된 ReAct 스타일 응답을 생성하여

- 도구 호출
- Trajectory 기록
- (Enhanced의 경우) bank.json 규칙 생성

까지 전체 파이프라인을 비용 없이 확인할 수 있습니다.

---

## 4. 실행 방법

### 4.1 사전과제 입력

`test/사전과제.json`에는 과제에서 제공한 세 개의 문제와 파일명이 다음과 같이 정의되어 있습니다.

```json
[
  {
    "question": "What is the final numeric output from the attached Python code?",
    "file_name": "f918266a-b3e0-4914-865d-4faa564f1a1ef.py"
  },
  {
    "question": "The attached spreadsheet contains the sales of menu items for a regional fast-food chain. Which city had the greater total sales: Wharvton or Algrimand?",
    "file_name": "7cc4acfa-63fd-4acc-a1a1-e8e529e0a97f.xlsx"
  },
  {
    "question": "The attached file lists the locomotives owned by a local railroad museum... What are the odds that today’s Sunset Picnic Trip will use a steam locomotive? Assume that each day’s excursion picks one of its assigned locomotives at random, and express the answer in the form “1 in 4”, “1 in 5”, etc.",
    "file_name": "4d0aa727-86b1-406b-9b33-f870dd14a4a5.xlsx"
  }
]
```

---

### 4.2 Baseline 모드 실행 (ReAct + Reflection only)

```bash
# Mock 모드 (API 호출 없음)
python run_baseline.py --mock

# 실제 OpenAI API 사용
python run_baseline.py   --api_key YOUR_API_KEY   --model gpt-4o-mini
```

실행 결과:

- `answers_baseline.json`  
  → 각 task_id별 최종 답변과 judgment(`answered` / `failed`) 정리
- `runs/{task_id}/baseline_{run_id}.json`  
  → 스텝별 Trajectory (thought, action, observation, retrieved_rules)를 포함한 로그

---

### 4.3 Enhanced 모드 실행 (ReasoningBank + ReAct + Reflection)

```bash
# Mock 모드
python run_enhanced.py --mock

# 실제 OpenAI API 사용
python run_enhanced.py   --api_key YOUR_API_KEY   --model gpt-4o-mini
```

실행 결과:

- `answers_enhanced.json`
- `runs/{task_id}/enhanced_{run_id}.json`
- `memory/bank.json` (ReasoningBank 규칙 저장 파일, Enhanced 모드에서만 생성/업데이트)

---

## 5. 구현 상세

### 5.1 ReAct + Reflection (Baseline)

- `agent_baseline.py`의 `ReActAgent`는 다음을 수행합니다.
  - ReAct 루프: Thought → Action → Observation
  - 도구:
    - `python_exec(path)`: 스크립트 실행, stdout 캡처, 마지막 숫자 정수로 추출
    - `xlsx_query(path, query)`: 엑셀 전체 시트 탐색, 특정 도시의 total sales 합산, 운영 상태별 기관차 개수 집계 등
  - Reflection:
    - Observation에 에러가 있거나, 모델 출력에 불확실 표현(`not sure`, `uncertain`)이 있을 때 Reflection 트리거
    - 최대 2회까지 Reflection을 추가해 Trajectory에 기록

- Trajectory JSON 예시 (`runs/1/baseline_0.json`):

```json
{
  "step": 1,
  "thought": "Thought: I should query the spreadsheet using the question.
Action: xlsx_query("test/7cc4acfa-...xlsx", "Which city had the greater total sales: Wharvton or Algrimand?")",
  "action": {
    "tool": "xlsx_query",
    "input": {
      "path": "test/7cc4acfa-...xlsx",
      "query": "Which city had the greater total sales: Wharvton or Algrimand?"
    }
  },
  "observation": {
    "raw_query": "...",
    "sheets": [...]
  },
  "retrieved_rules": []
}
```

Baseline 모드에서는 `retrieved_rules`가 항상 빈 리스트입니다.

---

### 5.2 ReasoningBank (라이트 버전) 및 Enhanced 모드

#### 5.2.1 ReasoningBank 스키마

`reasoning_bank.py`는 다음 스키마로 규칙을 관리합니다.

```json
{
  "id": "rb_0001",
  "title": "규칙 제목",
  "description": "언제/왜 유용한지에 대한 짧은 설명",
  "content": [
    "체크리스트 또는 적용 순서 (1~3줄)",
    "예: 숫자 비교 문제에서는 두 값을 모두 도구로 계산한 뒤, 차이를 명시적으로 비교한다."
  ],
  "tags": ["xlsx", "sales", "city"],
  "polarity": "success",
  "evidence": ["traj_step#2"],
  "created_at": "2025-11-21T12:00:00Z",
  "use_count": 3
}
```

- `polarity`: 성공/실패 모두 허용 (`success` 또는 `failure`)
- `tags`: 태스크 타입(xlsx, python, sales 등)을 반영
- `evidence`: 이 규칙이 추출된 Trajectory 스텝 정보

#### 5.2.2 규칙 생성 (Reflection 이후)

`agent_enhanced.py`의 `EnhancedAgent`는 Reflection 시점에 Figure 8 스타일의 Memory Item 프롬프트를 사용하여, Trajectory와 Reflection에서 규칙을 추출합니다.

- 성공/실패 여부와 관계없이, 최대 3개의 Memory Item을 생성하도록 유도
- LLM 출력(마크다운)을 파싱해 `title`, `description`, `content`를 추출하고, 현재 태스크에서 추론한 `tags`를 부여
- `ReasoningBank.add_rule()`를 통해 `memory/bank.json`에 규칙이 누적됩니다.

Mock 모드에서는 비용 절감을 위해 규칙 내용을 고정된 템플릿으로 생성하지만, 실제 API 사용 시에는 Trajectory에 특화된 규칙이 생성됩니다.

#### 5.2.3 규칙 검색 및 프롬프트 주입

Enhanced 모드에서 각 스텝은 다음 과정을 따릅니다.

1. `_infer_tags(question, file_path)`로 태스크 태그를 추론  
   (예: xlsx, sales, city, operating_status 등)
2. `ReasoningBank.retrieve_rules(tags, max_rules=2)`로 관련 규칙 최대 2개 검색
3. `prompt_templates.build_react_prompt_enhanced`에서  
   “Here are some past reasoning strategies…” 블록에 규칙 내용을 주입
4. ReAct 루프가 이 규칙을 참고하면서 Thought/Action을 생성

Trajectory JSON의 `retrieved_rules`에는 해당 스텝에서 참고한 규칙 ID 리스트가 기록됩니다.

---

## 6. 제출물 정리

이 프로젝트에서 생성되는 주요 제출물은 다음과 같습니다.

1. **코드 전부 + README.md**
2. **Baseline 모드 결과**
   - `answers_baseline.json`
   - `runs/{task_id}/baseline_{run_id}.json`
3. **Enhanced 모드 결과**
   - `answers_enhanced.json`
   - `runs/{task_id}/enhanced_{run_id}.json`
   - `memory/bank.json`
4. **입력 데이터**
   - `test/사전과제.json`
   - `test/` 폴더 내 첨부 파일들

이 상태로 폴더 전체를 압축하여 제출하면, 로컬 환경에서 재현 가능한 형태로 실행 및 평가가 가능합니다.
