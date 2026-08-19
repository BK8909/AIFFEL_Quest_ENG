# 모델 배포 개론 - Day 5 프로젝트 1 체크포인트 답변

**주제:** California Housing 정형 데이터 기반 주택 가격 예측 서비스  
**구성:** PyTorch 회귀 모델 → FastAPI 추론 API → Streamlit 입력 폼 → 통합 테스트

---

## 1. 프로젝트 개요 체크포인트

### 1. 이 프로젝트의 입력과 출력은 각각 무엇입니까?

- **입력:** California Housing 데이터셋의 숫자형 피처 8개  
  `MedInc`, `HouseAge`, `AveRooms`, `AveBedrms`, `Population`, `AveOccup`, `Latitude`, `Longitude`
- **출력:** 중위 주택 가격 예측값. 모델의 기본 단위는 **$100,000 단위**이며 서비스에서는 USD 금액으로도 변환합니다.
- **문제 유형:** **회귀(Regression)**

### 2. MNIST 프로젝트와 비교했을 때 데이터 형태가 어떻게 다릅니까?

MNIST는 28×28 이미지의 픽셀값을 입력으로 받아 0~9 중 하나의 클래스를 예측하는 **이미지 분류** 문제입니다.  
이번 프로젝트는 8개의 숫자형 주택 정보를 입력으로 받아 연속적인 주택 가격을 예측하는 **정형 데이터 회귀** 문제입니다.

따라서 입력 전처리, Pydantic 스키마, Streamlit UI, 모델의 최종 출력 형태가 모두 달라집니다.

### 3. 오늘 새로 만들 파일 3개의 이름과 역할은 무엇입니까?

1. `app/housing_model.py`
   - `HousingModel` 모델 구조 정의
   - 저장된 모델과 전처리 파라미터 로드
   - `HousingPredictor`에서 정규화와 추론 수행

2. `app/housing_schemas.py`
   - `HousingRequest`, `HousingResponse` Pydantic 스키마 정의
   - 입력 타입과 도메인 범위 검증

3. `app/housing_api.py`
   - FastAPI 앱 생성
   - 모델 로드
   - `/health`, `/predict` 엔드포인트 제공
   - 스레드풀을 이용한 비동기 추론 처리

---

## 2. 모델 준비 체크포인트

### 1. 정규화에서 학습 데이터의 통계를 테스트 데이터에도 사용하는 이유는 무엇입니까?

테스트 데이터는 실제 배포 환경에서 처음 들어오는 새로운 데이터를 모사합니다. 테스트 데이터 자체의 평균과 표준편차를 사용하면 테스트 데이터의 정보를 미리 이용하는 **데이터 누수(Data Leakage)** 가 발생합니다.

따라서 학습 데이터에서 계산한 `mean`, `std`를 학습·테스트·실서비스 입력 모두에 동일하게 사용해야 합니다.

### 2. 모델 가중치 외에 함께 저장해야 하는 것은 무엇이고, 왜 필요합니까?

`mean`, `std`, `feature_names`가 포함된 전처리 파라미터를 함께 저장해야 합니다.

- `mean`: 학습 데이터의 피처별 평균
- `std`: 학습 데이터의 피처별 표준편차
- `feature_names`: 모델이 학습할 때 사용한 피처 순서

배포 환경에서도 학습 때와 동일한 정규화와 피처 순서를 재현하기 위해 필요합니다.

### 3. `HousingPredictor.predict()`에서 피처를 `self.feature_names` 순서로 배열하는 이유는 무엇입니까?

모델은 피처 이름이 아니라 정해진 위치의 숫자를 입력으로 받습니다. 학습 때와 추론 때 피처 순서가 달라지면 각 값이 다른 변수로 해석되어 잘못된 예측이 발생할 수 있으므로 동일한 순서를 유지해야 합니다.

---

## 3. FastAPI 백엔드 체크포인트

### 1. `HousingRequest`에서 `Latitude`에 `ge=32, le=42` 제한을 넣은 이유는 무엇입니까?

California Housing 데이터의 도메인 범위를 벗어나는 위도 값을 모델에 전달하기 전에 차단하기 위해서입니다. 범위를 벗어난 입력은 Pydantic 검증 단계에서 4xx 응답으로 거부됩니다.

### 2. `request.model_dump()`는 어떤 역할을 합니까?

Pydantic의 `HousingRequest` 객체를 일반 Python `dict`로 변환합니다. 변환된 딕셔너리는 `HousingPredictor.predict(features)`의 입력으로 전달됩니다.

### 3. `run_in_executor`를 사용하지 않으면 어떤 문제가 발생할 수 있습니까?

모델 추론은 동기식 CPU 연산이므로 이벤트 루프에서 직접 실행하면 추론 중 다른 요청 처리가 지연될 수 있습니다. `run_in_executor`를 사용하면 추론을 별도 스레드에서 실행하여 FastAPI 이벤트 루프의 블로킹을 줄일 수 있습니다.

---

## 4. Streamlit 프론트엔드 체크포인트

### 1. MNIST 대시보드와 비교했을 때 입력 방식이 어떻게 다릅니까?

- **MNIST:** 이미지 파일 업로드 또는 샘플 이미지 선택
- **California Housing:** 8개의 숫자형 피처를 `st.number_input()`으로 직접 입력

이미지 중심의 비정형 입력에서 명시적인 필드 이름과 숫자 값을 갖는 정형 데이터 입력 폼으로 바뀌었습니다.

### 2. `st.number_input()`에서 `min_value`, `max_value`를 설정하는 이유는 무엇입니까?

사용자가 UI 단계에서 비현실적이거나 허용 범위를 벗어난 값을 입력하지 못하도록 제한하기 위해서입니다. 다만 API는 다른 클라이언트에서도 직접 호출될 수 있으므로 백엔드의 Pydantic 검증도 함께 필요합니다.

### 3. `request_data`의 키 이름이 `HousingRequest`의 필드 이름과 정확히 일치해야 하는 이유는 무엇입니까?

FastAPI는 JSON의 키를 Pydantic 스키마의 필드명과 매칭해 검증합니다. 또한 백엔드에서는 같은 피처 이름을 기준으로 학습 때의 순서에 맞춰 배열을 만들기 때문에 프론트엔드 → 스키마 → 모델 사이에서 이름이 일관되어야 합니다.

---

## 5. Day 5 최종 체크포인트

### Q1. 전처리 파라미터(mean, std)를 모델과 함께 저장해야 하는 이유는?

학습 때 적용한 것과 동일한 정규화를 배포 환경에서도 재현하기 위해서입니다. 평균과 표준편차가 달라지면 같은 원본 입력도 모델에는 다른 값으로 전달되어 예측이 달라질 수 있습니다.

### Q2. `HousingRequest`에서 `Latitude`에 `ge=32, le=42`를 넣은 이유는?

캘리포니아 데이터의 도메인 범위를 벗어나는 위도 값을 API 입구에서 차단하기 위해서입니다.

### Q3. Streamlit의 입력값 이름이 Pydantic 스키마의 필드 이름과 일치해야 하는 이유는?

JSON 키를 기준으로 Pydantic이 입력을 검증하고, 같은 피처 이름을 기준으로 모델 입력 순서를 구성하기 때문입니다. 이름이 다르면 검증 오류 또는 잘못된 피처 매핑으로 이어질 수 있습니다.

### Q4. 이 프로젝트에서 `run_in_executor`를 제거하면 어떤 문제가 생길 수 있습니까?

동기식 모델 추론이 FastAPI의 이벤트 루프를 점유하여 다른 요청 처리가 지연될 수 있습니다. 동시 요청이 많아질수록 응답성이 떨어질 수 있습니다.

### Q5. MNIST 프로젝트(Day 1~4)와 오늘 프로젝트의 가장 큰 차이는 무엇입니까?

서비스 아키텍처는 `Streamlit → FastAPI → PyTorch 모델`로 유사하지만 문제와 데이터 형태가 다릅니다.

- MNIST: 이미지 → 숫자 클래스, **분류**
- Day 5: 숫자형 피처 8개 → 연속적인 주택 가격, **회귀**

그 결과 전처리는 픽셀 처리에서 표준화(`mean`, `std`)로, 스키마는 이미지 데이터에서 8개의 숫자 필드로, UI는 파일 업로드에서 숫자 입력 폼으로 바뀌었습니다.

---

## 6. `# *your code*` 셀프 점검 결과

- [x] `train_test_split(..., test_size=0.2, random_state=42)`
- [x] 피처별 통계 축: `axis=0`
- [x] 정규화: `(X_train - train_mean) / train_std`
- [x] 타겟 shape 변환: `torch.FloatTensor(y_train).unsqueeze(1)`
- [x] 첫 Linear: `nn.Linear(input_dim, 64)`
- [x] 회귀 출력층: `nn.Linear(32, 1)`
- [x] 손실함수: `nn.MSELoss()`
- [x] 옵티마이저: `torch.optim.Adam(model.parameters(), lr=1e-3)`
- [x] 순전파: `model(X_batch)`
- [x] 손실 계산: `criterion(predictions, y_batch)`
- [x] 모델 저장: `torch.save(model.state_dict(), ...)`
- [x] 피처 순서: `[features[name] for name in self.feature_names]`
- [x] 배포 추론 정규화: `(values - self.mean) / self.std`
- [x] `MedInc`: `gt=0`
- [x] `HouseAge`: `ge=0, le=100`
- [x] `Latitude`: `ge=32, le=42`
- [x] 추론 스레드풀: `ThreadPoolExecutor(max_workers=4)`
- [x] 서버 시작 시 `HousingPredictor(MODEL_PATH, PREPROCESS_PATH)` 로드
- [x] Pydantic → dict: `request.model_dump()`
- [x] 비동기 추론: `loop.run_in_executor(inference_executor, predictor.predict, features)`
- [x] HTTP 오류 검사: `resp.raise_for_status()`
- [x] `MedInc` UI: `0.1 ~ 15.0`, 기본값 `3.5`, step `0.1`
- [x] `Latitude` UI: `32.0 ~ 42.0`, 기본값 `37.5`, step `0.1`
- [x] Streamlit의 8개 입력값을 동일한 스키마 키로 `request_data` 구성
- [x] 통합 테스트 POST: `requests.post(f"{API_BASE}/predict", json=case)`

---

## 7. 통합 테스트 결과

- [x] 기본값으로 가격 예측 실행 시 예상 가격 표시
- [x] `MedInc=8.0`과 `MedInc=1.0`의 새니티 체크 정상
- [x] 필드 누락 / 위도 50 / 음수 소득 / 잘못된 포맷에 대해 4xx 처리
- [x] 8개 동시 요청 처리 정상
- [x] `/health` 응답 `healthy` 확인

정상 요청, 에러 처리, 동시 요청, 헬스체크를 모두 정상적으로 확인했습니다.

---

## 8. 회고

이번 프로젝트에서는 Day 1~4에서 각각 배웠던 모델 저장, API 설계, 입력 검증, 비동기 처리, Streamlit UI가 실제 서비스에서는 하나의 파이프라인으로 연결된다는 점을 확인할 수 있었습니다.

특히 모델 가중치뿐 아니라 학습 데이터의 `mean`, `std`와 피처 순서를 함께 저장해야 배포 환경에서도 학습 때와 동일한 입력을 만들 수 있다는 점이 중요했습니다. 모델 파일만 정상이어도 전처리가 달라지면 오류 없이 잘못된 예측이 발생할 수 있다는 점을 이해했습니다.
