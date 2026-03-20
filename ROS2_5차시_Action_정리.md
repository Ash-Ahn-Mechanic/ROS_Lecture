# ROS2 프로그래밍 입문 5차시 정리  
## 액션 패키지

---

# 1. 핵심 내용

## 1.1 Action이란
- 목표 전달 `send_goal`
- 목표 취소 `cancel_goal`
- 결과 받기 `get_result`  
  <!-- goal/result/cancel은 서비스 성격으로 처리됨 -->

## 1.2 Goal State Machine
- `ACCEPTED`
- `EXECUTING`
- `CANCELING`
- `SUCCEEDED`
- `ABORTED`
- `CANCELED`  
  <!-- Action은 상태를 추적하는 통신이라는 점이 핵심 -->

## 1.3 Action Server 역할
- 액션 제공
- goal 수락/거부
- 작업 실행
- feedback 제공
- cancel 요청 처리
- 최종 result 반환  
  <!-- 실제 작업을 수행하는 쪽 -->

## 1.4 Action Client 역할
- goal 전송
- feedback 모니터링
- 상태 모니터링
- cancel 요청
- result 확인  
  <!-- 요청 + 감시를 담당 -->

## 1.5 Action interface 구조
```text
# Goal
...
---
# Result
...
---
# Feedback
...
```
<!-- action 파일은 항상 Goal / Result / Feedback 3구간으로 나뉨 -->

## 1.6 상태 전이
- 서버 트리거
  - `execute`
  - `succeed`
  - `abort`
  - `canceled`
- 클라이언트 트리거
  - `send_goal`
  - `cancel_goal`  
  <!-- 누가 상태를 바꾸는지 구분하는 용도 -->

---

# 2. 핵심 실습 — Fibonacci Action

## 2.1 패키지 생성
```bash
ros2 pkg create --build-type ament_python py_Fibonacci --dependencies rclpy action_tutorials_interfaces
```
<!-- Python action 예제 패키지 생성 -->

## 2.2 setup.py entry_points
```python
entry_points={
    'console_scripts': [
        'fibonacci_action_server = py_fibonacci.fibonacci_action_server:main',
        'fibonacci_action_client = py_fibonacci.fibonacci_action_client:main',
    ],
}
```
<!-- 서버/클라이언트를 ros2 run으로 실행 가능하게 등록 -->

## 2.3 Fibonacci.action
```text
# Goal
int32 order
---
# Result
int32[] sequence
---
# Feedback
int32[] partial_sequence
```
<!-- 몇 항까지 만들지(order) 받고, 중간 수열(partial_sequence), 최종 수열(sequence) 반환 -->

## 2.4 실습 핵심 흐름
```text
client -> goal(order) 전송
server -> 수락
server -> partial_sequence feedback 반복 전송
server -> 최종 sequence result 반환
```
<!-- 시간이 걸리는 작업을 Action으로 구현한 기본 예제 -->

---

# 3. 핵심 정리

- Service는 **요청-응답**
- Action은 **요청-진행상황-최종결과**
- 따라서 **오래 걸리고 중간 상태가 중요한 작업은 Action으로 구현**한다.

---

# 4. 한 줄 요약

> Action은 **시간이 걸리는 작업을 처리하면서 진행 상태를 주고받을 수 있는 ROS2 통신 방식**이다.
