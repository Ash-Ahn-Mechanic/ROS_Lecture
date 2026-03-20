# ROS2 프로그래밍 입문 7차시 정리  
## 인터페이스 프로그래밍(응용 2) — Hangman 패키지

---

# 1. 핵심 내용

## 1.1 실습 주제
- Python을 이용하여 행맨(Hangman) 게임 만들기
- Topic, Service, Action을 함께 사용하여 게임 진행 구조 구현  
  <!-- 6차시의 계산기 예제를 게임 형태로 확장한 응용 실습 -->

## 1.2 Hangman 게임 개요
- 단어나 문구를 맞추는 고전적인 단어 게임
- 게임의 목적은 플레이어가 주어진 단어를 맞추는 것
- 한 명의 플레이어(또는 시스템)가 특정 단어를 선택
- 다른 플레이어는 알파벳 한 글자씩 추측
- 맞는 글자를 추측하면 해당 위치에 표시
- 틀린 글자를 추측할 때마다 기회가 줄어듦
- 모든 글자를 맞추면 승리
- 모든 기회가 소진되면 패배  
  <!-- 게임 규칙 자체를 ROS 통신 구조로 바꿔 구현 -->

## 1.3 전체 구조도
```text
Letter publisher -> topic publish
User input -> Word service request
Action client <-> Action server
progress publish
game_progress request
```
<!-- Topic / Service / Action이 각각 게임의 입력, 판정, 진행상태 관리 역할을 맡음 -->

## 1.4 구성 요소별 역할

### Letter Publisher
- `a`부터 `z`까지의 알파벳을 순서대로 publish  
  <!-- 자동 입력 생성기 역할 -->

### Word Service
- 임의의 단어를 선택
- 행맨 게임 진행
- 진행 상황 publish  
  <!-- 실제 정답 판정과 현재 상태 관리 담당 -->

### Action Client
- Goal 설정
- action server와 상호작용을 통해 행맨 게임 진행 상태 업데이트  
  <!-- 게임 시작 및 상태 요청 담당 -->

### Action Server
- 사용자의 진행 상황을 관리
- 진행 상태를 추적
- 게임 결과를 클라이언트에게 전달  
  <!-- 게임 전체 상태를 관리하는 감시/제어 역할 -->

---

# 2. 핵심 실습 구조

## 2.1 패키지 생성
```bash
ros2 pkg create --build-type ament_python hangman_game --dependencies rclpy hangman_interfaces std_msgs
ros2 pkg create --build-type ament_cmake hangman_interfaces
```
<!-- 실행 패키지와 인터페이스 패키지를 분리 생성 -->

## 2.2 전체 코드 구조
```text
hangman_game/
 ├─ hangman_game/
 │   ├─ __init__.py
 │   ├─ letter_publisher.py
 │   ├─ progress_action_client.py
 │   ├─ progress_action_server.py
 │   ├─ user_input.py
 │   └─ word_service.py
 ├─ resource/
 ├─ setup.cfg
 └─ setup.py

hangman_interfaces/
 ├─ action/
 │   └─ GameProgress.action
 ├─ msg/
 │   └─ Progress.msg
 ├─ srv/
 │   └─ CheckLetter.srv
 ├─ CMakeLists.txt
 └─ package.xml
```
<!-- 실행 로직과 인터페이스 정의를 패키지 단위로 분리 -->

---

# 3. 인터페이스 정의

## 3.1 CheckLetter.srv
```text
# Empty request
---
string updated_word_state
bool is_correct
string message
```
<!-- 요청은 비어 있고, 현재 입력된 글자를 내부 상태로 판정해 응답 -->

## 3.2 Progress.msg
```text
string current_state
int32 attempts_left
bool game_over
bool won
```
<!-- 현재 맞춘 상태, 남은 시도 횟수, 종료 여부, 승리 여부 표현 -->

## 3.3 GameProgress.action
```text
# Goal
# Empty since the client doesn't need to send any data

---
# Result
bool game_over
bool won

---
# Feedback
bool game_over
```
<!-- 클라이언트가 별도 데이터를 보내지 않고 게임 진행 상태만 추적 -->

---

# 4. 핵심 실습 코드

## 4.1 letter_publisher.py
```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String

class LetterPublisher(Node):
    def __init__(self):
        super().__init__('letter_publisher')
        self.publisher_ = self.create_publisher(String, 'letter_topic', 10)
        self.timer = self.create_timer(1.0, self.publish_letter)
        self.current_letter = ord('a')

    def publish_letter(self):
        msg = String()
        msg.data = chr(self.current_letter)
        self.publisher_.publish(msg)
        self.get_logger().info(f'Publishing: {msg.data}')
        self.current_letter += 1
        if self.current_letter > ord('z'):
            self.current_letter = ord('a')

def main(args=None):
    rclpy.init(args=args)
    letter_publisher = LetterPublisher()
    rclpy.spin(letter_publisher)
    letter_publisher.destroy_node()
    rclpy.shutdown()
```
- `create_publisher(String, 'letter_topic', 10)`  
  <!-- 문자 토픽 발행 -->
- `create_timer(1.0, self.publish_letter)`  
  <!-- 1초마다 실행 -->
- `ord('a')`, `chr(...)`  
  <!-- 아스키 코드로 a~z 순환 구현 -->

## 4.2 word_service.py 구조
```python
class WordService(Node):
    def __init__(self):
        super().__init__('word_service')
        self.srv_service = self.create_service(CheckLetter, 'check_letter', self.check_letter_callback)
        self.subscription = self.create_subscription(String, 'letter_topic', self.letter_callback, 10)
        self.progress_publisher = self.create_publisher(Progress, 'progress', 10)

        data = Progress()
        data.current_state = ""
        data.attempts_left = 20
        data.game_over = False
        data.won = False
        self.progress_publisher.publish(data)

        self.current_letter = ""
        self.word_list = ['python', 'hangman', 'robot', 'ros', 'interface']
        self.word = random.choice(self.word_list)
        self.word_state = ['_'] * len(self.word)
        self.attempts_left = 20
```
- `create_service(CheckLetter, 'check_letter', ...)`  
  <!-- 글자 판정 서비스 생성 -->
- `create_subscription(String, 'letter_topic', ...)`  
  <!-- 최신 입력 글자 수신 -->
- `create_publisher(Progress, 'progress', 10)`  
  <!-- 현재 상태 발행 -->
- `random.choice(self.word_list)`  
  <!-- 정답 단어 랜덤 선택 -->
- `['_'] * len(self.word)`  
  <!-- 초기 화면을 빈칸 상태로 구성 -->

## 4.3 letter_callback
```python
def letter_callback(self, msg):
    self.current_letter = msg.data
```
<!-- 토픽으로 들어온 최신 글자를 저장 -->

## 4.4 check_letter_callback
```python
def check_letter_callback(self, request, response):
    letter = self.current_letter
    if letter in self.word:
        for idx, char in enumerate(self.word):
            if char == letter:
                self.word_state[idx] = letter
        response.is_correct = True
        response.message = "Correct!"
    else:
        self.attempts_left -= 1
        response.is_correct = False
        response.message = "False"

    response.updated_word_state = ''.join(self.word_state)

    progress_msg = Progress()
    progress_msg.current_state = response.updated_word_state
    progress_msg.attempts_left = self.attempts_left
    progress_msg.game_over = '_' not in self.word_state or self.attempts_left <= 0
    progress_msg.won = '_' not in self.word_state
    self.progress_publisher.publish(progress_msg)
    return response
```
- `if letter in self.word:`  
  <!-- 입력 글자가 정답 단어에 포함되는지 검사 -->
- `self.word_state[idx] = letter`  
  <!-- 맞춘 위치 갱신 -->
- `self.attempts_left -= 1`  
  <!-- 틀리면 기회 차감 -->
- `''.join(self.word_state)`  
  <!-- 현재 상태를 문자열로 결합 -->
- `game_over`, `won` 계산  
  <!-- 종료 조건과 승리 여부 판단 -->
- `self.progress_publisher.publish(progress_msg)`  
  <!-- 상태를 토픽으로 발행 -->

## 4.5 progress_action_client.py
- Goal 전송
- 서버 대기
- goal accepted 여부 확인
- feedback / result 수신  
  <!-- 액션 클라이언트는 게임 시작과 결과 수신 담당 -->

## 4.6 progress_action_server.py
- 진행 상태 관리
- 남은 시도 횟수 추적
- 성공 / 실패 결과 반환  
  <!-- 액션 서버는 게임 전체 상태를 제어 -->

## 4.7 user_input.py
- 사용자 입력을 받아 서비스 / 액션 흐름과 연결  
  <!-- 실제 플레이어 입력을 ROS 구조에 연결하는 역할 -->

---

# 5. 실행 구조 요약

## 5.1 역할 분리
```text
letter_publisher  -> 알파벳 발행
word_service      -> 정답 판정 + 진행 상태 publish
user_input        -> 사용자 입력 처리
action_client     -> 진행 요청
action_server     -> 전체 게임 상태 관리
```
<!-- 패키지 내부 노드가 명확히 역할 분담됨 -->

## 5.2 전체 흐름
```text
알파벳 입력 수신
-> 서비스로 정답 판정
-> 상태 메시지 publish
-> 액션으로 진행상황 관리
-> 승리/패배 결과 반환
```
<!-- Topic / Service / Action을 한 게임 흐름 안에서 결합 -->

---

# 6. 핵심 정리

- 6차시가 계산기였다면, 7차시는 게임 형태 응용
- 패키지 내부 노드를 역할별로 분리
- 인터페이스를 msg / srv / action으로 정의
- Topic / Service / Action을 조합해 실제 프로그램 구조 구현

---

# 7. 한 줄 요약

> 7차시는 **패키지 내부 노드를 분리하고, 인터페이스와 통신 구조를 이용해 실제 게임 로직을 ROS2 시스템으로 구현하는 차시**이다.
