# Lesson 4.6: 계획 및 실행 시스템 설계 (Designing a Plan-and-Execute System) 📋

이 문서는 단순한 1단계 추론 루프인 ReAct 패턴을 넘어, 복잡한 과제를 해결하기 위해 거시적으로 계획을 수립하고 이를 동적으로 갱신하며 실행하는 **계획 및 실행(Plan-and-Execute)** 에이전트 시스템에 대한 종합 기술 지침서입니다.

---

## 1. ReAct 패턴과의 비교 및 아키텍처적 진화

### 🔄 ReAct(Lesson 4.2) vs Plan-and-Execute(Lesson 4.6)

```mermaid
flowchart TD
    %% 패턴 비교 아키텍처
    subgraph ReAct_Pattern ["ReAct 패턴 (근시안적 실행)"]
        UserQ1["사용자 질문"] --> LLM1["LLM (생각)"]
        LLM1 -->|도구 선택| Tool1["도구 실행"]
        Tool1 -->|결과 관찰| LLM1
        LLM1 -->|최종 답변| UserA1["답변 완료"]
    end

    subgraph Plan_Execute_Pattern ["Plan-and-Execute 패턴 (거시적 제어)"]
        UserQ2["사용자 질문"] --> Planner["1. Planner (계획 수립)<br/>전체 작업 리스트 분할"]
        Planner --> Executor["2. Executor (단계별 실행)<br/>현재 단계 집중 해결 (ReAct)"]
        Executor --> Replanner["3. Replanner (재계획)<br/>결과 분석 후 계획 갱신"]
        Replanner -->|계획 수정| Executor
        Replanner -->|최종 완성| UserA2["답변 완료"]
    end

    style ReAct_Pattern fill:#fff3e0,stroke:#ffb74d,stroke-width:1px
    style Plan_Execute_Pattern fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
```

* **ReAct 패턴의 한계 (Lesson 4.2 복습)**:
  * ReAct 에이전트는 매 단계마다 "생각(Thought)하고 행동(Action)하고 관찰(Observation)"합니다. 
  * 목표가 복잡하고 하위 태스크가 많을 경우, 에이전트가 앞서 수행한 작업을 망각하거나, 동일한 도구를 중복 호출하는 비효율적인 루프(무한 루프)에 빠지기 쉽습니다. 또한 매 연산마다 전체 프롬프트를 재평가하므로 API 비용이 급증합니다.
* **Plan-and-Execute 패턴의 진화**:
  * **목표 세분화**: 사용자의 최종 목표를 이루기 위한 하위 실행 계획 목록(`plan`)을 먼저 명시적으로 설계합니다.
  * **역할 분담과 모델 최적화**: 
    * **Planner/Replanner (대형 모델 사용)**: 정교하고 넓은 시야가 필요한 계획 수립 및 재계획 과정에는 성능이 우수한 모델(예: GPT-4o)을 배치합니다.
    * **Executor (소형 모델 사용)**: 이미 계획된 명확한 1개 단계만 빠르게 도구로 실행하면 되므로 가볍고 저렴한 모델(예: gpt-4o-mini)을 배치해 자원 효율성을 최대화합니다.

---

## 2. 통합 예시를 통한 동적 순환 흐름 분석 및 상세 해설

사용자가 **`"다음 FIFA 월드컵은 어느 나라에서 개최되나요? 그리고 그 나라 수도의 3대 관광 명소는 무엇인가요?"`**라는 고난도 복합 질문을 던졌을 때, 플래너 ➡️ 실행자 ➡️ 재계획자 노드가 순환하며 상태(`State`)를 갱신해 나가는 전 과정을 단계별로 풀어서 설명합니다.

```mermaid
flowchart TD
    %% 실시간 루프 흐름도
    Start(["사용자 질문 유입"]) --> Stage1["[Step 1] Planner 가동"]
    
    Stage1 -->|계획 수립 완료| Loop1["plan = [<br/>1. 월드컵 개최국 확인,<br/>2. 개최국 수도 확인,<br/>3. 각 수도 3대 명소 검색<br/>]"]
    
    Loop1 --> Stage2["[Step 2] Executor 실행"]
    Stage2 -->|1번 작업 클리어| Loop2["plan = [<br/>2. 개최국 수도 확인,<br/>3. 각 수도 3대 명소 검색<br/>]"]
    
    Loop2 --> Stage3["[Step 3] Replanner 가동"]
    Stage3 -->|중간 결과 기반 계획 수정| Loop3["plan = [<br/>1. 오타와, 멕시코시티, 워싱턴 DC의 3대 명소 검색<br/>]"]
    
    Loop3 --> Stage4["[Step 4] Executor 재실행"]
    Stage4 -->|남은 1번 작업 클리어| Loop4["plan = []"]
    
    Loop4 --> Stage5["[Step 5] Replanner 최종 검토"]
    Stage5 -->|답변 완성 판정| End(["최종 답변 반환 및 종료"])

    style Loop1 fill:#e3f2fd,stroke:#1e88e5
    style Loop2 fill:#e3f2fd,stroke:#1e88e5
    style Loop3 fill:#ede7f6,stroke:#5e35b1
    style Loop4 fill:#e3f2fd,stroke:#1e88e5
```

### 📍 [1단계] 플래너(Planner)의 초기 계획 수립
* **에이전트의 판단**: "두 가지 정보(월드컵 개최국 확인, 개최국 수도의 관광 명소 검색)가 복합된 질문이군. 순차적으로 해결하기 위한 초기 로드맵을 작성하겠다."
* **상태의 변화 (State)**:
  * `plan`: `["다음 월드컵 개최국 확인하기", "그 개최국들의 수도 확인하기", "각 수도의 3대 관광 명소 검색하기"]` (총 3단계의 하위 태스크 리스트 생성)
  * `past_steps`: `[]` (아직 실행된 단계가 없으므로 비어 있음)

### 📍 [2단계] 실행자(Executor)의 1차 실행 (첫 번째 태스크 해결)
* **에이전트의 판단**: `plan` 목록의 가장 첫 번째 작업인 `"다음 월드컵 개최국 확인하기"`를 가져와 Tavily Search 도구를 기동합니다.
* **실행 과정**: 웹에서 최신 자료를 검색하여 "2026 FIFA 월드컵은 캐나다, 멕시코, 미국에서 공동 개최됩니다"라는 정보를 수집합니다.
* **상태의 변화 (State)**:
  * `plan`: `["그 개최국들의 수도 확인하기", "각 수도의 3대 관광 명소 검색하기"]` (이미 실행한 첫 단계를 슬라이싱 `plan[1:]` 처리하여 제거)
  * `past_steps`: `[("다음 월드컵 개최국 확인하기", "개최국은 캐나다, 멕시코, 미국입니다.")]` (첫 번째 단계의 결과가 누적됨)

### 📍 [3단계] 재계획자(Replanner)의 1차 검토 및 계획 수정
* **에이전트의 판단**: "개최국이 캐나다, 멕시코, 미국 3개국이라는 중간 정보가 확보되었군. 그렇다면 기존 계획 중 '그 개최국들의 수도 확인하기' 단계는 내 내부 지식(캐나다 = 오타와, 멕시코 = 멕시코시티, 미국 = 워싱턴 DC)으로 즉각 해결할 수 있어. 따라서 남은 계획을 이 3개 수도의 명소를 각각 검색하는 작업으로 구체화해서 다시 덮어써야겠다."
* **상태의 변화 (State)**:
  * `plan`: `["오타와의 3대 관광 명소 검색하기", "멕시코시티의 3대 관광 명소 검색하기", "워싱턴 DC의 3대 관광 명소 검색하기"]` (기존의 추상적인 계획을 구체화된 3개의 태스크로 갱신하여 덮어씀)
  * `past_steps`: 이전 이력 유지

### 📍 [4단계] 실행자(Executor)의 2차 실행 (구체화된 첫 태스크 해결)
* **에이전트의 판단**: 현재 `plan` 리스트의 첫 번째 작업인 `"오타와의 3대 관광 명소 검색하기"`를 가져와 검색 도구를 기동합니다.
* **실행 과정**: 캐나다 오타와의 국회의사당, 리도 운하, 국립미술관 정보를 웹 검색으로 수집합니다.
* **상태의 변화 (State)**:
  * `plan`: `["멕시코시티의 3대 관광 명소 검색하기", "워싱턴 DC의 3대 관광 명소 검색하기"]` (수행 완료한 오타와 태스크 제거)
  * `past_steps`: `[("다음 월드컵 개최국 확인하기", "개최국은 캐나다, 멕시코, 미국입니다."), ("오타와의 3대 명소 검색", "오타와 명소는 국회의사당, 리도 운하... 입니다.")]` (오타와 결과 추가 누적)

### 📍 [5단계] 실행자(Executor)의 3~4차 실행 (남은 수도들 검색)
* **실행 과정**: 루프를 돌며 `"멕시코시티의 3대 관광 명소 검색하기"`와 `"워싱턴 DC의 3대 관광 명소 검색하기"`를 순차적으로 실행하여 각 도시의 명소 정보를 웹 검색으로 획득합니다.
* **상태의 변화 (State)**:
  * `plan`: `[]` (모든 계획 리스트 실행 완료)
  * `past_steps`: `[..., (각 도시별 명소 검색 결과가 전부 누적됨)]`

### 📍 [6단계] 재계획자(Replanner)의 최종 판단 및 답변 작성
* **에이전트의 판단**: "남은 계획이 존재하지 않고(`plan=[]`), 3개 수도의 3대 명소 정보가 `past_steps`에 완벽히 수집 완료되었다. 더 이상 웹 검색이나 계획 수정은 필요 없다. 수집된 결과들을 종합하여 사용자에게 최종적인 아름다운 답변을 작성해 제공하자."
* **상태의 변화 (State)**:
  * `response`: `"다음 FIFA 월드컵은 캐나다, 멕시코, 미국에서 공동 개최됩니다. 각국 수도의 3대 관광 명소는 다음과 같습니다: 1. 오타와(국회의사당, 리도 운하...), 2. 멕시코시티(소칼로 광장...), 3. 워싱턴 DC(내셔널 몰...)"` (최종 완성 텍스트 주입)

### 📍 [7단계] 조건부 에지 라우터 통과 후 최종 완료
* **제어 흐름**: 라우터 (`routing_condition`)가 상태 딕셔너리에서 `response` 속성에 값이 존재함을 확인하고 그래프를 `END`로 분기시킵니다.
* **최종 완료**: 전체 에이전트 루프가 완전히 종료되고 사용자에게 답변이 송출됩니다.

---

## 3. Plan-and-Execute 핵심 코드 세부 지침

### ① 상태 관리 정의 (`State` 딕셔너리)
```python
import operator
from typing import Annotated, Tuple
from typing_extensions import TypedDict

class State(TypedDict):
    input: str                         # 사용자 질문 원본
    plan: list[str]                     # 앞으로 수행해야 할 남은 하위 계획 목록 (리스트)
    past_steps: Annotated[list[tuple], operator.add]  # 실행 완료한 태스크와 결과 이력 누적
    response: str                      # 최종 완성된 답변 (비어있지 않으면 종료)
```
* **🔍 기술 해설**: 
  * `plan`은 실행자가 단계를 끝낼 때마다 앞부분을 잘라내는 슬라이싱(`plan[1:]`)을 거칩니다.
  * `past_steps`에 달린 `Annotated[..., operator.add]` 리듀서는 새로운 실행 노드의 결과 데이터가 들어올 때, 기존 리스트를 덮어쓰지 않고 뒤에 안전하게 차곡차곡 이어 붙이도록(Append) 지시합니다.

---

### ② 계획 수립용 Pydantic 모델 및 LCEL 체인 (`Planner`)
```python
from langchain_core.prompts import ChatPromptTemplate
from pydantic import BaseModel, Field

# 1. 출력 규격 정의
class Plan(BaseModel):
    steps: list[str] = Field(description="목표를 달성하기 위한 단계들의 리스트")

# 2. 프롬프트 작성
planner_prompt = ChatPromptTemplate.from_messages([
    ("system", "당신은 주어진 질문을 해결하기 위한 계획 수립 전문가입니다. 단계를 쪼개어 나열하세요."),
    ("user", "질문: {input}")
])

# 3. 구조화된 출력(Structured Output)이 결합된 체인 빌드
planner = planner_prompt | llm.with_structured_output(Plan)
```
* **🔍 기술 해설**: 
  * `llm.with_structured_output(Plan)`은 LLM의 반환값을 JSON 스키마 기반으로 검증하여 `Plan` 객체 타입으로 강제 변환합니다. 이를 통해 신뢰할 수 없는 원시 텍스트 대신 깔끔한 파이썬 `list` 형태의 하위 작업들을 수집합니다.

---

### ③ 의사결정 Pydantic 모델 및 재계획 체인 (`Replanner`)
```python
from typing import Union

# 최종 응답 완료 규격
class Response(BaseModel):
    response: str

# 계획 수정 또는 추가 규격
class Act(BaseModel):
    action: Union[Response, Plan]

replanner_prompt = ChatPromptTemplate.from_messages([
    ("system", "당신은 완료된 단계와 결과를 바탕으로 계획을 업데이트하는 계획 시스템입니다."),
    ("user", "목표: {input}\n원래 계획: {plan}\n지나온 단계: {past_steps}")
])

replanner = replanner_prompt | llm.with_structured_output(Act)
```
* **🔍 기술 해설**:
  * Pydantic의 `Union[Response, Plan]` 구조는 강력한 동적 라우팅 기준이 됩니다.
  * 모델이 판단하기에 정보가 충분하다면 `Response` 타입의 객체를 반환하고, 정보가 부족해 후속 태스크를 더 해야 한다면 수정된 `Plan` 타입 객체를 반환하도록 구조를 강제합니다.

---

### ④ 조건부 라우터 함수 (`routing_condition`)
```python
def routing_condition(state: State):
    # 최종 답변이 명시되어 있다면 루프를 정지하고 END로 라우팅
    if state.get("response"):
        return "END"
    # 남은 계획 리스트가 비어있지 않다면 실행자 노드로 이동
    elif state.get("plan"):
        return "executor"
    # 계획은 다 썼는데 아직 response가 없다면 재계획 노드로 송출
    else:
        return "replanner"
```

---

## 4. 실습 환경 세팅 및 실행 가이드

1. **필수 패키지 설치**:
   ```bash
   pip install langgraph langchain-openai langchain-community pydantic dotenv
   ```
2. **API 키 설정 (`.env`)**:
   프로젝트 루트 폴더에 `.env` 파일을 생성하고 아래 키를 입력합니다.
   ```env
   OPENAI_API_KEY="your_openai_api_key"
   TAVILY_API_KEY="your_tavily_api_key"  # 웹 검색 실행을 위해 필수
   ```
3. **노트북 위치 안내**:
   * **관련 개념 코드 파일**: [lesson_4_c_human_in_the_loop_wait_for_input.ipynb](file:///Users/shinwookkang/Desktop/AI_Agent/Module%204:%20Building%20Agents%20with%20LangGraph/code/lesson_4_c_human_in_the_loop_wait_for_input.ipynb)
   * **주의**: 현재 로컬 워크스페이스에 있는 `lesson_4_c_...` 파일은 인간 참여형(Human-in-the-loop: Wait for Input) 기능을 담은 파일입니다. 본 과정의 **Plan-and-Execute** 실습 코드는 이 아키텍처를 기반으로 확장하는 다른 예제 파일에서 파생되므로, 실행 시 노드들의 상태 정의(`.with_structured_output()`)와 `Act` Pydantic 결합 부분을 위 상세 코드를 참고하여 빌드하셔야 합니다.

---

## 5. 핵심 요약 정리 (Cheat Sheet)

| 핵심 항목 | 핵심 기술 내용 | 비고 |
| :--- | :--- | :--- |
| **패턴 핵심** | 플래너(계획 수립) ➡️ 실행자(도구 사용) ➡️ 리플래너(동적 계획 수정) 순환 루프 | ReAct의 무한 루프 한계 극복 |
| **모델 최적화** | 계획 단계(GPT-4o 등 대형 모델) / 실행 단계(gpt-4o-mini 등 소형 모델) 이중 구성 | 비용 및 지연 속도 대폭 개선 |
| **상태 누적** | `Annotated[list, operator.add]`를 적용해 과거 단계 결과의 자동 Append 수행 | 상태 유실 방지 |
| **제어 분기** | `Union[Response, Plan]` 아웃풋 규격으로 답변 완성 시 즉시 탈출 조건 만족 | 라우팅 기준 명확화 |
