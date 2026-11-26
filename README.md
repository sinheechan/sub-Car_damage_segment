# Car_damage_segment

U-Net 세그멘테이션 기반 차량 손상 탐지 · 수리비 산출 API

<br/>

<img src="test_result_folder/img_result.jpg">

<br/><br/>

## Object

<br/>

- 차량 사진 한 장으로 손상 종류·면적 탐지 후 수리비 산출
- 손상 종류 : 파손, 찌그러짐, 스크래치, 이격
- 종류별 U-Net 모델 4개를 한 번에 적용
- 산출 근거를 종류별 히트맵 이미지로 함께 제공
- 서비스화를 가정해 Flask API로 구현

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

- `app.py` : 수리비 산출 API 서버 (Flask)
  - 서버 시작 시 U-Net 모델 4개 로드
  - `/estimate` POST → 종류별 면적·가격·총 수리비 반환
  - 종류별 히트맵을 합친 이미지 저장 후 경로 반환
- `client.py` : API 테스트용 이미지 전송 클라이언트
- `test.py` : 단일 모델 → 4개 모델 → 수리비 계산 단계별 검증
- `src/Models.py` : U-Net (ResNet34 인코더)
- `src/Datasets.py` : COCO 라벨 → 손상 종류별 마스크 변환
- `samples/damage/` : 테스트용 손상 차량 사진

<br/><br/>

## 2. 진행 과정

<br/>

### ① 스크래치 모델 단독 테스트

<br/>

- 이미지를 256x256으로 줄여 스크래치 모델로 예측
- 맨 위 이미지 : 왼쪽 원본, 오른쪽 스크래치 영역

<br/>

### ② 4개 모델 동시 적용

<br/>

<img src="test_multi_result_folder/img_result.jpg">

<br/>

- 같은 사진에서 손상 종류별 영역을 각각 분리
- 위 사진 : 범퍼 긁힘은 스크래치로 크게, 파손은 작게 탐지
- 여러 손상이 섞여 있어도 종류별 면적 계산 가능

<br/>

### ③ API 구현

<br/>

<img src="resultscombined_heatmap_20240504164530.png">

<br/>

- 테스트 코드를 Flask API로 이전
- 원본과 종류별 히트맵을 합친 이미지 생성
- 수리비 결과와 이미지 경로를 함께 반환

<br/><br/>

## 3. 수리비 계산 방법

<br/>

- 계산식 : 손상 면적(픽셀 수) x 손상 종류별 단가
- 차종·보험 기준을 구할 수 없어 단가는 임의 설정

<br/>

| 손상 종류 | 모델 파일 | 픽셀당 단가 (원) |
|---|---|---|
| 파손 | `[DAMAGE][Breakage_3]Unet.pt` | 100 |
| 찌그러짐 | `[DAMAGE][Crushed_2]Unet.pt` | 200 |
| 스크래치 | `[DAMAGE][Scratch_0]Unet.pt` | 50 |
| 이격 | `[DAMAGE][Seperated_1]Unet.pt` | 120 |

<br/>

API 응답 형식 (숫자는 예시)

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

- 데이터셋 : [AI허브 차량 파손 이미지](https://www.aihub.or.kr/aihubdata/data/view.do?dataSetSn=581)
- 사전 학습 모델 : AI허브 데이터 기반 공개 U-Net 가중치 4종
- 다운로드 : [Google Drive](https://drive.google.com/drive/folders/1q0l5vT14Kka_iu0WZgn1EFJLUbWD8EtY?usp=sharing)
- 용량 문제로 저장소 미포함, `models/` 폴더에 저장

<br/>

### ② 실행

<br/>

```bash
pip install -r requirments.txt
python app.py       # API 서버 (http://127.0.0.1:5000)
python client.py    # 테스트 이미지 전송
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

- 모바일 사진 전송 기능은 미구현
- 촬영 거리에 따라 픽셀 면적이 달라져 보정 기준 필요
- 단가는 `app.py` 의 `models_info` 에 분리, 교체 용이
- 결과 저장 경로가 로컬 절대경로라 환경별 수정 필요
