# ROS2 프로그래밍 입문 6차시 정리  
## 인터페이스 프로그래밍(응용 1) — Calculator 패키지

---

# 1. 핵심 내용

## 1.1 패키지 설계
- ROS2의 Topic, Service, Action 프로그래밍을 이용하여 각 노드들이 서로 연동되도록 패키지를 설계
- 프로세스를 목적별로 나누어 노드 단위 프로그램을 작성
- 노드와 노드 간 데이터 통신을 고려하여 전체 시스템을 구성  
  <!-- 이번 차시 핵심은 패키지 내부에서 노드를 역할별로 분리하고 연결하는 것 -->

## 1.2 실습 패키지 설계 목표
- 계산기 개발
- 현재 시간과 변수 `a`, `b`를 받아 연산
- 연산 결과를 누적
- 목표치에 도달하면 결과를 표시  
  <!-- 단순 계산기가 아니라 Topic + Service + Action을 함께 쓰는 통합 예제 -->

## 1.3 노드별 역할

### argument
- `arithmetic_argument` 토픽 이름으로 현재 시간과 변수 `a`, `b`를 publish  
  <!-- 입력 데이터 생성 노드 -->

### calculator
- 토픽이 생성 시점과 변수 `a`, `b`를 `arithmetic_argument` 토픽으로 수신
- 수신한 변수 `a`, `b`와 operator 노드로부터 받은 연산자를 통해 계산 수행
- 연산 결과를 `arithmetic_operator` 서비스 응답값으로 operator 노드에 전송
- checker 노드로부터 액션 목표값을 수신 후, 저장된 변수와 연산자를 활용해 연산한 값을 합산
- 계산이 완료된 결과를 `arithmetic_checker` 액션 피드백으로 checker 노드에 전송
- 합산 결과값이 액션 목표값을 넘기면 최종 연산 합계를 `arithmetic_checker` 액션 결과값으로 전송  
  <!-- 시스템 중심 노드 -->

### operator
- `arithmetic_operator` 서비스 이름으로 calculator 노드에게 연산자 `+`, `-`, `*`, `/`를 요청값으로 전달  
  <!-- 요청 역할 -->

### checker
- 연산값 합계의 한계치를 `arithmetic_checker` 액션 이름으로 목표값으로 전달  
  <!-- 상태 감시 역할 -->

## 1.4 Topic / Service / Action 비교
- Topic: 지속 데이터 흐름, Publisher / Subscriber, `msg`
- Service: 요청 / 응답 구조, Client / Server, `srv`
- Action: 진행 상태 포함, Client / Server, `action`  
  <!-- calculator 예제는 이 3가지를 한 시스템 안에서 결합함 -->

---

# 2. 핵심 실습 — ex_calculator 패키지

## 2.1 의존 패키지 설치
```bash
rosdep install --from-paths src --ignore-src -r -y
```
<!-- 누락된 시스템 의존성 자동 설치 -->

## 2.2 폴더 구조
```text
ex_calculator/
 ├─ arithmetic/
 │   ├─ __init__.py
 │   ├─ argument.py
 │   ├─ operator.py
 ├─ calculator/
 │   ├─ __init__.py
 │   ├─ calculator.py
 │   ├─ main.py
 ├─ checker/
 │   ├─ __init__.py
 │   ├─ checker.py
 │   ├─ main.py
 ├─ launch/
 ├─ param/
 ├─ resource/
 ├─ test/
 ├─ package.xml
 └─ setup.py
```
<!-- 노드를 역할별 폴더로 분리한 구조 -->

## 2.3 인터페이스 패키지 CMakeLists.txt 핵심
```cmake
set(msg_files
  "msg/MyMsg.msg"
  "msg/ArithmeticArgument.msg"
)

set(srv_files
  "srv/MySrv.srv"
  "srv/ArithmeticOperator.srv"
)

set(action_files
  "action/MyAction.action"
  "action/ArithmeticChecker.action"
)
```
<!-- 실제 사용할 msg / srv / action 파일 등록 -->

---

# 3. 인터페이스 정의

## 3.1 ArithmeticArgument.msg
```text
builtin_interfaces/Time stamp
float32 argument_a
float32 argument_b
```
<!-- 시간 + 두 피연산자 전달 -->

## 3.2 ArithmeticOperator.srv
```text
# Constants
int8 PLUS = 1
int8 MINUS = 2
int8 MULTIPLY = 3
int8 DIVISION = 4

# Request
int8 arithmetic_operator

---
# Response
float32 arithmetic_result
```
<!-- 연산자를 숫자 코드로 요청하고 결과를 반환 -->

## 3.3 ArithmeticChecker.action
```text
# Goal
float32 goal_sum

---
# Result
string[] all_formula
float32 total_sum

---
# Feedback
string[] formula
```
<!-- 목표 누적합, 중간 식 목록, 최종 전체 식/합계 관리 -->

---

# 4. 설정 파일

## 4.1 package.xml / setup.py
- `package.xml`: 빌드 / 실행 의존성 선언
- `setup.py`: 패키지 설치 정보 및 `console_scripts` 등록  
  <!-- Python 패키지를 ROS2에서 실행 가능한 형태로 만드는 핵심 설정 -->

## 4.2 파라미터 파일
```yaml
/**:
  ros__parameters:
    qos_depth: 30
    min_random_num: 0
    max_random_num: 9
```
<!-- argument 노드 등의 동작 조건을 외부 설정으로 조절 -->

---

# 5. 코드 설명 포인트

## 5.1 Topic Publisher / Subscriber
- `Argument` 클래스
  - `Node` 상속
  - 노드 이름 `argument`
  - QoS 설정: `RELIABLE`, `KEEP_LAST`, `DEPTH 10`, `VOLATILE`
  - `create_publisher()` 사용
  - 토픽 타입 `ArithmeticArgument`
  - 토픽 이름 `arithmetic_argument`  
  <!-- a, b 값을 지속적으로 publish -->

## 5.2 Calculator 노드
- Topic Subscriber
- Service Server
- Action Server  
  <!-- 세 기능이 모두 한 노드에 들어간 복합 노드 -->

## 5.3 핵심 구조
```text
argument -> Topic -> calculator
operator -> Service -> calculator
checker  -> Action -> calculator
```
<!-- calculator를 중심으로 여러 노드가 연결됨 -->

---

# 6. 빌드 및 실행

## 6.1 빌드
```bash
colcon build --packages-select ros_study_msgs ex_calculator
```
<!-- 인터페이스 패키지와 실행 패키지를 함께 빌드 -->

## 6.2 실행 예시
```bash
ros2 run ex_calculator calculator
ros2 run ex_calculator argument
ros2 run ex_calculator operator
ros2 run ex_calculator checker
```
<!-- 노드를 각각 실행하여 통신 구조를 확인 -->

---

# 7. 핵심 정리

- 패키지는 **기능 묶음**
- 노드는 **역할 단위**
- 인터페이스는 **노드 간 약속된 데이터 구조**
- calculator 예제는 **패키지 내부 노드 분리 + Topic / Service / Action 통합 설계**를 보여준다

---

# 8. 한 줄 요약

> 6차시는 **패키지 내부에서 노드를 역할별로 나누고, 인터페이스와 통신 방식으로 하나의 시스템을 만드는 법**을 익히는 차시이다.
