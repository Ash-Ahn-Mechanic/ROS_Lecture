# ROS2 프로그래밍 입문 9차시 정리  
## ROS2 응용 핵심정리 — 주요 함수 + 카메라 영상 실습

---

# 1. 핵심 내용

## 1.1 차시 목적
- Topic, Service, Action 관련 주요 함수(method) 정리
- 이전 차시에서 배운 통신 구조를 코드 관점에서 다시 확인
- 카메라 영상 다루기 실습으로 ROS2 응용 확장  
  <!-- 9차시는 앞선 내용을 함수 중심으로 정리하고, 센서 토픽 실습으로 연결하는 차시 -->

---

# 2. Topic 관련 주요 함수

## 2.1 Publisher
```python
publisher = self.create_publisher(MsgType, 'topic_name', qos_profile)
publisher.publish(msg)
```
- `create_publisher(...)`  
  <!-- 퍼블리셔 생성 -->
- `publish(msg)`  
  <!-- 메시지 전송 -->

## 2.2 Subscriber
```python
self.subscription = self.create_subscription(
    MsgType,
    'topic_name',
    self.callback,
    qos_profile
)
```
- `create_subscription(...)`  
  <!-- 서브스크라이버 생성 -->
- `callback(msg)`  
  <!-- 수신한 메시지를 처리하는 함수 -->

## 2.3 Topic 핵심 흐름
```text
Publisher 생성
-> 메시지 publish
-> Subscriber가 수신
-> callback에서 처리
```
<!-- 가장 기본적인 ROS2 데이터 흐름 -->

---

# 3. Service 관련 주요 함수

## 3.1 Client
```python
self.cli = self.create_client(SrvType, "service_name")

req = SrvType.Request()
req.param = value

while not self.cli.wait_for_service(timeout_sec=1.0):
    self.get_logger().info("Waiting for service...")

future = self.cli.call_async(req)
future.add_done_callback(response_callback)
```
- `create_client(...)`  
  <!-- 서비스 클라이언트 생성 -->
- `SrvType.Request()`  
  <!-- 요청 객체 생성 -->
- `wait_for_service(...)`  
  <!-- 서버가 준비될 때까지 대기 -->
- `call_async(req)`  
  <!-- 비동기 서비스 요청 -->
- `future.add_done_callback(...)`  
  <!-- 결과 도착 후 실행할 콜백 연결 -->

## 3.2 Future 기반 결과 처리
- 서비스 호출 결과는 `Future` 객체로 반환
- `future.result()`로 실제 응답 확인 가능
- 필요 시 `rclpy.spin_once()` 또는 `spin_until_future_complete()` 사용  
  <!-- Service는 요청 즉시 결과가 오는 것이 아니라 Future를 통해 나중에 받음 -->

## 3.3 Server
```python
self.srv = self.create_service(SrvType, 'service_name', self.call_back)
```
- `create_service(...)`  
  <!-- 서비스 서버 생성 -->
- `callback(request, response)`  
  <!-- 요청 처리 함수 -->
- `response.param = value`  
  <!-- 응답 값 설정 -->
- `return response`  
  <!-- 최종 응답 반환 -->

## 3.4 Service 핵심 흐름
```text
Client 생성
-> Request 작성
-> call_async()
-> Server callback 실행
-> Response 반환
-> Client가 Future로 결과 수신
```
<!-- 요청 / 응답 구조를 코드 흐름으로 정리 -->

---

# 4. Action 관련 주요 함수

## 4.1 Action Client
```python
self._action_client = ActionClient(self, ActionType, 'action_name')
self._action_client.wait_for_server()

goal_msg = ActionType.Goal()
goal_msg.param = value

self._send_goal_future = self._action_client.send_goal_async(
    goal_msg,
    feedback_callback=self.feedback_callback
)
self._send_goal_future.add_done_callback(self.goal_response_callback)
```
- `ActionClient(...)`  
  <!-- 액션 클라이언트 생성 -->
- `wait_for_server()`  
  <!-- 액션 서버 준비 대기 -->
- `ActionType.Goal()`  
  <!-- goal 메시지 작성 -->
- `send_goal_async(...)`  
  <!-- goal 비동기 전송 -->
- `feedback_callback=...`  
  <!-- 중간 피드백 처리 -->
- `goal_response_callback(...)`  
  <!-- goal 수락 여부 확인 -->

## 4.2 Result 수신
```python
goal_handle.get_result_async()
```
- `get_result_async()`  
  <!-- 최종 결과 비동기 요청 -->

## 4.3 Action 실행 흐름
```text
send_goal_async(goal_msg)
  -> feedback_callback()
  -> goal_response_callback(future)
      -> get_result_async()
          -> get_result_callback(future)
```
<!-- Action은 Goal -> Feedback -> Result 순서로 진행 -->

## 4.4 Action 핵심 정리
- Service처럼 요청만 보내고 끝나는 구조가 아님
- 중간 진행 상태를 받을 수 있음
- 오래 걸리는 작업 처리에 적합  
  <!-- Topic / Service / Action 차이를 코드 수준에서 다시 확인 -->

---

# 5. 핵심 실습 — 카메라 영상 다루기

## 5.1 실습 목표
- 카메라 영상을 ROS2 토픽으로 publish
- 다른 노드에서 subscribe하여 확인
- bag으로 저장하고 다시 재생  
  <!-- 8차시의 bag 실습과 9차시의 토픽 실습이 연결됨 -->

## 5.2 실습 순서
1. 패키지 생성: `my_cam_pubsub`
2. `setup.py` 수정
3. `package.xml` 수정
4. 소스 코드 추가
5. 파일명 `cam_pub.py`, `cam_sub.py`
6. `colcon build --packages-select my_cam_pubsub`
7. `source install/setup.bash`
8. `rviz2`에서 영상 보기
9. `ros2 bag record`로 저장
10. `ros2 bag play`로 영상 출력  
   <!-- 패키지 생성부터 영상 확인, 저장/재생까지 전체 흐름 -->

---

# 6. 핵심 실습 코드

## 6.1 패키지 생성
```bash
ros2 pkg create --build-type ament_python my_cam_pubsub --dependencies rclpy sensor_msgs cv_bridge
```
<!-- 이미지 토픽과 OpenCV 변환을 위해 sensor_msgs, cv_bridge 필요 -->

---

## 6.2 cam_pub.py

### import
```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import Image
from cv_bridge import CvBridge
import cv2
```
- `sensor_msgs.msg.Image`  
  <!-- ROS 이미지 메시지 타입 -->
- `CvBridge`  
  <!-- OpenCV 이미지와 ROS Image 메시지 사이 변환 도구 -->
- `cv2`  
  <!-- 카메라 프레임 읽기 및 영상 처리 -->

### 클래스 / 초기화
```python
class CameraPublisher(Node):
    def __init__(self):
        super().__init__("camera_publisher")
        self.publisher = self.create_publisher(Image, "camera/image_raw", 10)
        self.timer = self.create_timer(0.1, self.timer_callback)
        self.cap = cv2.VideoCapture(0)
        self.bridge = CvBridge()
```
- `create_publisher(Image, "camera/image_raw", 10)`  
  <!-- 카메라 이미지를 publish할 토픽 생성 -->
- `create_timer(0.1, ...)`  
  <!-- 0.1초마다 실행 = 약 10Hz -->
- `cv2.VideoCapture(0)`  
  <!-- 기본 카메라 장치 열기 -->
- `CvBridge()`  
  <!-- OpenCV 프레임을 ROS 메시지로 바꾸기 위해 사용 -->

### timer_callback
```python
def timer_callback(self):
    ret, frame = self.cap.read()
    if not ret:
        self.get_logger().error("Failed to read from camera")
        return
    msg = self.bridge.cv2_to_imgmsg(frame, encoding="bgr8")
    self.publisher.publish(msg)
    self.get_logger().info("Published image")
```
- `cap.read()`  
  <!-- 카메라 프레임 읽기 -->
- `if not ret:`  
  <!-- 프레임 읽기 실패 시 종료 -->
- `cv2_to_imgmsg(frame, encoding="bgr8")`  
  <!-- OpenCV 이미지를 ROS Image 메시지로 변환 -->
- `publish(msg)`  
  <!-- 토픽으로 송신 -->

### main
```python
def main():
    rclpy.init()
    node = CameraPublisher()
    rclpy.spin(node)
    node.cap.release()
    node.destroy_node()
    rclpy.shutdown()

if __name__ == "__main__":
    main()
```
- `rclpy.spin(node)`  
  <!-- 노드를 계속 실행 상태로 유지 -->
- `node.cap.release()`  
  <!-- 종료 시 카메라 자원 해제 -->

---

## 6.3 cam_sub.py

### import
```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import Image
from cv_bridge import CvBridge
import cv2
```
<!-- 퍼블리셔와 유사하지만, 여기서는 수신 후 화면 표시가 목적 -->

### 초기화
```python
class CameraSubscriber(Node):
    def __init__(self):
        super().__init__("camera_subscriber")
        self.subscription = self.create_subscription(
            Image,
            "camera/image_raw",
            self.listener_callback,
            10
        )
        self.bridge = CvBridge()
```
- `create_subscription(Image, "camera/image_raw", ...)`  
  <!-- 퍼블리셔가 보낸 카메라 토픽 구독 -->
- `CvBridge()`  
  <!-- ROS Image 메시지를 OpenCV 이미지로 바꾸기 위해 사용 -->

### listener_callback
```python
def listener_callback(self, msg):
    frame = self.bridge.imgmsg_to_cv2(msg, desired_encoding="bgr8")
    cv2.imshow("Camera Subscriber", frame)
    cv2.waitKey(1)
```
- `imgmsg_to_cv2(...)`  
  <!-- ROS 이미지 메시지를 OpenCV 프레임으로 변환 -->
- `cv2.imshow(...)`  
  <!-- 화면에 이미지 출력 -->
- `cv2.waitKey(1)`  
  <!-- 영상 창 갱신 -->

---

# 7. 빌드 및 실행

## 7.1 빌드
```bash
colcon build --packages-select my_cam_pubsub
```
<!-- 패키지 빌드 -->

## 7.2 환경 반영
```bash
source install/setup.bash
```
<!-- 빌드된 패키지를 현재 터미널 환경에 반영 -->

## 7.3 실행
```bash
ros2 run my_cam_pubsub cam_pub
ros2 run my_cam_pubsub cam_sub
```
<!-- 한쪽은 카메라 publish, 다른 쪽은 subscribe 후 화면 표시 -->

---

# 8. Bag과 연계한 영상 저장 / 재생

## 8.1 저장
```bash
ros2 bag record /camera/image_raw
```
<!-- 카메라 토픽 기록 -->

## 8.2 재생
```bash
ros2 bag play <bag_file_name>
```
<!-- 저장된 영상 토픽 다시 재생 -->

## 8.3 의미
- 실시간 카메라 입력을 바로 확인 가능
- 기록한 영상을 나중에 다시 재생 가능
- rviz2와 연결하면 ROS 시각화 도구로도 확인 가능  
  <!-- 8차시의 bag 실습이 9차시 센서 토픽 응용으로 이어짐 -->

---

# 9. 핵심 정리

- Topic은 `create_publisher`, `publish`, `create_subscription` 중심
- Service는 `create_client`, `call_async`, `create_service` 중심
- Action은 `ActionClient`, `send_goal_async`, `get_result_async` 중심
- 카메라 실습은 ROS 이미지 토픽과 OpenCV를 연결하는 기본 예제
- bag을 함께 사용하면 영상 저장 / 재생까지 가능

---

# 10. 한 줄 요약

> 9차시는 **Topic / Service / Action의 주요 함수를 코드 수준에서 정리하고, 카메라 영상을 ROS2 토픽으로 송수신하며 bag으로 저장·재생하는 응용 실습**을 다루는 차시이다.
