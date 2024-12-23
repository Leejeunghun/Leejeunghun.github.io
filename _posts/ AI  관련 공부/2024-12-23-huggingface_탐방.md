---
title: "HuggingFace 탐방 "
date: 2024-12-23
last_modified_at: 2024-12-23
categories:
  - HuggingFace
tags:
  - HuggingFace


published: true
toc: true
toc_sticky: true
---

# 영상생성 모델 

# 출처

https://huggingface.co/IamCreateAI/Ruyi-Mini-7B


Ruyi-Mini-7B 은 오픈소스 이미지를 비디오 영상으로 만드는 모델이다.
이미지를 넣으면 , Ruyi 가 연속적인 이미지를 만들어준다. 
Ruyi 비디오 생성에 있어 더 큰 유연성과 창의성을 제공합니다

# 설치

```
git clone https://github.com/IamCreateAI/Ruyi-Models
cd Ruyi-Models
pip install -r requirements.txt
```

# 실행 방법
두가지가 있는데 하나는 직접 실행 방법과
```
python3 predict_i2v.py
```
ComfyUI wrapper를 사용하는 방법이 있다.,



# 모델 구조 

