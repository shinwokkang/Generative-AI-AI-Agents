# Lesson 2.1: 시작하기 - 에이전트 클래스 생성 및 프롬프트 작성 (Creating the Agent Class & Writing Prompts)

이 레슨에서는 파이썬과 OpenAI SDK(GPT-4o-mini)를 활용하여 **ReAct(Reasoning + Acting) 에이전트**를 밑바닥부터 직접 구현해 봅니다. 
파이썬을 전혀 모르는 초보자도 쉽게 코드를 이해하고 구현할 수 있도록 한 줄 한 줄 주석을 달아 상세히 설명합니다.

---

## 📌 초기 구현 로드맵 및 한계점

이번 레슨에서 구현할 ReAct 에이전트는 **루프(반복 사이클)가 없는 가장 단순한 선형 형태**입니다.

```
[ 단일 선형 실행 흐름 ]
사용자 요청(Query) ➡️ 생각(Thought) ➡️ 행동(Action) ➡️ 일시정지(PAUSE) ➡️ 관찰(Observation) ➡️ 종료 (루프 없음)
```

> **⚠️ 초기 구현의 한계점:**
> 아직 생각과 행동을 여러 번 반복하는 **루프(Cycles)**를 구현하지 않았기 때문에, 단 한 번의 생각과 행동만 필요한 단순 작업은 수행하지만 **다단계 추론(Multi-step reasoning)**이 필요한 복잡한 작업은 해결하지 못하고 중간에 멈추게 됩니다.

---

## 1. 환경 설정 및 라이브러리 가져오기 (Imports)

에이전트 구현에 필요한 파이썬 라이브러리를 설치하고 불러오는 과정입니다.

```python
from openai import OpenAI
import re  # 정규표현식(Regular Expression) 처리를 위한 라이브러리
from dotenv import load_dotenv  # .env 파일에서 환경변수를 로드하는 라이브러리

# .env 파일에서 OpenAI API Key를 자동으로 읽어와 프로세스 환경변수에 주입합니다.
# 로컬 작업 디렉토리에 OPENAI_API_KEY="your-api-key" 가 들어있는 .env 파일이 존재해야 합니다.
_ = load_dotenv()

# OpenAI 클라이언트 객체를 생성합니다. (API 호출의 주체가 됨)
client = OpenAI()
```

### 💡 파이썬 초보자를 위한 팁:
* `from ... import ...`: 특정 라이브러리에서 필요한 클래스나 함수만 골라 가져옵니다.
* `import re`: 정규표현식 엔진을 가져와 텍스트 매칭에 활용합니다.
* `load_dotenv()`: API 키와 같은 민감한 설정 정보를 소스 코드에 하드코딩하지 않고 외부 `.env` 파일로부터 안전하게 로드할 때 사용합니다.

---

## 2. 에이전트 클래스(Agent Class) 설계 및 구현

에이전트의 대화 히스토리 관리와 LLM(OpenAI) 호출을 담당하는 핵심 클래스입니다.

```python
class Agent:
    """
    OpenAI API를 사용하여 대화 히스토리를 관리하고 LLM의 응답을 받아오는 에이전트 클래스
    """

    def __init__(self, system=""):
        """
        [생성자 메서드] 에이전트가 생성될 때 최초 1회 실행됩니다.
        - system: 에이전트의 역할과 행동 강령을 정의하는 시스템 프롬프트(지침)입니다.
        """
        self.system = system  # 시스템 프롬프트 문자열을 저장
        self.messages = []    # 대화 히스토리(메시지 리스트)를 빈 리스트로 초기화
        
        # 시스템 지침(system prompt)이 인자로 주어졌을 경우, 메시지 히스토리 맨 처음에 추가합니다.
        if self.system:
            self.messages.append({"role": "system", "content": system})

    def __call__(self, message):
        """
        [호출 메서드] 에이전트 인스턴스를 함수처럼 직접 호출할 때 실행됩니다.
        예: agent("질문 내용")
        """
        # 1. 사용자의 질문을 'user' 역할로 히스토리에 추가합니다.
        self.messages.append({"role": "user", "content": message})
        
        # 2. execute() 메서드를 실행하여 LLM(OpenAI)에게 전체 대화 이력을 전달하고 응답을 얻습니다.
        result = self.execute()
        
        # 3. LLM이 답변한 결과를 'assistant' 역할로 히스토리에 추가합니다.
        self.messages.append({"role": "assistant", "content": result})
        
        # 4. 최종적으로 LLM이 텍스트 형태로 생성한 답변(결과)을 반환합니다.
        return result

    def execute(self):
        """
        [실행 메서드] 실제로 OpenAI API를 호출하여 LLM의 응답 내용을 받아옵니다.
        """
        # OpenAI의 ChatCompletion API를 호출합니다.
        completion = client.chat.completions.create(
            model="gpt-4o-mini",  # 비용 효율적이고 고성능인 GPT-4o-mini 모델 사용
            temperature=0,        # 출력을 일관되고 고정된 형태로 제어하기 위해 온도를 0으로 설정
            messages=self.messages  # 지금까지 누적된 전체 메시지 히스토리 전달
        )
        
        # 생성된 응답 텍스트를 반환합니다.
        return completion.choices[0].message.content
```

### 💡 파이썬 초보자를 위한 팁:
* `__init__`: 파이썬 클래스의 인스턴스가 생성될 때 속성값(변수)을 초기화하는 약속된 함수입니다.
* `__call__`: 클래스의 객체를 `agent = Agent()`, `agent("질문")` 형태로 함수처럼 직관적으로 호출 가능하도록 만들어 줍니다.
* `self`: 클래스 내부에서 자기 자신의 속성(변수)이나 다른 함수를 호출할 때 사용하는 지시어입니다.
* `messages`: OpenAI Chat API는 이전 대화를 기억하지 못하는 Stateless 구조이므로, 대화가 끊기지 않도록 `[{"role": "user", "content": "..."}, {"role": "assistant", "content": "..."}]` 형태로 이력을 계속 누적해 넘겨주어야 합니다.

---

## 3. ReAct 프롬프트(Prompt) 작성

에이전트가 논리적으로 사고하고(Thought), 올바른 행동(Action)을 파악하고, 결과를 관찰(Observation)하도록 흐름을 제어하는 프롬프트 지침입니다.

```python
# 에이전트의 페르소나 및 행동 규칙을 정의하는 프롬프트 문자열
prompt = """
You run in a loop of Thought, Action, PAUSE, Observation.
At the end of the loop you output an Answer.
Use Thought to describe your thoughts about the question you have been asked.
Use Action to run one of the actions available to you - then return PAUSE.
Observation will be the result of running those actions.

Your available actions are:

calculate_total_price:
e.g. calculate_total_price: apple: 2, banana: 3
Runs a calculation for the total price based on the quantity and prices of the fruits.

get_fruit_price:
e.g. get_fruit_price: apple
returns the price of the fruit when given its name.

Example session:

Question: What is the total price for 2 apples and 3 bananas?
Thought: I should calculate the total price by getting the price of each fruit and summing them up.
Action: get_fruit_price: apple
PAUSE

Observation: The price of an apple is $1.5.

Action: get_fruit_price: banana
PAUSE

Observation: The price of a banana is $1.2.

Action: calculate_total_price: apple: 2, banana: 3
PAUSE

You then output:

Answer: The total price for 2 apples and 3 bananas is $6.6.
""".strip()
```

### 💡 프롬프트 구조 뜯어보기
* **동작 루프 선언:** 에이전트에게 `Thought(생각)` ➡️ `Action(행동)` ➡️ `PAUSE(일시정지)` ➡️ `Observation(관찰)` ➡️ `Answer(최종 답변)` 순서로 작동하라고 명확히 지시합니다.
* **사용 가능한 행동(도구) 명세:** `calculate_total_price`와 `get_fruit_price`라는 행동명과 예상 입력 포맷(예: `apple: 2, banana: 3`)을 미리 상세하게 주입합니다.
* **퓨샷(Few-shot) 예시 제공:** 에이전트가 예시 세션을 참고하여, `Action`을 취한 뒤 반드시 `PAUSE` 문자열을 출력하고 멈춰서 외부 시스템(파이썬 실행 코드)이 데이터를 채워줄 때까지 대기하게 유도합니다.

---

## 4. 에이전트가 사용할 도구(Actions) 함수 구현

에이전트가 필요에 따라 호출할 실제 파이썬 함수와 데이터셋입니다.

```python
# 과일 가격을 저장하고 있는 로컬 데이터베이스 대용 딕셔너리
fruit_prices = {
    "apple": 1.5,
    "banana": 1.2,
    "orange": 1.3,
    "grapes": 2.0
}

# 1. 특정 과일의 단가를 조회하는 함수
def get_fruit_price(fruit):
    """
    fruit (str): 과일 이름
    """
    # 딕셔너리에 해당 과일이 존재하는지 확인
    if fruit in fruit_prices:
        return f"The price of a {fruit} is ${fruit_prices[fruit]}"
    else:
        return f"Sorry, I don't know the price of {fruit}."

# 2. 수량을 기반으로 총 금액을 계산하는 함수
def calculate_total_price(fruits):
    """
    fruits (str): "apple: 2, banana: 3" 형태로 전달되는 문자열
    """
    total = 0.0
    # 콤마(, )를 기준으로 여러 과일 정보를 분리하여 리스트로 만듭니다.
    # 결과: ['apple: 2', 'banana: 3']
    fruit_list = fruits.split(", ")
    
    for item in fruit_list:
        # 콜론(: )을 기준으로 과일 이름과 수량을 분리합니다.
        # 결과: fruit = 'apple', quantity = '2'
        fruit, quantity = item.split(": ")
        quantity = int(quantity)  # 문자열 숫자를 정수형(Int)으로 변환
        
        if fruit in fruit_prices:
            # 단가 * 수량을 총액에 가산
            total += fruit_prices[fruit] * quantity
        else:
            return f"Sorry, I don't have the price of {fruit}."
            
    # 소수점 둘째 자리까지 포맷팅하여 총액 문자열을 반환합니다.
    return f"The total price is ${total:.2f}"

# 3. 에이전트가 프롬프트에서 도구명으로 매핑할 수 있도록 딕셔너리로 묶어둡니다.
known_actions = {
    "get_fruit_price": get_fruit_price,
    "calculate_total_price": calculate_total_price
}
```

### 💡 팁:
* `fruits.split(", ")`: 문자열을 지정한 구분자(여기선 `, `)로 쪼개어 파이썬 리스트 구조로 반환합니다.
* `int(quantity)`: 문자 형식인 `"2"`는 계산할 수 없기 때문에 수학 연산이 가능한 정수형 데이터 타입으로 형변환 해주는 필수 작업입니다.
* `f"{total:.2f}"`: 파이썬의 F-string 포맷팅으로 소수점 아래 두 번째 자리(`2f`)까지만 실수(Float)를 출력하라는 서식 지정법입니다.

---

## 5. 단일 턴 쿼리 처리 헬퍼 함수 구현

에이전트에게 쿼리를 전달하고, 에이전트의 답변에서 `Action` 문구를 정규 표현식으로 추출해 파이썬 함수를 단 한 번 대행 실행해 주는 헬퍼 함수입니다.

```python
# 정규표현식 컴파일: "Action: 도구명: 입력값" 형태를 매칭하여 캡처 그룹으로 추출합니다.
# ^Action: : 줄 시작이 "Action: "으로 시작해야 함
# (\w+) : 알파벳/숫자/언더바가 연속되는 도구명을 첫 번째 그룹으로 지정
# : (.*)$ : 콜론 뒤의 모든 매칭 텍스트를 두 번째 그룹(인풋값)으로 지정
action_re = re.compile(r'^Action: (\w+): (.*)$')

def query(question):
    # 1. 우리가 작성한 프롬프트 지침을 주입하여 에이전트 객체를 새로 만듭니다.
    bot = Agent(prompt)
    
    # 2. 에이전트에게 질문을 전달하고 LLM의 응답(Thought 및 Action)을 반환받습니다.
    result = bot(question)
    print(result)  # LLM이 내놓은 텍스트 출력
    
    # 3. LLM의 답변을 줄바꿈(\n) 단위로 쪼개어 정규표현식에 매칭되는 'Action' 줄이 있는지 찾습니다.
    actions = [
        action_re.match(a) 
        for a in result.split('\n') 
        if action_re.match(a)
    ]
    
    # 4. 만약 추출된 액션(도구 실행 요구)이 존재한다면 실행합니다.
    if actions:
        # 정규표현식 매칭 그룹에서 도구명(action)과 인자(action_input)를 추출합니다.
        action, action_input = actions[0].groups()
        
        # 정의되지 않은 도구를 실행하려 할 경우 에러를 발생시킵니다.
        if action not in known_actions:
            raise Exception(f"Unknown action: {action}: {action_input}")
            
        print(f" -- running {action} {action_input}")
        
        # 5. 매핑 딕셔너리를 활용해 실제 파이썬 함수를 호출하고 관찰(Observation) 결과를 얻습니다.
        observation = known_actions[action](action_input)
        print("Observation:", observation)
    else:
        # 실행할 액션이 없다면 그대로 종료합니다.
        return
```

---

## 6. 테스트 실행 및 한계 검증

### 🧪 테스트 1: 단일 도구 조회가 필요한 단순 쿼리
> **질문:** "What is the price of a banana?" (바나나 가격은 얼마인가요?)

**출력 결과:**
```
Thought: I need to find out the price of a banana by using the appropriate action.
Action: get_fruit_price: banana
PAUSE
 -- running get_fruit_price banana
Observation: The price of a banana is $1.2
```
* **동작 설명:** 단 한 번의 검색 도구 실행(`get_fruit_price`)만 필요하므로, 에이전트가 올바른 도구를 선택해 멈추고 파이썬 코드가 결과를 가져와 관찰값으로 제공하는 데까지 정상 동작합니다.

#### ⚠️ 환각(Hallucination)의 실질적인 예시 (오렌지 쿼리 시)
일부 경우에 LLM은 실제 도구를 실행하여 결과(`Observation`)를 받기 전에, 성급하게 스스로 결론을 내려 "오렌지 가격은 1.0이다"라고 거짓 정보를 출력하기도 합니다(환각). 하지만 우리가 헬퍼 함수를 통해 직접 도구를 파싱 및 강제 구동하여 `Observation: The price of a orange is $1.3`이라는 정확한 외부 데이터를 가져와 주입함으로써 LLM이 훈련 데이터 외의 최신 현실 데이터를 반영하도록 제어할 수 있습니다.

---

### 🧪 테스트 2: 다단계 추론이 필요한 복잡한 쿼리 (현재 구현의 한계점 봉착)
> **질문:** "What is the price of 2 bananas and 3 oranges?" (바나나 2개와 오렌지 3개의 총 가격은 얼마인가요?)

**출력 결과:**
```
Thought: I need to find the price of each fruit first, then calculate the total price for the given quantities.
Action: get_fruit_price: banana
PAUSE
 -- running get_fruit_price banana
Observation: The price of a banana is $1.2
```

* **동작 설명 및 한계:** 
  1. 에이전트는 먼저 각 과일의 가격을 차례로 구하고 총합을 계산해야 한다는 **논리적 계획**을 수립합니다.
  2. 첫 단추로 바나나 가격을 가져오기 위한 `Action: get_fruit_price: banana`를 출력하고 `PAUSE` 상태로 멈춥니다.
  3. 파이썬 헬퍼 함수가 함수를 실행하여 `Observation: The price of a banana is $1.2`를 획득합니다.
  4. **그러나 여기서 완전히 멈춰버립니다.** 
  5. 오렌지 가격을 조회하는 다음 액션과 최종 가격을 계산하는 액션 단계로 나아가지 못합니다. 왜냐하면 헬퍼 함수에 **에이전트를 다시 호출하고 관찰값을 넘겨주는 루프(Cycles)** 구조가 설계되어 있지 않기 때문입니다.
