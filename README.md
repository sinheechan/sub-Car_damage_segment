# Car_damage_segment

U-Net 세그멘테이션으로 차량 손상 부위를 찾고 수리비를 계산하는 API

<br/>

<img src="test_result_folder/img_result.jpg">

<br/><br/>

## Object

<br/>

- 차량이 파손됐을 때 사진 한 장만 찍으면 손상 종류와 면적을 찾아서 수리비를 대략 계산해 주는 모델입니다.

- 손상 종류는 파손, 찌그러짐, 스크래치, 이격 4가지이고, 종류마다 따로 학습된 U-Net 모델 4개를 한 번에 돌려서 각각의 손상 영역을 구합니다.

- 수리비를 어떻게 계산했는지 알 수 있도록 손상 종류별 히트맵 이미지도 같이 보여줍니다.

- 서비스로 만든다면 사진만 올려서 예상 수리비를 바로 확인할 수 있을 것 같아 Flask API 형태로 만들었습니다.

<br/><br/>

## Content

<br/>

1. Module 소개

2. 진행 과정

3. 수리비 계산 방법

4. 실행 방법

5. Version

6. Feedback

<br/><br/>

## 1. Module 소개

<br/>

- `app.py` : 수리비 계산 API 서버입니다. (Flask)

  ① 서버가 켜질 때 손상 종류별 U-Net 모델 4개를 불러옵니다.

  ② `/estimate` 로 이미지를 POST 하면 4개 모델로 예측해서 종류별 면적, 가격, 총 수리비를 JSON으로 돌려줍니다.

  ③ 종류별 히트맵을 하나로 합친 이미지를 저장하고 그 경로도 같이 돌려줍니다.

- `client.py` : 이미지를 API로 보내서 결과를 확인하는 테스트용 클라이언트입니다.

- `test.py` : API 만들기 전에 스크래치 모델 하나로 먼저 테스트하고, 그다음 4개 모델을 같이 돌려 수리비까지 계산해 본 코드입니다.

- `src/` : 학습·평가 코드입니다. `Models.py` 는 segmentation_models_pytorch 의 U-Net(ResNet34 인코더)을 쓰고, `Datasets.py` 는 COCO 형식 라벨을 손상 종류별 마스크로 바꿉니다.

- `samples/damage/` : 테스트에 쓴 손상 차량 사진

<br/><br/>

## 2. 진행 과정

<br/>

### ① 스크래치 모델 하나로 테스트

<br/>

처음에는 스크래치 모델만 불러와서 이미지를 256x256으로 줄이고 예측해 봤습니다. 맨 위 이미지가 그 결과로, 왼쪽이 원본이고 오른쪽이 스크래치로 잡힌 영역입니다.

<br/>

### ② 4개 모델 같이 돌리기

<br/>

<img src="test_multi_result_folder/img_result.jpg">

<br/>

같은 사진에 4개 모델을 모두 돌리면 종류별로 손상 영역이 따로 나옵니다. 위 사진은 범퍼 쪽 긁힘이 스크래치로 크게 잡혔고 파손은 아주 작게 잡혔습니다. 한 사진에 손상이 여러 종류 섞여 있어도 각각 면적을 계산할 수 있습니다.

<br/>

### ③ API로 만들기

<br/>

<img src="resultscombined_heatmap_20240504164530.png">

<br/>

테스트 코드를 Flask API로 옮겼습니다. 사진을 보내면 원본과 손상 종류별 히트맵을 합친 이미지를 만들어 저장하고, 수리비 결과와 함께 돌려줍니다.

<br/><br/>

## 3. 수리비 계산 방법

<br/>

차종, 보험 등 실제 수리비 기준을 구할 수가 없어서 **손상 면적(픽셀 수) x 손상 종류별 단가** 로 계산했습니다. 단가는 임의로 정한 값입니다.

| 손상 종류 | 모델 파일 | 픽셀당 단가 (원) |
|---|---|---|
| 파손 | `[DAMAGE][Breakage_3]Unet.pt` | 100 |
| 찌그러짐 | `[DAMAGE][Crushed_2]Unet.pt` | 200 |
| 스크래치 | `[DAMAGE][Scratch_0]Unet.pt` | 50 |
| 이격 | `[DAMAGE][Seperated_1]Unet.pt` | 120 |

<br/>

API 응답은 아래 형태입니다. (숫자는 예시)

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

<br/>

- 데이터셋 : [AI허브] 차량 파손 이미지 - https://www.aihub.or.kr/aihubdata/data/view.do?dataSetSn=581

- 사전 학습 모델 : AI허브 데이터로 학습된 공개 U-Net 가중치 4개를 받아서 썼습니다. ([다운로드](https://drive.google.com/drive/folders/1q0l5vT14Kka_iu0WZgn1EFJLUbWD8EtY?usp=sharing))

  용량이 커서 저장소에는 넣지 않았고, 받은 파일을 `models/` 폴더에 넣으면 됩니다.

<br/>

### ② 실행

<br/>

```bash
pip install -r requirments.txt

python app.py       # API 서버 (http://127.0.0.1:5000)
python client.py    # 다른 터미널에서 테스트 이미지 전송
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

- 쏘카 서비스처럼 모바일에서 사진을 바로 보내는 부분은 만들지 못했습니다.

- 픽셀 면적은 사진을 얼마나 가까이서 찍었는지에 따라 달라지기 때문에, 실제로 쓰려면 부품 크기 같은 기준으로 면적을 보정해야 합니다.

- 단가는 `app.py` 의 `models_info` 에 따로 빼 두어서, 실제 단가표가 생기면 거기만 바꾸면 됩니다.

- 결과 이미지 저장 경로가 제 PC 기준 절대경로라서 다른 환경에서는 바꿔야 합니다.
