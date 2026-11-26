# Car_damage_segment

U-Net 세그멘테이션 기반 차량 손상 탐지 및 수리비 산출 API

<br/>

<img src="test_result_folder/img_result.jpg">

<br/><br/>

## Object

<br/>

이 프로젝트는 차량 파손 부위를 사진으로 촬영하면 **손상 종류(파손 · 찌그러짐 · 스크래치 · 이격)와 면적을 탐지하고, 면적 기준으로 예상 수리비를 산출**하는 것을 목표로 합니다.

손상 종류별로 학습된 U-Net 모델 4개를 한 번에 적용해 영역을 분리하고, 손상 면적(픽셀 수) × 종류별 단가로 수리비를 계산합니다.

결과는 Flask API로 제공하며, 가격 책정 근거를 손상 종류별 히트맵 이미지로 함께 반환해 시각적으로 확인할 수 있도록 했습니다.

<br/><br/>

## Content

<br/>

1. Module 소개

2. 프로세스 구성

3. 수리비 산출 기준

4. 실행 방법

5. Version

6. Feedback

<br/><br/>

## 1. Module 소개

<br/>

- `app.py` : 수리비 산출 API 서버 메인 모듈입니다. (Flask)

  ① 서버 시작 시 손상 종류별 U-Net 모델 4개를 로드

  ② `/estimate` (POST) : 이미지 업로드 → 4개 모델 추론 → 종류별 면적 · 가격 · 총 수리비(JSON) 반환

  ③ 종류별 히트맵을 합친 결과 이미지를 저장하고 경로를 함께 반환

- `client.py` : 이미지를 API에 전송하고 결과를 확인하는 테스트 클라이언트입니다.

- `test.py` : 단일 모델(스크래치) 추론 → 4개 모델 동시 추론 → 수리비 계산까지 단계별로 검증한 테스트 모듈입니다.

- `src/` : 모델 학습 · 평가 코드

  ① `Models.py` : `segmentation_models_pytorch` 기반 U-Net (ResNet34 인코더, ImageNet 사전학습)

  ② `Datasets.py` : COCO 형식 어노테이션을 손상 종류별 마스크로 변환

  ③ `Train.py` · `Evaluation.py` · `Utils.py` : 학습 루프, mIoU 평가, Focal Loss 등

- `main.py`, `train.py` : 학습 / 평가 실행 진입점

- `samples/damage/` : 테스트용 손상 이미지

<br/><br/>

## 2. 프로세스 구성

<br/>

```
이미지 업로드 → 256x256 리사이즈 · 정규화 → U-Net 4종 추론 (파손 · 찌그러짐 · 스크래치 · 이격)
            → 종류별 손상 마스크 · 면적 계산 → 면적 × 단가 = 수리비 → JSON + 히트맵 이미지 반환
```

<br/>

### ① 손상 종류별 마스크

<img src="test_multi_result_folder/img_result.jpg">

<br/>

### ② API 결과 히트맵

<img src="resultscombined_heatmap_20240504164530.png">

<br/>

- 하나의 이미지에서 종류별 손상 영역을 따로 분리하므로, 스크래치와 파손이 함께 있는 경우도 각각 면적을 계산할 수 있습니다.

<br/><br/>

## 3. 수리비 산출 기준

<br/>

차종 · 보험 등 실제 가격 기준이 공개되어 있지 않아, 본 모델에서는 **손상 면적(픽셀) × 손상 종류별 단가**로 계산했습니다.

| 손상 종류 | 모델 | 픽셀당 단가 (원) |
|---|---|---|
| 파손 | `[DAMAGE][Breakage_3]Unet.pt` | 100 |
| 찌그러짐 | `[DAMAGE][Crushed_2]Unet.pt` | 200 |
| 스크래치 | `[DAMAGE][Scratch_0]Unet.pt` | 50 |
| 이격 | `[DAMAGE][Seperated_1]Unet.pt` | 120 |

<br/>

응답 형식 (숫자는 형식을 보여주기 위한 예시 값)

```json
{
  "results": [
    {"label": "Breakage_3", "area": 12, "price": 1200},
    {"label": "Scratch_0", "area": 1830, "price": 91500}
  ],
  "total_price": 92700,
  "combined_heatmap": "results/combined_heatmap_<생성시간>.png"
}
```

<br/><br/>

## 4. 실행 방법

<br/>

### ① 데이터셋 · 사전 학습 모델

- 데이터셋 : [AI허브] 차량 파손 이미지 — https://www.aihub.or.kr/aihubdata/data/view.do?dataSetSn=581
- 사전 학습 모델 : AI허브 데이터로 학습된 공개 U-Net 가중치 4종을 사용했습니다. ([다운로드](https://drive.google.com/drive/folders/1q0l5vT14Kka_iu0WZgn1EFJLUbWD8EtY?usp=sharing))
  - 받은 파일은 `models/` 폴더에 넣습니다. (용량 문제로 저장소에는 포함하지 않음)

<br/>

### ② 실행

```bash
pip install -r requirments.txt

# 1. API 서버 실행 (http://127.0.0.1:5000)
python app.py

# 2. 다른 터미널에서 테스트 이미지 전송
python client.py
```

<br/><br/>

## 5. Version

<br/>

- python 3.11
- torch
- segmentation-models-pytorch
- albumentations
- opencv-python
- pandas, matplotlib, pytz
- Flask

<br/><br/>

## 6. Feedback

<br/>

- 쏘카 서비스처럼 모바일 환경에서 사진을 바로 전송하는 부분은 구현하지 않았습니다.
- 픽셀 면적은 촬영 거리에 따라 달라지므로, 실제 서비스에서는 부품 크기 기준으로 면적을 보정해야 합니다.
- 수리비 단가는 임의로 정한 값이며, 차종 · 부품 · 보험 기준 단가표가 있으면 바로 교체할 수 있도록 `app.py` 의 `models_info` 에 분리해 두었습니다.
- 결과 이미지 저장 경로가 로컬 절대경로로 지정되어 있어, 실행 전 환경에 맞게 수정해야 합니다.
