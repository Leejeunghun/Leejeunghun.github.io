---
title: "OpenCV Cuda 설치"
categories:
  - OpenCV

tags: [OpenCV]
toc : true
comments: true
date: 2023-06-01
last_modified_at: 2023-10-23
---
>>
1. [사전 준비](#사전 준비) 
2. [CUDA 와 cuDNN 설치](#cuda-와-cudnn-설치)
3. [Cmake 이용하여 OpenCV 만들기](#CUDA 와 cuDNN 설치)
4. [Visual studio 이용하여 OpenCV build 하기](#Visual studio 이용하여 OpenCV build 하기)
5. [검증하기 ](#검증하기)


10월 23일 추가

6. Nvidia Codec 추가
>>


# 사전 준비

## GPU 검사

장치 관리자 에서 확인

![GPU 스펙 보기](/assets/img/%EA%B0%9C%EB%B0%9C%ED%99%98%EA%B2%BD/OpenCV_CUDA/GPU%20%EC%82%AC%EC%96%91.png)

GPU와 알맞은 연산 능력 버젼 확인

<br/>

![위키피디아 CUDA](https://ko.wikipedia.org/wiki/CUDA)



![GPU 버젼 8.6](/assets/img/%EA%B0%9C%EB%B0%9C%ED%99%98%EA%B2%BD/OpenCV_CUDA/GPU_%EB%B2%84%EC%A0%BC.png)


## 2. OpenCV 소스 다운로드

[OpenCV 소스](https://opencv.org/releases/)

[OpenCV 확장 모듈](https://github.com/opencv/opencv_contrib/tags)

OpenCV 4.7.0 소스와 확장 모듈까지 소스 다운로드한다
압축 파일이기에 원하는 공간에 압축 풀어준다.
또 Build 파일 생성해준다.



## 3. CMAKE 파일 다운로드


[CMAKE 다운로드 링크](https://cmake.org/download/)


## 4. Visual Studio 다운로드


# CUDA 와 cuDNN 설치

[CUDA 설치 링크 ](https://developer.nvidia.com/cuda-downloads?target_os=Windows&target_arch=x86_64&target_version=11&target_type=exe_local)

![CUDA 설치 예시](/assets/img/%EA%B0%9C%EB%B0%9C%ED%99%98%EA%B2%BD/OpenCV_CUDA/CUDA%20%EC%84%A4%EC%B9%98.png)

해당 설정에 맞게 설치 진행한다

[cuDNN 다운로드 링크 ](https://developer.nvidia.com/cudnn)

cuDNN 인경우 Nvidia 회원 가입이 필요하다


![cuDNN 설치](/assets/img/%EA%B0%9C%EB%B0%9C%ED%99%98%EA%B2%BD/OpenCV_CUDA/cuDNN%20%EC%84%A4%EC%B9%98.png)

CUDA 설치후 설치 폴더 경로에 cuDNN 파일 을 덮어 넣습니다.



* 사전 세팅
 기존에 설치 되어있던 OpenCV 버젼 삭제 진행한다.
```bash
pip uninstall opencv-python
```

![삭제 확인](/assets/img//%EA%B0%9C%EB%B0%9C%ED%99%98%EA%B2%BD/OpenCV_CUDA/openCV%20%EC%82%AD%EC%A0%9C.png)

# numpy 설치 필요
```bash
pip install numpy
```

##  Cmake 이용하여 OpenCV 만들기

![환경설정](/assets/img/%EA%B0%9C%EB%B0%9C%ED%99%98%EA%B2%BD/OpenCV_CUDA/configuration.png)


필수적인 선택 사항으로

1. OPENCV_DNN_CUDA 

2. WITH_CUDA

3. ENABLE_FAST_MATH

4. OPENCV_EXTRA_MODULES_PATH (확장 모듈 경로 선택)
ex) D:\CUDA\opencv_contrib-4.7.0\opencv_contrib-4.7.0\modules

5. OPENCV_PYTHON_VERSION (Release 모드로 선택할떄)

6. INSTALL_PYTHON_EXAMPLES (선택)

7. WITH_OPENGL

8. BUILD_opencv_world //라이브러리 통합 할경우
>
 이후 Configure 이후 다시 체크해줘야하는것이 

1. WITH_CUDNN

2. WITH_CUBLAS

3. CUDA_FAST_MATH

4. CUDA_ARCH_BIN (자신에 맞는 버젼 GPU)
RTX 3060Ti = 8.6

Cmake 에서 Generate 클릭후


# Visual studio 이용하여 OpenCV build 하기

![build](/assets/img/%EA%B0%9C%EB%B0%9C%ED%99%98%EA%B2%BD/OpenCV_CUDA/build.png)


빌드 모드르 Release 변경후 ALL_BUILD 빌드 하면된다 (1시간 이상 걸림)
-> 오류 없이완료후 아래 INSTALL 빌드 실행


# 검증하기

```python 

import cv2 as cv


vod = cv.VideoCapture(0)
ret, frame = vod.read()

gpu_frame = cv.cuda_GpuMat()

# as long as the last frame was successfully read
while ret:

  # send current frame to GPU
  gpu_frame.upload(frame)

  # grab next frame with CPU
  ret, frame = vod.read()
  cv.imshow("test",frame)

```

# 결론
 C ++ 관련해서 동작확인 했지만 python 실행시 
 빌드하고 나서 
python import cv 해보니 관련 오류가 발생했습니다

 >>ImportError: DLL load failed while importing cv2: 지정된 모듈을 찾을 수 없습니다.

 물론 아무 개발 환경이 설치 되어있는 환경에서 설치에서 문제 없이 동작했으니 참고하셔서 개발 환경 구축에 성공하시기 바랍니다.
 
 혹시 이 글을 보시고 해당 문제 해결 법을 아시면 알려주시면 감사합니다.
 
 
 # 6. Nvidia Video Codec 추가

 1.  최신 Nvidia Video Codec 을 다운 받습니다. (압축풀어 줍니다)

https://developer.nvidia.com/nvidia-video-codec-sdk/download

2. 다운 받은 Codec 을 자신의 Cuda 버젼 이 있는 폴더에 확인합니다.

보통 
C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\<버젼정보>

위치에 존재합니다

3. Codec 파일에 lib 폴더에 있는 폴더는 cuda 에 lib 옮겨주고 
Codec 파일에 lib 폴더에 있는 폴더는 cuda 버젼 include 옮겨 줍니다

위에 build 과정을 진행합니다

Cmake에서 Nvidia 설정이 되었는지 확인할 수 있습니다

![Nvidia codec 설치](/assets/img/%EA%B0%9C%EB%B0%9C%ED%99%98%EA%B2%BD/OpenCV_CUDA/NvidiaCodec%EC%84%A4%EC%B9%98%ED%99%95%EC%9D%B8.png)

# 출처
1. 쿠다 버젼
https://ko.wikipedia.org/wiki/CUDA  
2. 참고 영상

https://www.youtube.com/watch?v=tjXkW0-4gME

https://www.youtube.com/watch?v=5NwU1MmmqWo
3. 한글 자료
https://prlabhotelshoe.tistory.com/24

4. Cmake 옵션
https://docs.opencv.org/4.x/db/d05/tutorial_config_reference.html