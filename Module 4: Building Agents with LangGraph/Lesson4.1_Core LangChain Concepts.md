# Lesson 4.1: 핵심 LangChain 개념 (Core LangChain Concepts) 🦜

이 레슨에서는 LangGraph 에이전트 시스템의 기초 뼈대가 되는 **LangChain 프레임워크의 핵심 개념**과 **LCEL(LangChain Expression Language)**을 활용한 선언적 체인(Chain) 설계, 그리고 RAG의 기초 구성 요소를 학습하고 실제 작동하는 파이썬 코드를 한 줄씩 상세히 분석합니다.

---

## 1. LangChain 에코시스템 및 모듈식 아키텍처

LangChain은 LLM 기반의 애플리케이션을 빠르고 모듈식으로 조립할 수 있게 돕는 프레임워크입니다. LangGraph는 LangChain에 반드시 의존할 필요는 없지만, LangChain의 빌딩 블록(모델 래퍼, 프롬프트 템플릿 등)을 사용하면 구현 편의성이 대폭 향상됩니다.

```mermaid
graph TD
    subgraph Packages ["LangChain 핵심 패키지"]
        Core["langchain-core <br> (인터페이스 & LCEL 기본 추상화)"]
        Community["langchain-community <br> (서드파티 허브 & 커뮤니티 연동)"]
        Partners["Partner Packages <br> (langchain-openai, langchain-anthropic 등)"]
    end

    subgraph Extensions ["에코시스템 확장 도구"]
        LG["★ LangGraph <br> (다중 액터 그래프 에이전트)"]
        LServe["LangServe <br> (체인/그래프 REST API 배포)"]
        LSmith["LangSmith <br> (디버깅, 모니터링, 평가)"]
    end

    Core --> Community
    Core --> Partners
    Partners --> LG
    LG --> LSmith
    Core --> LServe
    
    style LG fill:#ffebee,stroke:#c62828,stroke-width:2px
```

---

## 2. 모델 구분: 챗 모델 (Chat Models) vs LLM (Legacy)

LangChain은 다양한 AI 모델 제공업체(OpenAI, Anthropic, Meta 등)의 API를 일관된 문법으로 다룰 수 있는 래퍼(Wrappers)를 제공합니다.

| 구분 | 챗 모델 (Chat Models) - 실무 권장 | 레거시 LLM (Legacy LLMs) |
| :--- | :--- | :--- |
| **입력 형태** | `[SystemMessage, HumanMessage, AIMessage]`와 같은 **메시지 객체 리스트** | 단일 완성형 **텍스트 문자열(String)** |
| **출력 형태** | `AIMessage` 객체 (구조화된 정보 및 메타데이터 포함) | 단순 텍스트 문자열(String) |
| **특징** | • 멀티모달(이미지, 오디오) 입력을 지원<br>• 시스템/유저 역할 분리가 용이 | 단순 텍스트 완성 기능만 제공 (본 강좌에서는 사용하지 않음) |

---

## 3. 프롬프트 템플릿 (Prompt Templates)

사용자가 전달한 원시 입력(Raw Inputs)을 모델이 잘 해석할 수 있는 명확한 지침 형태로 치환(Formatting)해 주는 도구입니다.

### ① 문자열 프롬프트 템플릿 (String Prompt Template)
* 단일 문자열 템플릿으로, 플레이스홀더를 단순히 채워 넣습니다.
* **코드 예시:**
  ```python
  from langchain_core.prompts import PromptTemplate
  
  template = PromptTemplate.from_template("Tell me a joke about {topic}")
  formatted = template.format(topic="cats") # "Tell me a joke about cats" 생성
  ```

### ② 챗 프롬프트 템플릿 (Chat Prompt Template)
* 역할(Role)이 할당된 여러 메시지의 시퀀스를 생성합니다. 시스템 프롬프트 조율에 핵심적입니다.
* **코드 예시:**
  ```python
  from langchain_core.prompts import ChatPromptTemplate
  
  chat_template = ChatPromptTemplate.from_messages([
      ("system", "You are a helpful assistant specialized in {specialty}."),
      ("user", "Hello, help me with {question}")
  ])
  ```

---

## 4. LCEL (LangChain Expression Language)

LCEL은 다양한 컴포넌트(프롬프트, 모델, 파서 등)를 파이프 연산자(`|`)를 사용해 직관적인 체인(Chain)으로 엮을 수 있게 해주는 **선언적 언어**입니다.

### 🔄 LCEL 데이터 흐름 및 파이프라인 구성

```mermaid
graph LR
    Input["1. 입력 데이터 <br> (Dictionary 형식)"] -->|전달| Prompt["2. 프롬프트 템플릿 <br> (ChatPromptTemplate)"]
    Prompt -->|포맷팅 완료 메시지| LLM["3. 언어 모델 <br> (ChatOpenAI)"]
    LLM -->|원시 AI 응답 <br> (AIMessage)| Parser["4. 아웃풋 파서 <br> (StringOutputParser)"]
    Parser --> Output["5. 최종 가공 데이터 <br> (Clean String / JSON)"]

    style Prompt fill:#e8f5e9,stroke:#2e7d32
    style LLM fill:#e3f2fd,stroke:#1e88e5
    style Parser fill:#fff3e0,stroke:#ffb74d
```

### 🌟 LCEL의 핵심 이점
* **스트리밍(Streaming) 기본 지원:** 중간 토큰이나 노드 출력을 실시간 스트리밍할 수 있습니다.
* **비동기 및 병렬 연산:** 여러 개의 LLM 호출이나 외부 도구 연동을 비동기(`ainvoke`) 및 병렬로 동시 실행할 수 있습니다.
* **예외 복구(Failover):** API 시간 초과나 장애 시 재시도(Retries) 및 대체 모델 전환(Fallbacks)을 선언적으로 등록할 수 있습니다.
* **LangSmith 추적성:** 체인을 구성하는 각 파이프의 입출력을 자동으로 로깅하고 디버깅할 수 있습니다.

---

## 5. RAG (검색 증강 생성)의 3대 구성 요소

학습 데이터 시점의 한계(Knowledge Cutoff)나 사내 보안 데이터 조회 한계를 극복하기 위해, **검색 모델과 생성 모델을 결합**하는 핵심 RAG 아키텍처의 3대 빌딩 블록입니다.

1. **임베딩 모델 (Embedding Models):** 비정형 문서를 다차원 공간의 수학적 벡터로 변환하여 의미론적(Semantic) 비교를 가능하게 만듭니다.
2. **벡터 저장소 (Vector Stores):** 대량의 벡터 데이터를 실시간으로 인덱싱하고 보관하는 데이터베이스입니다.
3. **리트리버 (Retrievers):** 사용자의 입력 질문 벡터와 가장 일관성 있는(유사한) 상위 K개의 문서를 찾아 LLM 컨텍스트에 밀어 넣어주는 검색 인터페이스입니다.

---

## 🛠️ 실습: LCEL 기반 여행 플래너 (Trip Planner) 구현 코드 분석

여행 목적지(`destination`)와 관심사(`preferences`)를 인풋으로 받아 마크다운 형식으로 맞춤 일정을 생성하는 파이썬 코드 전체입니다.

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StringOutputParser
from IPython.display import display, Markdown

# 1. 챗 모델 초기화
# 비용 효율적이면서도 뛰어난 언어 능력을 지닌 gpt-4o 모델을 지정합니다.
model = ChatOpenAI(model="gpt-4o", temperature=0)

# 2. 챗 프롬프트 템플릿 설계
# 변수 {destination}과 {preferences}를 가질 플레이스홀더 템플릿을 생성합니다.
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are an expert trip planner. Help me plan a trip to the destination, considering my preferences."),
    ("user", "What should I do in {destination}? My preferences: {preferences}")
])

# 3. 아웃풋 파서 초기화
# LLM이 출력한 AIMessage 객체에서 순수 텍스트(문자열)만 정제하여 꺼내는 역할을 합니다.
parser = StringOutputParser()

# 4. LCEL 파이프라인 조립
# 파이프 연산자(|)를 사용해 '입력 ➡️ 프롬프트 치환 ➡️ LLM 쿼리 ➡️ 문자열 정제' 흐름을 한 줄로 선언합니다.
chain = prompt | model | parser

# 5. 실행을 위한 헬퍼 함수 정의
def plan_trip(destination, preferences):
    # 인풋으로 들어갈 딕셔너리를 생성합니다.
    inputs = {
        "destination": destination,
        "preferences": preferences
    }
    # 체인을 호출(invoke)하여 실행 결과를 가져옵니다.
    result = chain.invoke(inputs)
    return result

# 6. 실행 및 마크다운 렌더링 테스트 (파리 편)
paris_plan = plan_trip("Paris", "museums, cafes, historical sites")
display(Markdown(paris_plan))

# 7. 동일 체인을 재사용하여 테스트 (도쿄 편)
tokyo_plan = plan_trip("Tokyo", "technology, culture, nightlife")
display(Markdown(tokyo_plan))
```

### 💡 파이썬 초보자를 위한 코드 팁:
* `chain = prompt | model | parser`: 이 방식은 각각의 함수 호출을 수동으로 중첩하는 `parser(model(prompt(inputs)))` 방식보다 가독성이 훨씬 뛰어나며, 내부적으로 동기/비동기 스트리밍이 병렬로 활성화되도록 LangChain 엔진이 동적 래핑해 줍니다.
* `chain.invoke(inputs)`: 체인 객체의 `invoke` 메서드는 준비된 JSON 딕셔너리를 전체 LCEL 라인에 주입하여 동기 방식으로 연산을 구동하고 최종 가공 데이터를 수집합니다.
