# gstreamer-install

목표 : raspberry pi 4에서 jetson xavier nx보드로 gstreamer를 사용해 영상을 송/수신

문제점과 해결방안 : SDK Manager를 통해 설치된 opencv로는 gstreamer 사용이 불가한 문제가 생겨 opencv를 공식 저장소에서 직접 빌드하여 사용하려고함



# 환경

- 보드 : jetson xavier nx

- OS : Ubuntu-20.04 focal

- jetpack version : 5.1.5

- python : 3.8

- cuda : 11.4.315

- cudnn : 8.6.0.166

- gstreamer : 1.16.3

* 설치한 것

ultralytics

opencv : 4.5.4




# 설치 전 확인할 것

1) 설치된 python버전이 여러개 있는지 확인인 (path 지정만 잘 해주면 문제없지만, 문제가 생겼을 때 찾기 힘듦)

-> 문제가 발생하지 않도록 설치 전에 python 버전, 경로, 링크 등 확인 필수

2) python path 확인
```
which python
# 결과 /usr/bin/면 성공)
```
* 핵심은 python, opencv, gstreamer의 버전과 설치하려는 opencv버전이 gstreamer를 지원하는지임

# 설치 방법

1) 작업공간 생성
```
mkdir workspace && cd ~/workspace/
```

2) 가상환경 생성, 활성화
```
python -m venv venv
. venv/bin/activate
```

3-1) ultralytics 설치
```
pip install ultralytics
```
3-2) opencv-python제거 (ultralytics를 설치하며 함께 설치된 opencv를 제거하는 것임)
```
pip uninstall opencv-python
# 이 명령으로 제거되지만 잔여 파일이 남아있는지 확인하는 것이 확실함 / 아래 명령어로 확실하게 제거
# rm -rf venv/lib/python3.8/site-packages/cv2
# rm -rf ~/workspace/venv/lib/python3.8/site-packages/opencv_python-x.xx.x.x.dist-info
# rm -rf venv/lib/python3.8/site-packages/opencv_python.libs
```

4-1) opencv 공식 github에서 소스 다운로드
```
wget -O opencv.zip https://github.com/opencv/opencv/archive/4.5.4.zip
wget -O opencv_contrib.zip https://github.com/opencv/opencv_contrib/archive/4.5.4.zip
unzip opencv.zip
unzip opencv_contrib.zip
mv opencv-4.5.4 opencv
mv opencv_contrib-4.5.4 opencv_contrib
rm opencv.zip opencv_contrib.zip
```

4-2) opencv/ 아래에 build 디렉토리 생성
```
cd opencv
mkdir build && cd build
```
5) cmake 파일 빌드
```
# PYTHON_PACKAGES_PATH 지정을 정확히 해줘야함
# 아래 명령어 실행 시 path가 가상환경 venv의 site-packages로 되어있어야함

python -c "from sysconfig import get_paths as gp; print(gp()['purelib'])"
# ex) /home/username/workspace/venv/lib/python3.8/site-packages
```

경로가 제대로 출력되면 빌드 수행
```
PYTHON_EXECUTABLE=$(which python)
PYTHON_INCLUDE_DIR=$(python -c "from sysconfig import get_paths as gp; print(gp()['include'])")
PYTHON_LIBRARY=$(python -c "import sysconfig; print(sysconfig.get_config_var('LIBDIR'))")/libpython$(python -c "import sys; print(f'{sys.version_info.major}.{sys.version_info.minor}')").so
PYTHON_PACKAGES_PATH=$(python -c "from sysconfig import get_paths as gp; print(gp()['purelib'])")
PYTHON_NUMPY_INCLUDE_DIR=$(python -c "import numpy; print(numpy.get_include())")

# cmake 실행
cmake -D CMAKE_BUILD_TYPE=RELEASE \
  -D CMAKE_INSTALL_PREFIX=~/linetracer_ws/opencv_install \
  -D OPENCV_EXTRA_MODULES_PATH=~/linetracer_ws/opencv_contrib/modules \
  -D WITH_CUDA=ON \
  -D WITH_GSTREAMER=ON \
  -D BUILD_opencv_python3=ON \
  -D BUILD_opencv_python2=OFF \
  -D PYTHON3_EXECUTABLE=${PYTHON_EXECUTABLE} \
  -D PYTHON3_INCLUDE_DIR=${PYTHON_INCLUDE_DIR} \
  -D PYTHON3_LIBRARY=${PYTHON_LIBRARY} \
  -D PYTHON3_PACKAGES_PATH=${PYTHON_PACKAGES_PATH} \
  -D PYTHON3_NUMPY_INCLUDE_DIRS=${PYTHON_NUMPY_INCLUDE_DIR} \
  ..
```
6) 컴파일
```
make -j$(nproc)
# 발열이 심할 수 있으니 make -j2 사용 권장함
```
7) 설치
```
make install
```

=> 설치까지 하면 ~/workspace/opencv/build/lib/python3/
~/workspace/venv/lib/python3.8/site-packages/cv2/python-3.8/  두 경로에 .so파일이 생성됨

- opencv 설치 확인 (opencv 4.5.4버전이 출력되어야함)
```
import cv2
print(cv2.__version__)
```
![image](https://github.com/user-attachments/assets/bb6fb114-1679-4129-a0b2-c7f463019c76)

- gstreamer를 사용할 수 있는지 확인 (YES가 출력되어야함)
```
import cv2
print(cv2.getBuildInformation())
```
![image](https://github.com/user-attachments/assets/93792ddd-8664-47a4-bef2-eac0983577b0)



# 실행

## raspberry pi 4 (명령어)

v4l2-ctl --list-formats-ext -d /dev/video0 명령어로 카메라의 지원 포맷/해상도 확인후 명령어 수정

csi카메라는 13개의 포맷 지원, 해상도는 1가지로 통일되어있음

![image](https://github.com/user-attachments/assets/b8698143-ce91-4704-8d6c-6081989cbe33)

usb카메라는 1개의 포맷 지원, 3가지 해상도 가능

![image](https://github.com/user-attachments/assets/f93dd52d-c1ae-4661-8c61-eb87d0594c12)


```
# csi카메라
# v4l2-ctl --list-formats-ext -d /dev/video0
gst-launch-1.0 v4l2src device=/dev/video0 ! \video/x-raw,width=640,height=480,framerate=30/1 ! \videoconvert ! \x264enc tune=zerolatency bitrate=4000 speed-preset=superfast ! \rtph264pay config-interval=1 pt=96 ! \udpsink host=192.168.0.xxx port=5000 sync=false
```
```
# usb카메라
# v4l2-ctl --list-formats-ext -d /dev/video1 
gst-launch-1.0 v4l2src device=/dev/video1 ! image/jpeg,width=960,height=540,framerate=30/1 ! jpegdec ! videoconvert ! x264enc tune=zerolatency bitrate=4000 speed-preset=superfast ! rtph264pay config-interval=1 pt=96 ! udpsink host=192.168.0.xxx port=5000 sync=false
```

## jetson xavier nx (python 테스트 코드)
```
# 테스트용 명령어
gst-launch-1.0 udpsrc port=5000 caps="application/x-rtp, media=video, encoding-name=H264, payload=96" ! rtph264depay ! h264parse ! nvv4l2decoder ! nvvidconv ! videoconvert ! autovideosink
```
```
import cv2

def main():
    gst_str = (
        'udpsrc port=5000 caps="application/x-rtp, media=video, encoding-name=H264, payload=96" ! '
        'rtph264depay ! h264parse ! nvv4l2decoder ! '
        'nvvidconv ! video/x-raw, format=BGRx ! '
        'videoconvert ! video/x-raw, format=BGR ! '
        'appsink'
    )
    cap = cv2.VideoCapture(gst_str, cv2.CAP_GSTREAMER)
    if not cap.isOpened():
        print("pipeline open failed")
        return

    while True:
        ret, frame = cap.read()
        if not ret:
            print("Failed to receive frame")
            break

        cv2.imshow("Jetson Video Stream", frame)
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break

    cap.release()
    cv2.destroyAllWindows()

if __name__ == "__main__":
    main()
```


# 설치 중 발생한 문제점들 (Q & A)

Q1. nx환경에서 gstreamer가 명령어로는 열리지만 python코드로 열지 못함

A. pip으로 설치한 opencv-python은 gstreamer를 지원하지 않아, opencv를 직접 빌드해서 사용해야됨 (SDK Manager로 설치한 opencv도 포함)

-> opencv를 특정 작업공간에 설치해서 사용했기 때문에 jtop명령어로 opencv확인시 라이브러리를 찾지 못해도 실행에 문제는 없음

-----

Q2. ultralytics 패키지가 필요해 설치할 경우 opencv를 자동으로 설치해주기 때문에 다시 문제가 생김

A. ultralytics 패키지를 설치한 후 opencv 관련 파일을 모두 제거한 후 opencv를 직접 빌드함

-----

Q3. nx환경에서 python 여러가지 버전을 사용하기 위해 pyenv를 설치하였는데, 이로 인해 빌드옵션으로 python버전을 3.8로 명시해주어도 pyenv가 우선순위로 지정하고있는 python버전으로 openv를 빌드하는 문제가 있었음.

-> which python 명령어로 확인해보면 /usr/bin/python이 아닌 .pyenv를 통한 경로가 나오면 이후에 opencv 빌드시에 원하는 python버전으로 설치가 불가능했음.

A. pyenv환경을 off 하였음
```
pyenv global system
hash -r
```





