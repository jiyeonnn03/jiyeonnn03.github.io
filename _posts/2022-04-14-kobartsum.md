---
layout: post
title:  "[KoBART-summarization] fine-tuning 및 모델 추출 방법"
subtitle:   ""
categories: envops
comments: true
---

뉴스 기사 요약 프로그램 개발 프로젝트를 진행하며 사용한 [KoBART-summarization](https://github.com/seujung/KoBART-summarization) 사용 방법을 정리합니다.

### KoBART-summarization
[KoBART 문서](https://github.com/SKT-AI/KoBART) 에 따르면, KoBART는 BART 논문에서 사용된 Text Infilling 노이즈 함수를 사용하여 40GB 이상의 한국어 텍스트에 대해서 학습한 한국어 encoder-decoder 언어 모델이라고 합니다.

이 포스트에서는, 현재 진행하고 있는 프로젝트에서 요약 모델을 fine-tuning해 사용하는 과정에서 발생한 문제에 대해 간단히 정리하려 합니다.

### [ fine-tuning 방법 ]
1. 로컬 어딘가에 kobart clone할 폴더 위치에서 cmd 실행하고 kobart 프로젝트를 클론합니다.

   ```git clone https://github.com/seujung/KoBART-summarization.git```


2. 추가로 학습하고자 하는 데이터를 train/test로 나눈 후 tsv 형태로 저장하여 data 폴더 안에 넣습니다.

- csv -> tsv를 변환하는 편한(?) 방법은..   
    excel로 csv 파일을 열어서 [다른 이름으로 저장]할 때 **텍스트(탭으로 분리)(\*.txt)** 로 저장한 후,
    확장자명을 tsv로 수정해 저장하는 것인 것 같습니다.

3. KoBART-summarization git README를 바탕으로 fine-tuning을 진행합니다.
   ```
   pip install -r requirements.txt
   ```
    
    - GPU 사용 시
      
      ```
      python train.py  --gradient_clip_val 1.0 --max_epochs 50 --default_root_dir logs --gpus 1 --batch_size 4 --num_workers 4
      ```
      또는
       ```
      python train.py  --gradient_clip_val 1.0 --max_epochs 50 --default_root_dir logs --strategy ddp --gpus 2 --batch_size 4 --num_workers 4
      ```
    - CPU 사용 시
       ```
       python train.py  --gradient_clip_val 1.0 --max_epochs 50 --default_root_dir logs --strategy ddp --batch_size 4 --num_workers 4
      ```
    - train.py 실행 중 발생하는 문제가 몇가지 있었는데, GPU 관련 문제는 CUDA 드라이버 및 CUDNN, tensorflow 등을 버전에 맞게 설치하니 해결되었습니다.
        - KoBART-sum github에서 권장하는 CUDA 버전은 10.2입니다. ([KoBART-summarization issue20](https://github.com/seujung/KoBART-summarization/issues/20))
    - RuntimeError: ~ 'indices' 문제는 [KoBART-summarization issue7](https://github.com/seujung/KoBART-summarization/issues/7) 을 참고하세요.
    - 아무런 문제 없이 학습이 시작되었다면, 다음으로 확인해야 할 사항은 데이터 형식이 원본 데이터(기존 tsv)와 일치하는지 입니다.
      제 경우에는 na.drop을 했음에도 비어있는 값이 있어서 오류가 발생했습니다.
      tsv 변환 전, 학습하고자 하는 데이터에 빈 열이 존재하지는 않는지 미리 확인하시면 학습이 중단되는 것을 방지할 수 있습니다.
    

### [ 모델 추출 방법 ]
1. 학습이 끝나면 logs 폴더 내에 모델 파라미터에 대한 정보가 저장된 로그 파일이 생성됩니다.
   ![image](https://user-images.githubusercontent.com/49242144/163433230-c3347e5c-9325-4c86-8769-81fce7b1daab.png)


2. 생성된 로그 파일을 바탕으로 모델을 추출하기 위한 코드는 아래와 같습니다. KoBART-sum 위치에서 아래 코드를 실행하면 됩니다.

   ```
   python get_model_binary.py --hparams hparam_path --model_binary model_binary_path
   ```
   이 때,
    - hparam : ./logs/tb_logs/default/version_0/hparams.yaml 활용
    - model_binary: ./logs/kobart_summary-model_chp/epoch=02-val_loss=1.726.ckpt 활용   
      하므로 아래와 같이 각 path에 폴더의 경로를 넣어줍니다.
   ```
   python get_model_binary.py --hparams ./logs/tb_logs/default/version_0/hparams.yaml --model_binary ./logs/kobart_summary-model_chp/epoch=02-val_loss=1.726.ckpt
   ```
   오류가 발생한다면 .logs 앞에 파일의 절대 경로를 넣어보세요 :)


3. 위 코드가 정상적으로 동작했다면 아래와 같이 kobart_summary 폴더 내에 config.json, pytorch_model.bin 이 생성됩니다.
   ![kobart_파일_구조](https://user-images.githubusercontent.com/49242144/163422235-33ff7b77-c1b8-46be-a7c2-321eb7832fb2.PNG)
   ![image](https://user-images.githubusercontent.com/49242144/163433603-f0b3cbf1-deee-4c36-88fd-71f5645591f7.png)


4. 성공적으로 bin 파일을 얻었다면 다음을 실행해 새롭게 얻은 모델을 실행할 수 있습니다.
   ```
   streamlit run infer.py
   ```
   코드 실행 시 http://localhost:8501/ 에 데모 페이지가 생성됩니다.


- infer.py 실행 시 발생 오류
    1) KoBART 미설치
       ```
       pip install git+https://github.com/SKT-AI/KoBART#egg=kobart
       ```
       
    2) KoBART 설치 중 egg_info 오류
       ```
       # setuptools 업그레이드
       pip install --upgrage setuptools
       
       # ez_setup 설치
       pip install ez_setup
       
       # 설치 재시도
       pip install unroll
       easy_install -U setuptools
       pip install unroll
       ```   



{%- if site.disqus.shortname -%}
{%- include disqus.html -%}
{%- endif -%}