# rc_controller.py

파일명: rc_controller.py

설명:

두 종류의 Python 클래스(Class)가 있습니다.

1. Class PCA9685
    1. PWM컨트롤러보드 PCA9685를 제어하기 위한 클래스입니다.
    2. PCA9685는 RC Car의 전자변속기(ESC, Electronic Speed Controller)에 PWM신호를 제공하여 RC Car의 속도를 제어합니다.
2. Class Vehicle
    1. PCA9685 클래스를 사용하여 RC Car의 조향(Steering)과 구동의 속도 등을 제어합니다.