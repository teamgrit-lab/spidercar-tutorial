# 코드 설명 - SpiderDonkey_Async.py

> SpiderCar의 전신이 되는 코드의 설명입니다.
> 

[](https://github.com/teamgrit-lab/hello-mars-spidercar/blob/master/SpiderDonkey_Async.py)

## 📢 Import 부분

---

```python
import json
import asyncio

from math import log
from RTCCam import DualCSICam, SingleCam
from RTCPub import RTCDataChannel
from RCController import Vehicle
from HallSensor import HallLapTimeCheck

from SpiderConfig import SPIDER_CARS
```

사용된 패키지들에 대한 설명을 첨부합니다.

- `json` : 자동차의 제어 영점, 최고 속도 등 여러 매개변수들이 json으로 저장됩니다.
- `asyncio` : SpiderCar에는 rtcbot, 제어, 센싱, 카메라 등 여러 작업들이 비동기적으로 구동됩니다. 이를 위한 파이썬 비동기 라이브러리가 바로 asyncio 입니다.
- `RTCCam` : 카메라를 다루기 위해 제작한 커스텀 패키지입니다. rtcbot과 연동을 위해 추가 소스가 구현되었습니다.
- `RTCPub` : Spider STUN 서버와 연결을 위한 로직과, dataChannel과의 pub sub을 하는 기능들이 들어가있습니다.
- `RCController` : RC카 제어를 위한 커스텀 패키지입니다.
- `HallSensor` : 추가 센서인 자석 센서의 사용을 위한 커스텀 패키지입니다.
- `SpiderConfig` :  자동차의 매개변수를 json으로 저장하기 전에, 파이썬 단에서 인터렉션을 하는 거대한 딕셔너리입니다.

## 📢 전역 변수 설명

---

 자주 수정되는 변수들은 전역으로 설정하였습니다.

```python
CAM_TYPE = "single"
CAR_NAME = "SpiderCar1"
JSON_FILE = '/home/spidercar/Documents/hello-mars-spidercar/calib_param.json'
```

- `CAM_TYPE` : 카메라를 2개 사용할 지 1개 사용할지 선택합니다.
    - single / dual
- `CAR_NAME` : 사용하는 RC 샤시의 이름입니다. SpiderCar0 ~ 19
    
    <aside>
    💡 C언어 배열과 같이 index 0 이 첫번째가 됩니다.
    
    예를 들면, **SpiderCar4 ⇒ 채널명 Mars Racing 3**이 됩니다.
    
    </aside>
    
- `JSON_FILE` : 차량의 모든 매개변수를 json으로 저장하며, 이의 절대 위치를 담고 있습니다.

# SpiderCar 클래스 설명

---

## 📢 생성자 설명

---

```python
self._myCar = Vehicle(self._carname)
self._myIR = HallLapTimeCheck(left_sensor_pin=4, right_sensor_pin=17)

self.setCarParams()

self._stream_data = {}
self._is_back = False
```

> **순서대로 설명하겠습니다.**
> 

### json 차량 매개변수를 불러옵니다.

정적으로 저장되어 있는 각 차량의 최대 속도, 정지 속도, 조향값 등을 불러오는 작업입니다.

```python
	with open(JSON_FILE, 'r') as f:
	    json_data = json.load(f)
	    self.config_dict = json_data[CAR_NAME]
	    print(self.config_dict)
```

### 부모 클래스의 생성자를 호출합니다.

SpiderCar 클래스는 RTCDataChannel의 상속을 받으며, 사용할 채널 선택을 RTCDataChannel에서 이루어집니다. 따라서 super를 통해 부모 클래스의 생성자를 호출하며  이 시점에서 Spider 채널명이 선택됩니다. 

지금의 경우 이는 전역 변수 `**CAR_NAME**`에 따라 달라지네요.  

```python
  super().__init__(channel=self.config_dict["ChannelID"])
```

### 송출할 카메라 설정을 진행합니다.

현재 전방 **single** 카메라와 전후방 **dual** 카메라 모드가 제공되고 있으며, **CAM_TYPE**에 따라 부합하는 클래스를 사용합니다.

`self._cam`에 생성된 카메라 핸들 인스턴스를 할당하고, 이를 `_peer`에 등록합니다.

<aside>
💡 SingleCam, DualCSICam등 카메라 클래스 자체는 별도로 다루도록 하겠습니다.

</aside>

```python
	if CAM_TYPE == "single":
	    self._cam = SingleCam(cam_num=0, width=640, height=480, flip=2)
	elif CAM_TYPE == "dual":
	    self._cam = DualCSICam(
	        cam_num1=0, cam_num2=1, width=640, height=480, flip1=2, flip2=2
	    )
	
	self._peer.video.putSubscription(self._cam)
	
```

### 계속해서 필요한 핸들러들을 생성합니다.

차량을 제어하는 Vehicle 클래스, 적외선/자석 센서를 다루는 클래스를 생성합니다.

<aside>
💡 HallLapTimeCheck 등 핸들러 클래스 자체는 별도로 다루도록 하겠습니다.

</aside>

```python
      self._myCar = Vehicle(self._carname)
      self._myIR = HallLapTimeCheck(left_sensor_pin=4, right_sensor_pin=17)
```

## 📢 setCarParams  함수 호출

---

이는,  json으로 부터 parsing 한 여러가지 자동차의 매개변수를 클래스 내 변수로 저장하는 함수입니다. 함수 내부에 특별한 로직이 있지는 않고, 단순한 대입이 이루어지고 있습니다.

```python

    self.setCarParams()

    self._stream_data = {}
    self._is_back = False

def setCarParams(self):
    # Configure Min Max Values

    self._myCar.speed_min = self.config_dict["Calibration"]["THROTTLE_MIN"]
		...

    self._default_speed = self.config_dict["Calibration"]["THROTTLE_ZERO"]
		...

		print(
            f"""==== Setup Done ====
            THROTTLE_ZERO : {self._myCar.speed_pulse}
            ...
						"""
```

### 실행되는 모든 태스크들은 `asyncio`를 통해 관리됩니다.

asyncio의 사용법은 무척 방대하지만, 현 우리의 프로젝트에서 사용되는 부분만을 살펴보자면 다음과 같습니다. 

```python
def run(self):
    try:
        asyncio.ensure_future(self._myCar.control_loop())
        asyncio.ensure_future(self._myIR.check_loop())
        asyncio.ensure_future(self.connect())
        asyncio.ensure_future(self.receiver())
        asyncio.ensure_future(self.sender())

        self._loop.run_forever()
    except Exception as e:
        print(e)
    finally:
        self._cam.close()
        print("Done...")

if __name__ == "__main__":
    
    myRTCBot = SpiderCar()
    myRTCBot.run()
```

쉽게 말해서 asyncio를 사용하면 비동기 처리로 **무한 while 루프를 여려개 실행 가능**합니다.

각각의 무한 while 루프들이 서로 비동기로 실행되면서 자원의 공유를 하는 방식이지요.

`ensure_future`는 인자로 들어오는 함수가 종료될 때까지 지속 실행되는 것을 보장해주는 함수이며,  여러 무한 while 문들을 `ensure_future`로 등록한 뒤, **run_forever**를 통해 일괄 실행하게 됩니다.

 지금부터는 `ensure_future`로 등록된 함수들을 하나하나 뜯어보겠습니다.

## 📢 receiver 함수

---

**dataChannel**로 유입된 데이터를 받아 차량에게 전달하기 위한 형태 변형이 일어납니다. 그런 다음 실제 구동까지를 담당하는 함수이며, 길이가 길어 조금씩 나누어 분석해보겠습니다.

- `calcControlParams`

*예를 들어, 전진속도 1에서는 실제 차량에 375의 제어값을 주어야 한다 등. 제어와 관련된 매개변수를 미리 계산하는 함수입니다.* 

여기서 전진 스텝이 계산되는 로직은 다음과 같습니다.

$전진스텝 = \frac{최대 전진 속도 - 최소 전진 속도 }{제어 스텝}$

1 ⇒ 10으로 점차 높은 값을 보내면, raw 제어값도 이에 비례하여 선형적으로 커지는 로직입니다.

위 과정을 후진 스텝, 우회전 스텝, 좌회전 스텝에 동일하게 적용하여 모두 계산한 뒤, 이후 과정에서  계속해서 사용하게 됩니다. 

```python
def calcControlParams(self):
        self.forward_gain = (
            self._myCar.speed_max - self._default_forward
        ) / self._throttle_step

        self.forward_quadratic_gain = (
            self._myCar.speed_max - self._default_forward
        ) / (self._throttle_step ** 2)

        self.backward_gain = (
            self._default_speed - self._myCar.speed_min
        ) / self._throttle_step

        self.steering_left_gain = (self._myCar.steering_max - self._default_steering) / (
            self._steering_step
        )

        self.steering_right_gain = (self._default_steering - self._myCar.steering_min) / (
            self._steering_step
        )
        print(f"=== {self.forward_gain} {self.forward_quadratic_gain} {self.steering_left_gain}, {self.steering_right_gain}===")
```

### receiver 루프 로직

다시 receiver 함수 내부로 들어와서, 계산된 제어값을 사용하여 실제 차량을 구동하는 부분을 살피고자합니다.

이때, `stream` dataChannel로 유입되는 정보는 두가지가 있습니다.

- speed를 key로 갖는 속도 설정 정보

- motion을 key로 갖는 제어 정보

위 두가지 정보가 모두 같은 dataChannel로 들어오기 때문에, 이를 선별하여 각각에 적합한 후속 절차를 진행하는 작업이 필요합니다. 

### speed 데이터 처리

speed 데이터에는 설정할 최대 속도 값이 담겨있습니다.

이에 따라 해당 값을 추출한 뒤, 바뀐 최대 속도에 맞추어  calcControlParams을 다시 호출하고, 바뀐 최대 속도를 다시 사용하기 위해  정적 json 파일로 저장하는 작업이 필요합니다. 

더불어, echo request를 위해 metric dataChannel로 유입된 데이터와 동일한 값을 보내고 있습니다. 

```python
elif "speed" in self._stream_data:
  print(self._stream_data["speed"])
  if self._stream_data["speed"]["key"] == "maxSpeed":
      self._myCar.speed_max = self._stream_data["speed"]["value"]
      self.calcControlParams()

      with open(JSON_FILE, 'w') as f:
          self.config_dict["Calibration"]["THROTTLE_MAX"] = self._myCar.speed_max
          SPIDER_CARS[CAR_NAME] = self.config_dict
          json.dump(SPIDER_CARS, f, indent=4)

      **recali_metric = {"speed": {"key": "maxSpeed", "value": self._myCar.speed_max}}
      self._dc_metric.put_nowait(recali_metric)
  continue
```

### motion 데이터 처리

우선 `stream_data` 에서 필요한 부분인 key와 value 만을 골라냅니다. 

```python
		self._stream_data = await self._dc_subscriber.get()

    if "motion" in self._stream_data:
        key = self._stream_data["motion"]["key"]
        val = self._stream_data["motion"]["value"]
```

여기서 key 값에 따라 각각 별도의 후속 작업이 이루어집니다. 

```python
if key == "forward":
              
elif key == "back":

if key == "left":

elif key == "right":
```

기본적으로, 전후진과 조향의 제어가 들어가기 때문에 val을 실제 조향/전후진 값으로 다시 계산하는 과정이 필요합니다.  전후진 값은 `self._throttle` 로, 조향값은 `self._angle` 로 저장됩니다.

```python
self._throttle = (
    (self._default_forward + int( self.forward_gain * (val) ))
    if val > 0
    else self._default_speed
)

self._angle = (
    (self._default_steering + self.steering_left_gain * (val))
    if val > 0
    else self._default_steering
)
```

val이 0보다 작은 상황에 대한 예외처리가 들어가 있습니다.

### 후진 로직

후진의 경우, 별도의 로직이 추가됩니다.

이렇게 하는 이유는, 현재 사용중인 RC카가 **전진 후 바로 후진이 되지 않습니다.** ( RC 경기 룰이 원래 후진이 불가이기 때문에 그런 것이 아닐까 짐작합니다.)

따라서 전진을 하고 있는 상황에서 후진 명령이 들어오면, 우선 정지에 해당하는 신호를 잠깐 동안 전달한 뒤 다시 후진 명령을 내리게 됩니다. 

```python
if self._is_back == False:
    for i in range(10):
        self._myCar.user_control(self._default_speed - 50, self._angle)

    for i in range(10):
        self._myCar.user_control(self._default_speed, self._angle)

    self._is_back = True
```

### 후진 카메라 로직

 `CAM_TYPE`이 dual 일 시, 후진 명령어가 들어오게 되면 자동으로 송출 영상이 후진 카메라도 변경됩니다. 이를 구현하는 로직은 다음과 같이 구성되어 있습니다.

```python
if key == "forward":
    ...
    self._cam.set_mode(True)
    self._is_back = False
elif key == "back":
    ...
    if CAM_TYPE == "dual":
        self._cam.set_mode(False)
		...
```

### 실 차량 제어

여러 기능 추가가 많아 잠시 다른 길로 빠졌는데요. 결국 계산된 전후진과 조향값으로 실제 차량이 제어되며 이는 아래 코드와 같이 `self._myCar`에게 전달해주는 식으로 작동합니다.

```python
self._myCar.speed_pulse = self._throttle
self._myCar.steering_pulse = self._angle
print(self._throttle, self._angle)
```

## 📢 sender 함수

---

SpiderCar에는 각종 센서들이 부착될 수 있습니다.

이런 센서들로부터 받은 데이터는 `metric` 이라는 이름의 dataChannel을 사용해서 전달되는데요.

이를 실제 사용하기 위한 함수가 sender안에 구현되어 있습니다.

metric 예시 중 **적외선 (ir)** 센서 활용을 예시로 들어 데이터 구조를 살펴보겠습니다.

```python
{
	"ir":{
		"key":"detectTime",
		"value": 0.0
	}
}
```

ir 센서의 경우, 위와 같이 key로는 해당 센서가 측정 당시의 시간 정보를 보내도록 구성되어 있으며, 

이때의 값은 timestamp를 받도록 되어 있습니다.

따라서, SpiderCar 단에서는 센서가 센싱한 그 타이밍에, timestamp를 가져와서 value에 넣고 보내게 되는 것이지요. 

이에 대한 코드는 아래와 같습니다.

```python
async def sender(self):

    ir_metric = {"ir": {"key": "detectTime", "value": 0.0}}
    prev_black = False

    while True:
        is_black, cur_timestamp = self._myIR.get_lineinfo()

        if prev_black and not is_black:
            print(is_black, cur_timestamp)
            ir_metric["ir"]["value"] = cur_timestamp
            self._dc_metric.put_nowait(ir_metric)
        
        prev_black = is_black
        await asyncio.sleep(0)
```

내부 로직을 살펴보면, 센서가 반응하고 있다가 센서가 반응하지 않는 그 시점에 (검은선을 센싱하고  다시 흰 선을 보는 순간 / 자석 위를 지나다가 다시 일반 땅을 보는 순간 ) metric publish 가 일어난다는 점을 알 수 있습니다. 

 

센서 데이터의 publish는 **rtcbot**에 의해 동작되며,  `put_nowait` 함수를 통해 보낼 텍스트 데이터가 송신됩니다. 

## 📢 그 외 함수들

---

기타 control_loop, check_loop, connect 함수는 모두 다른 클래스에서 핸들링하는 부분으로,

- 차량에게 계속 제어 신호를 전달하기
- 센서의 센싱 여부를 지속 주시하기
- Peer 연결을 지속 주시하기

위의 기능들을 계속해서 수행하는 함수들입니다.

해당하는 핸들러의 설명 페이지에서 다시 살피도록 하겠습니다.

```python
asyncio.ensure_future(self._myCar.control_loop())
asyncio.ensure_future(self._myIR.check_loop())
asyncio.ensure_future(self.connect())
asyncio.ensure_future(self.receiver())
asyncio.ensure_future(self.sender())
```

> **끝!!!**
>