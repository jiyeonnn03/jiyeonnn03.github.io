---
layout: post
title:  "[KoBART-summarization] fine-tuning 및 모델 추출 방법"
subtitle:   ""
categories: envops
comments: true
---

뉴스 기사 요약 프로그램 개발 프로젝트를 진행하며 사용한 [KoBART-summarization](https://github.com/seujung/KoBART-summarization) 사용 방법을 정리합니다.

### KoBART-summarization
Requirements


### fine-tuning 모델(bin) 실행 방법
1. 로컬 어딘가에 kobart clone할 폴더 위치에서 cmd 실행하고 kobart 프로젝트를 클론합니다.

   ```git clone https://github.com/seujung/KoBART-summarization.git```


2. 추가로 학습하고자 하는 데이터를 train/test로 나눈 후 tsv 형태로 저장하여 data 폴더 안에 넣습니다.


3. KoBART-summarization git README를 바탕으로 학습을 진행합니다.
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

4. 학습이 끝나면 logs 폴더 내에 모델 파라미터에 대한 정보가 저장된 로그 파일이 생성됩니다.
   ![kobart_파일_구조](https://user-images.githubusercontent.com/49242144/163422235-33ff7b77-c1b8-46be-a7c2-321eb7832fb2.PNG)


5. 생성된 로그 파일을 바탕으로 모델을 추출하기 위한 코드는 아래와 같습니다.

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


6. 성공적으로 bin 파일을 얻었다면 다음을 실행해 결과를 확인할 수 있습니다.
   ```
   streamlit run infer.py
   ```
   코드 실행 시 http://localhost:8501/ 에 데모 페이지가 생성됩니다.


- infer.py 실행 시 발생 오류
    1) KoBART 미설치
       pip install git+https://github.com/SKT-AI/KoBART#egg=kobart
    
    2) KoBart 설치 중 egg_info 오류
       [이 포스트](https://ychcom.tistory.com/entry/%ED%8C%8C%EC%9D%B4%EC%8D%AC-setuppy-egginfo%E2%80%9D-failed-with-error-code-1) 참고해서 코드 실행 후 다시 KoBART 설치