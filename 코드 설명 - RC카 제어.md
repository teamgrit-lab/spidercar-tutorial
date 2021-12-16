# 코드 설명 - RC카 제어

> SpiderCar의 하부 샤시를 제어하는 코드를 설명해봅니다.
> 

코드는 다음에 위치하고 있습니다.

[rc_controller.py](https://github.com/teamgrit-lab/hello-mars-spidercar/blob/master/RCController/rc_controller.py)

이번 부분은 하드웨어를 제어하는 것이기 때문에, 발생할 수 있는 여러 오류 상황들이 있습니다.

해당 문제가 하드웨어 문제인지, 소프트웨어 문제인지를 잘 판별하여 디버깅하시기 바랍니다. 

## 📢 연결 확인

- 모터드라이버인 `PCA9685` 연결 확인

```bash
$ sudo i2cdetect -r -y 1
0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: 40 -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: 70 -- -- -- -- -- -- --
```

위와 같이 40과 70이 확인되어야 제대로 연결된 것입니다. 

그렇지 않은 경우 조립 배선을 잘 확인해주세요!

다음으로 사용하는 차량의 **베터리가 충분한지** 반드시 확인하시기 바랍니다**.**

차량 전원을 켜봄으로 손쉽게 확인할 수 있습니다. 

## 📢 코드 설명

이번 코드는 크게 두 종류의 클래스로 구성되어 있습니다.

- `PCA9685` : PCA9685의 채널 하나를 다룰 수 있는 핸들러 클래스 입니다.
- `Vehicle` : 사용하는 차량은 PCA9685 채널 2개를 사용합니다. 따라서 `PCA9685` 인스턴스를 2개 포함하고 있으며, 기타 asyncio와 연동되기 위한 여러 함수들이 구현되어 있습니다.

## 📢 `PCA9685`  클래스 설명

---

- 생성자 부분입니다.

```python
def __init__(
        self, channel, address=0x40, frequency=60, busnum=None, init_delay=0.1
  ):

    self.default_freq = 60
    self.pwm_scale = frequency / self.default_freq

    import Adafruit_PCA9685

    # Initialise the PCA9685 using the default address (0x40).
    if busnum is not None:
        from Adafruit_GPIO import I2C

        # replace the get_bus function with our own
        def get_bus():
            return busnum

        I2C.get_default_bus = get_bus
    self.pwm = Adafruit_PCA9685.PCA9685(address=address)
    self.pwm.set_pwm_freq(frequency)
    self.channel = channel
    time.sleep(init_delay)  # "Tamiya TBLE-02" makes a little leap otherwise

    self.pulse = 340
    self.prev_pulse = 340
    self.running = True
```

파이썬으로 PCA9685를 다룰 수 있는 패키지인 `Adafruit_PCA9685`를 사용합니다. 

**address, busnum, frequency**등 여러 옵션들이 있지만, 현재는 크게 사용하지 않고, 위 설정 그대로 사용하시면 됩니다. 

- `address` : 0x40, 일전 i2cdetect 시, 40과 70이 나왔던 것을 기억하시나요? 따라서 40이 사용됩니다.
- `frequency` : 60, pwm 주파수이며, 60Hz를 사용합니다.
- `busnum` : None, 여러개의 PCA9685를 사용할 경우, 이를 사용할 수 있는데 현재는 미사용합니다.

위 값들을 바탕으로 pwm 핸들러를 생성하며 코드상으로는 아래와 같습니다.

```python
self.pwm = Adafruit_PCA9685.PCA9685(address=address)
```

기본 pulse는 340을 사용하고 있습니다.

```python
self.pulse = 340
```

- pwm pulse를 설정하기 위해선 `set_pwm` 함수를 사용합니다.

```python
def set_pwm(self, pulse):
    try:
        self.pwm.set_pwm(self.channel, 0, int(pulse * self.pwm_scale))
    except:
        self.pwm.set_pwm(self.channel, 0, int(pulse * self.pwm_scale))
```

pulse를 변경할 채널 번호, on/off (거의 무조건 0을 사용), 변경할 pulse를 순서대로 입력합니다.

pulse는 정수형만이 사용 가능하기 때문에 `int`로 형변환을 해줍니다.

## 📢 `Vehicle` 클래스 설명

---

실제 차량을 추상화한 클래스 입니다.

생성자 부를 살펴보면 PCA9685 클래스 2개를 포함하고 있는 것을 확인할 수 있습니다.

이는 전후진/조향 제어가 필요함에 따라 각각에 해당하는 채널 핸들러를 추상화한 것입니다.

```python
def __init__(self, name="donkey_ros"):

	  self._throttle = PCA9685(channel=0, busnum=1)
	  print("Throttle Controller Awaked!!")
	
	  self._steering_servo = PCA9685(channel=1, busnum=1)
	  print("Steering Controller Awaked!!")
```

- `self._throttle` :  전후진 pwn 채널 담당
- `self._steering_servo` :  좌우 조향 pwm 채널 담당

**rtcbot**이 **asyncio** 형식을 사용함에 따라 이 또한 비동기로 짜여져 있습니다.

때문에, 아래와 같이 제어 자체를 이벤트 루프로 감싸서 사용합니다. 

```python
def run(self):
    try:
        asyncio.ensure_future(self.control_loop())
        asyncio.ensure_future(self.input_loop())
        self._loop.run_forever()
    except Exception as e:
        print(e)
    finally:
        print("Done...")
```

`control_loop`가 제어를 위한 이벤트 루프이며, 이에 대한 설명을 이어나가겠습니다.

```python
async def control_loop(self):
    while True:
        await self._loop.run_in_executor(None, self.control)
```

control_loop 또한 ascynio 사용을 위해 control 이라는 함수를 감싸고 있습니다.

이와 같은 구현이 된 이유는 모두 **rtcbot과의 호환을 위함임**을 다시 한 번 상기시켜드립니다.

```python
def control(self):  # , speed, steering_angle
    if DEBUG_MODE:
        print(
            "speed_pulse : "
            + str(self._speed_pulse)
            + " / "
            + "steering_pulse : "
            + str(self._steering_pulse)
        )

    self._throttle.run(self._speed_pulse)
    self._steering_servo.run(self._steering_pulse)
```

control 함수가 하는 일은 단순합니다. 현재 설정되어있는 전후진과 조향 값을 지속적으로 전달하고 있습니다.

대부분 하드웨어를 다루거나 임베디드 프로그래밍을 하면, 이렇게 무한 while 루프를 걸고, 그 안에서 제어값을 바꾸게 되며, 현재 상황도 이와 같은 맥락입니다.

값을 변경하고 싶다면 `user_control` 함수를 사용합니다.

```python
def user_control(
        self, speed_pulse_in, steering_pulse_in
    ):
    if DEBUG_MODE:
        print(
            "User Control: speed_pulse : "
            + str(speed_pulse_in)
            + " / "
            + "steering_pulse : "
            + str(steering_pulse_in)
        )

    self._throttle.run(speed_pulse_in)
    self._steering_servo.run(steering_pulse_in)
```

매개변수로 들어온 전후진/조향 값을 **PCA9685** 클래스인 `self._throttle`, `self._steering_servo` 로 전달하게 되며, 이는 자동적으로 갱신되고 실제 RC카로 전달됩니다.

실제 **SpiderDonkey_Async.py**를 보면, 아래와 같이 `user_control`을 사용하여 제어하고 있습니다. 

```python
for i in range(10):
    self._myCar.user_control(self._default_speed - 50, self._angle)

for i in range(10):
    self._myCar.user_control(self._default_speed, self._angle)
```

## 📢 데모 실행

---

- 아래와 같이 데모를 실행합니다.

```python
$ python3 rc_controller.py
Throttle Controller Awaked!!
Steering Controller Awaked!!
Type Throttle (Currnet Value is 0) : 380
Type Angle (Current Value is 0) : 300
====================
Type Throttle (Currnet Value is 380) : 350
Type Angle (Current Value is 300) : 400
====================
...
```

- Type Throttle (Currnet Value is 0) ⇒ 전후진 제어값 입력 후 엔터
- Type Angle (Current Value is 0) ⇒ 조향 제어값 입력 후 엔터

ㄴ 위 과정 반복

위 데모를 통해 키보드 입력으로 전후진과 조향 pulse를 조작할 수 있습니다.

하지만, 이것을 통해 영점 조절을 하지는 마시고 영점 조절 시에는 **앱 내 calibration 기능** 사용을 권장드립니다.

```python
if __name__ == "__main__":

    myCar = Vehicle("donkey_ros")
    myCar.run()
```