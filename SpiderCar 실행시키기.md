# SpiderCar 실행시키기

SpiderCar를 실행시키기 앞서, 다음과 같이 하드웨어 연결들을 확인합니다. 

- 베터리 연결 확인
- 모든 하드웨어 연결 확인
- 카메라 연결 확인
- LTE, wifi 등 인터넷 연결 확인
- 기타 센서류 등 하드웨어 연결 확인

<aside>
💡 HDMI를 연결하여 쭉 확인 해보시는 것을 추천드립니다.

</aside>

# 🏃 SpiderCar 수동 실행

---

> SpiderCar를 수동으로 실행하기 위한 방법들을 소개합니다.
> 

가장 주축이 되는 코드는 [SpiderDonkey_Async.py](https://github.com/teamgrit-lab/hello-mars-spidercar/blob/master/SpiderDonkey_Async.py) 입니다. 따라서 다음과 같이 파이썬 파일을 실행하면 구동이 시작됩니다.

```bash
$ cd hello-mars-spidercar
$ python3 [SpiderDonkey_Async.py](https://github.com/teamgrit-lab/hello-mars-spidercar/blob/master/SpiderDonkey_Async.py)
# PeerConnection이 이루어졌는지 (success) 확인!!
```

하지만, 카메라 관련 설정이 꼬이는 경우가 종종 있으며, 이에 따라 다음과 같이 [run.sh](http://run.sh) 스크립트를 별도로 만들었습니다.

- [run.sh](http://run.sh)

```bash
sudo systemctl restart nvargus-daemon.service
python3 SpiderDonkey_Async.py
```

결국, 최우선으로 추천하는 방식은 [run.sh](http://run.sh)를 실행하는 것입니다.

```bash
$ ./run.sh
```