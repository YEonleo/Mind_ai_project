# AI Agent 사전과제 – ReAct + Reflection + ReasoningBank (라이트 버전)

## 1. 개요 및 구현 목표

이 프로젝트는 대규모 언어 모델(LLM) 기반 에이전트 개발 역량을 보여주기 위해, OpenAI Python SDK와 **ReAct(Reasoning and Action) 프레임워크**를 기반으로 두 가지 모드의 에이전트를 구현했습니다.

**구현 목표:**

1.  **Baseline Mode:** ReAct 루프와 **Reflection** 메커니즘을 결합한 에이전트 구현.
2.  **Enhanced Mode:** ReasoningBank 논문 구조를 라이트 버전으로 구현하여, 성공/실패 경험으로부터 **추론 전략(Reasoning Rules)**을 추출하고 재사용하는 에이전트 구현.
3.  **도구 구현:** `python_exec` 및 `xlsx_query` 두 가지 도구를 구현하여 에이전트가 복잡한 문제(코드 실행, 데이터 분석)를 해결할 수 있도록 지원.

---

## 2. 프로젝트 구조

. ├── agent_baseline.py # Baseline 에이전트 (ReAct + Reflection) ├── agent_enhanced.py # Enhanced 에이전트 (ReasoningBank + ReAct + Reflection) ├── reasoning_bank.py # ReasoningBank 클래스 (규칙 저장 및 검색 로직) ├── prompt_templates.py # 에이전트 프롬프트 템플릿 정의 ├── tools.py # python_exec, xlsx_query 도구 구현 ├── run_baseline.py # Baseline 모드 실행 스크립트 ├── run_enhanced.py # Enhanced 모드 실행 스크립트 ├── run.sh # 전체 과제 실행 쉘 스크립트 ├── test/ # 과제 파일 폴더 (사전과제.json 및 첨부 파일 포함) └── memory/ # ReasoningBank 규칙 파일 저장소 (bank.json)


---

## 3. 환경 설정 및 종속성

프로젝트 실행을 위해 Python 환경과 필요한 라이브러리를 설치해야 합니다.

```bash
# 필요한 패키지 설치 (requirements.txt에 정의되어 있다고 가정)
pip install -r requirements.txt
필수 라이브러리: openai, pandas, openpyxl 등이 필요합니다.

4. API 키 설정 (필수)
이 에이전트는 기본적으로 OpenAI API(GPT-4o-mini)를 사용합니다. API 키는 환경 변수로 설정하거나, 스크립트 실행 시 인수로 전달해야 합니다.

방법 1: 환경 변수 설정 (권장)

Bash

export OPENAI_API_KEY="YOUR_API_KEY_HERE"
방법 2: Mock 모드 사용 (API 비용 절감)

API 키 없이 로직 테스트만 진행하려면, 실행 스크립트에 --mock 플래그를 추가합니다. 이 경우, 에이전트는 미리 정의된 가짜(Mock) 응답을 사용하여 ReAct 루프를 진행합니다.

5. 실행 방법
제공된 3가지 과제에 대해 Baseline 및 Enhanced 모드를 순차적으로 실행합니다.

A. 쉘 스크립트를 이용한 전체 실행 (권장)
run.sh 스크립트는 모든 과제에 대해 두 모드를 실행하고 결과를 로그에 기록합니다.

Bash

# 실제 API를 사용하여 실행
./run.sh real 

# Mock 모드로 실행 (API 비용 없음)
./run.sh mock
B. 개별 Python 스크립트 실행
특정 모드만 실행하려면 다음 명령어를 사용합니다.

Bash

# 1. Baseline 모드 실행 예시
python run_baseline.py --api_key YOUR_API_KEY

# 2. Enhanced 모드 실행 예시 (ReasoningBank 적용)
python run_enhanced.py --api_key YOUR_API_KEY
6. 결과 및 로깅 확인
모든 실행 결과는 runs/ 디렉토리에 JSON 파일 형태로 저장됩니다.

로그 경로: runs/{task_id}/{mode}_{run_id}.json

파일 내용: 각 스텝별 thought, action, observation 및 최종 final_answer가 포함된 Trajectory Log.

제출물 확인: run_baseline.py 및 run_enhanced.py를 실행한 후 생성된 JSON 로그 파일을 통해 에이전트의 최종 정답 및 추론 과정을 확인할 수 있습니다.