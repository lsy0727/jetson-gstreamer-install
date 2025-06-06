# gstreamer-install

jetson xavier nx보드에 gstreamer를 설치해 영상을 받아오려고함

SDK Manager를 통해 opencv 설치시 gstreamer 사용이 불가한 문제가 생겨 opencv를 직접 빌드하여 사용함

# 환경

- 보드 : jetson xavier nx

- OS : Ubuntu-20.04 focal

- jetpack version : 5.1.5

- python : 3.8

- cuda : 11.4.315

- cudnn : 8.6.0.166



# 설치 방법

* python 버전이 여러가지 설치되어있으면 이후에 버전 충돌이 발생하는 문제가 있었음.

-> 문제가 발생하지 않도록 설치 전에 python 버전, 경로, 링크 등 확인 필수!! (나중에 문제 발견시 해결하기 힘들다)

1) 작업공간으로 이동
```
$ cd workspace/
```
2) opencv 공식 깃허브에서 4.5.4버전 소스 다운로드 (현재 디렉토리에 opencv, opencv_contrip이 설치되면 성공)
```
$ wget -O opencv.zip https://github.com/opencv/opencv/archive/4.5.4.zip

$ wget -O opencv_contrib.zip https://github.com/opencv/opencv_contrib/archive/4.5.4.zip

$ unzip opencv.zip

$ unzip opencv_contrib.zip

$ mv opencv-4.5.4 opencv

$ mv opencv_contrib-4.5.4 opencv_contrib

$ rm opencv.zip opencv_contrib.zip
```
3-1) 가상환경 생성, 활성화
```
$ python -m venv venv

$ . venv/bin/activate
```
3-2) opencv/ 아래에 build 디렉토리 생성
```
$ cd opencv

$ mkdir build && cd build
```
4) cmake 파일 빌드

* PYTHON_PACKAGES_PATH 지정을 정확히 해줘야함
아래 명령어 실행 시 path가 가상환경 venv의 site-packages로 되어있어야함
```
python -c "from sysconfig import get_paths as gp; print(gp()['purelib'])"
```
ex) /home/username/workspace/venv/lib/python3.8/site-packages

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
5) 컴파일
```
make -j$(nproc)
# 발열이 심할 수 있으니 make -j2 사용 권장함
```
6) 설치
```
make install
```

=> 설치까지 하면 ~/workspace/venv/lib/python3.8/site-packages/cv2/python-3.8 아래에 .so파일이 생성됨

- opencv 설치 확인 (opencv 4.5.4버전이 출력되어야함)
```
import cv2
print(cv2.__version__)
```
- gstreamer를 사용할 수 있는지 확인 (YES가 출력되어야함)
```
import cv2
print(cv2.getBuildInformation())
```







