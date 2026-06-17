# Lesson 2.2: 사이클을 활용한 복잡한 작업 관리 (Using Cycles to Manage Complex Tasks) 🔄

이 레슨에서는 단일 실행 흐름의 한계를 극복하고, 에이전트가 목표를 달성할 때까지 **생각 ➡️ 행동 ➡️ 관찰**의 흐름을 반복할 수 있도록 하는 **사이클(Cycles, 루프)**을 구현합니다. 파이썬 코드를 한 줄 한 줄 뜯어보며 에이전트가 어떻게 스스로 상태를 관리하고 복잡한 다단계 문제를 완수하는지 학습합니다.

---

## 1. 사이클(Cycles)의 중요성 및 장점

ReAct 사이클은 단순 일회성 API 호출을 넘어, 최종 답변(`Answer`)을 도출할 수 있을 때까지 연쇄적으로 도구를 실행하는 자동화 제어 구조를 의미합니다.

* **다단계 쿼리(Multi-step queries) 해결:** 복잡한 정보 취합이 필요한 경우, 선행 행동의 결과를 토대로 후행 행동을 동적으로 결정합니다.
* **작업의 세분화:** 큰 문제를 마주했을 때 스스로 더 작은 단위 작업들로 쪼개어 하나씩 순차적으로 검증 및 수행합니다.

---

## 2. 루프(Loop)를 지원하는 다중 턴 `query` 함수 구현

에이전트가 여러 번 생각하고 외부 함수를 작동시킬 수 있도록 `while` 루프를 적용하여 개선한 파이썬 코드입니다.

```python
import re

# Action 문구를 탐색하고 도구명과 인자를 그룹으로 분리하는 정규표현식
action_re = re.compile(r'^Action: (\w+): (.*)$')

def query(question, max_turns=5):
    """
    사용자의 질문을 입력받아 에이전트가 최대 max_turns 횟수만큼 ReAct 루프를 자율 실행하도록 합니다.
    - question: 사용자의 최초 질문
    - max_turns: 무한 루프 방지를 위한 최대 반복 횟수 제한 (기본값: 5)
    """
    i = 0
    # 1. Lesson 2.1에서 정의했던 Agent 인스턴스를 생성합니다 (시스템 프롬프트 주입).
    bot = Agent(prompt)
    
    # 2. 에이전트가 다음 번에 입력받을 프롬프트(이력)를 저장합니다. 초기값은 사용자의 질문입니다.
    next_prompt = question
    
    # 3. 최대 실행 횟수(max_turns)에 도달할 때까지 while 루프를 돕니다.
    while i < max_turns:
        i += 1
        
        # 4. 에이전트를 호출하여 LLM의 생각(Thought)과 행동(Action)을 출력으로 얻습니다.
        result = bot(next_prompt)
        print(result)
        
        # 5. LLM의 답변을 줄바꿈 단위로 쪼개어 "Action: 도구명: 인자" 구문이 있는지 검색합니다.
        actions = [
            action_re.match(a) 
            for a in result.split('\n') 
            if action_re.match(a)
        ]
        
        # 6. 실행해야 할 액션이 검출되었다면 실행 단계로 들어갑니다.
        if actions:
            # 캡처 그룹으로부터 도구 이름(action)과 입력값(action_input)을 추출합니다.
            action, action_input = actions[0].groups()
            
            # 사전에 정의하지 않은 미확인 도구인 경우 예외(Error)를 발생시킵니다.
            if action not in known_actions:
                raise Exception(f"Unknown action: {action}: {action_input}")
                
            print(f" -- running {action} {action_input}")
            
            # 7. 실제 파이썬 함수(도구)를 동적으로 구동하여 관찰 결과(observation)를 얻습니다.
            observation = known_actions[action](action_input)
            print("Observation:", observation)
            
            # ⭐ [핵심 피드백 메커니즘]
            # 도구의 실행 결과인 관찰(Observation)을 문자열 포맷으로 만들어,
            # 다음 루프(Next Turn)에서 LLM에게 제공할 입력 프롬프트로 재지정합니다.
            next_prompt = f"Observation: {observation}"
            
        else:
            # 8. 더 이상 수행할 'Action'이 없고 최종 'Answer'가 도출되었을 경우 루프를 즉시 탈출(Exit)합니다.
            return
```

### 💡 파이썬 초보자를 위한 팁:
* `while i < max_turns:`: 변수 `i`가 지정된 최대 수치를 넘지 않는 한 내부 블록의 코드를 끊임없이 반복해서 수행합니다.
* `next_prompt = f"Observation: {observation}"`: 에이전트에게 단순히 다음 질문을 던지는 것이 아니라, **"네가 방금 취했던 행동의 결과는 이것(Observation)이다. 다음 행동은 무엇인가?"**라는 새로운 컨텍스트 정보를 히스토리에 이어서 누적하는 원리입니다.

---

## 3. 테스트 검증 및 동작 원리 추적

### 🧪 테스트 1: 2단계 추론이 필요한 쿼리
> **질문:** "What is the price of 2 bananas?" (바나나 2개의 가격은 얼마인가요?)

**실행 로그 (Console Output):**
```text
Thought: To find the price of 2 bananas, I need to get the price of a single banana first.
Action: get_fruit_price: banana
PAUSE
 -- running get_fruit_price banana
Observation: The price of a banana is $1.2

Action: calculate_total_price: banana: 2
PAUSE
 -- running calculate_total_price banana: 2
Observation: The total price is $2.40

Answer: The price of 2 bananas is $2.40.
```
* **동작 원리 분석:**
  1. **Turn 1 (첫 번째 루프):** 에이전트가 `get_fruit_price: banana` 액션을 호출하고 `PAUSE`. 외부 파이썬 코드가 실행되어 가격 `$1.2`를 획득 후 `Observation`으로 주입.
  2. **Turn 2 (두 번째 루프):** 에이전트가 앞선 `Observation`을 보고 바나나의 단가가 `$1.2`임을 인지, 최종 계산을 위해 `calculate_total_price: banana: 2` 액션을 생성. 파이썬 코드가 총액 `$2.40` 산출.
  3. **Turn 3 (세 번째 루프):** 최종 데이터를 확보했으므로 더 이상 `Action`을 유발하지 않고 `Answer: ...` 결과 문자열을 내놓으며 자율적으로 종료.

---

### 🧪 테스트 2: 다중 루프가 요구되는 고난도 쿼리
> **질문:** "If i bought 10 apples, 10 bananas, and 2 oranges, how much would it cost?" (사과 10개, 바나나 10개, 오렌지 2개를 샀을 때 총 비용은?)

**실행 로그 (Console Output):**
```text
Thought: To find the total cost, I need to get the price of each fruit and then calculate the total price based on the quantities provided.
Action: get_fruit_price: apple
PAUSE
 -- running get_fruit_price apple
Observation: The price of a apple is $1.5

Action: get_fruit_price: banana
PAUSE
 -- running get_fruit_price banana
Observation: The price of a banana is $1.2

Action: get_fruit_price: orange
PAUSE
 -- running get_fruit_price orange
Observation: The price of a orange is $1.3

Action: calculate_total_price: apple: 10, banana: 10, orange: 2
PAUSE
 -- running calculate_total_price apple: 10, banana: 10, orange: 2
Observation: The total price is $29.60

Answer: The total cost for 10 apples, 10 bananas, and 2 oranges is $29.60.
```
* **결과:** 에이전트가 스스로 **총 4회의 사이클**을 조율하며 사과 가격 ➡️ 바나나 가격 ➡️ 오렌지 가격을 개별 조회하고 마지막에 모든 수량을 연합하여 총 가격을 계산하는 프로세스를 완전 자율로 수행했습니다.

---

## 🛠️ 실무 에이전트 설계를 위한 핵심 고려사항

1. **지능적인 모델 선택:**
   * 복잡한 다단계 ReAct 루프를 다룰 때 **GPT-4o-mini** 모델은 간혹 중간 과정을 생략하거나 환각을 일으켜 단계가 끊어질 수 있습니다.
   * 복잡도가 높은 논리적 연쇄 추론에는 성능이 검증된 **GPT-4o** 모델로 전환하여 신뢰성을 확보하는 설계가 좋습니다.
2. **온도(Temperature) 제어:**
   * 자율적이고 구조화된 도구 활용 에이전트를 구축할 때는 무작위적이고 창의적인 출력을 지양해야 하므로 온도를 **`0`**으로 고정하는 것이 베스트 프랙티스입니다.
3. **하이브리드 비용 최적화:**
   * 대형 멀티 에이전트 시스템을 기획할 때는 모든 모듈에 고비용의 최고사양 LLM을 사용하기보다, 단순 태스크(단순 정보 가공 등)에는 가벼운 모델을, 메인 오케스트레이터나 복잡한 추론 장치에는 고사양 모델을 믹스매치하여 구성하는 것이 좋습니다.

---

## 🎓 Module 2 종합 요약 (Module 2 Summary)

모듈 2에서는 파이썬과 OpenAI API만으로 외부 환경과 동적으로 상호작용하는 에이전트를 빌드해 보았습니다.

* **ReAct의 코드 구현체 이해:** `Agent` 클래스를 정의하고 대화 히스토리(`messages`)를 API 호출 시 누적해 넘겨줌으로써 문맥 유지가 가능하도록 만들었습니다.
* **프롬프트 디자인 및 도구 매핑:** `Thought`, `Action`, `PAUSE`, `Observation` 규칙을 Few-shot 기법으로 학습시켜, 에이전트가 규격화된 도구 실행 요구 사항을 출력하도록 유도했습니다.
* **사이클의 제어:** `while` 루프와 최대 실행 제한(`max_turns`) 장치를 통해 에이전트가 실패한 부분은 재시도하고, 여러 단계를 끊김 없이 엮어 최종 답변을 낼 수 있도록 하는 피드백 기반 **자율 에이전트 시스템**을 완성했습니다.
