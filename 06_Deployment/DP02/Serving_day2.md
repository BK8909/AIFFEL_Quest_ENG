# 모델 배포 개론 — Day 2 제출

**주제:** FastAPI 기초와 데이터 처리 (모델 추론 엔드포인트 구현)

> 제출 항목
> 1. 섹션 1.5 수행 내역
> 2. 섹션 2, 3 셀 출력
> 3. 섹션 5 수행 내역
> 4. 각 섹션 체크포인트 답변

---

## 1. 섹션 1.5 수행 내역 — 최소한의 FastAPI 서버 실행

![섹션 1.5 실행 화면](스크린샷%201-5.png)

첫 번째 API 서버가 정상 동작함을 확인했습니다. `dict` 반환값이 자동으로 JSON으로 직렬화되어 응답되었습니다.

---

## 2. 섹션 2 셀 출력 — Path / Query / Body 파라미터

### Path 파라미터 (`/models/{model_name}`)

```python
response = requests.get("http://localhost:8000/models/sentiment-v1")
print(response.json())
response = requests.get("http://localhost:8000/models/image-classifier")
print(response.json())
```

```
{'model_name': 'sentiment-v1', 'status': 'running', 'version': '1.0.0'}
{'model_name': 'image-classifier', 'status': 'running', 'version': '1.0.0'}
```

### Path 파라미터 타입 검증 (`prediction_id: int`)

```python
response = requests.get("http://localhost:8000/predictions/42")
print(f"상태: {response.status_code}, 응답: {response.json()}")
response = requests.get("http://localhost:8000/predictions/abc")
print(f"상태: {response.status_code}")
print(f"에러: {response.json()}")
```

```
상태: 200, 응답: {'prediction_id': 42, 'label': '긍정', 'confidence': 0.92}
상태: 422
에러: {'detail': [{'type': 'int_parsing', 'loc': ['path', 'prediction_id'], 'msg': 'Input should be a valid integer, unable to parse string as an integer', 'input': 'abc'}]}
```

타입 힌트만 선언했을 뿐인데 `"abc"` 입력에 대해 FastAPI가 자동으로 422를 반환했습니다.

### Query 파라미터 (`/models?status=&limit=`)

```python
response = requests.get("http://localhost:8000/models")
print("전체 모델:", response.json())
response = requests.get("http://localhost:8000/models?status=running")
print("running만:", response.json())
response = requests.get("http://localhost:8000/models?status=running&limit=1")
print("running, 1개만:", response.json())
```

```
전체 모델: {'total': 3, 'models': [{'name': 'sentiment-v1', 'status': 'running'}, {'name': 'image-clf-v2', 'status': 'running'}, {'name': 'ner-v1', 'status': 'stopped'}]}
running만: {'total': 2, 'models': [{'name': 'sentiment-v1', 'status': 'running'}, {'name': 'image-clf-v2', 'status': 'running'}]}
running, 1개만: {'total': 1, 'models': [{'name': 'sentiment-v1', 'status': 'running'}]}
```

### Request Body (`POST /predict`)

```python
response = requests.post("http://localhost:8000/predict",
    json={"text": "이 영화 정말 재밌다"})
print("기본 응답:", response.json())
response = requests.post("http://localhost:8000/predict",
    json={"text": "이 영화 정말 재밌다", "return_probabilities": True})
print("확률 포함:", response.json())
```

```
기본 응답: {'label': '긍정', 'confidence': 0.92, 'probabilities': None}
확률 포함: {'label': '긍정', 'confidence': 0.92, 'probabilities': {'긍정': 0.92, '부정': 0.05, '중립': 0.03}}
```

### Request Body 검증 실패 케이스

```
상태: 422
에러: Field required
상태: 422
에러: Input should be a valid string
```

---

## 3. 섹션 3 셀 출력 — Swagger UI / OpenAPI 스펙

### 자동 생성된 OpenAPI 스펙 확인

```python
response = requests.get("http://localhost:8000/openapi.json")
spec = response.json()
print(f"API 제목: {spec['info']['title']}")
print(f"API 버전: {spec['info']['version']}")
for path, methods in spec['paths'].items():
    for method in methods:
        print(f"  {method.upper():6s} {path}")
```

```
API 제목: Parameter Examples
API 버전: 0.1.0

등록된 엔드포인트:
  GET    /models/{model_name}
  GET    /predictions/{prediction_id}
  GET    /models
  POST   /predict
```

### PredictRequest JSON Schema

```python
predict_schema = spec['components']['schemas']['PredictRequest']
print(json.dumps(predict_schema, indent=2, ensure_ascii=False))
```

```json
{
  "properties": {
    "text": { "type": "string", "title": "Text" },
    "return_probabilities": {
      "type": "boolean",
      "title": "Return Probabilities",
      "default": false
    }
  },
  "type": "object",
  "required": ["text"],
  "title": "PredictRequest"
}
```

### ReDoc 응답 확인

```python
resp = requests.get("http://localhost:8000/redoc")
print(f"상태: {resp.status_code}")
print(f"내용 길이: {len(resp.text)}")
```

```
상태: 200
내용 길이: 899
```

---

## 4. 섹션 5 수행 내역 — 모델 추론 엔드포인트 구현 및 테스트

### 모델 로드 → 서버 실행

![모델 로드 및 서버 실행](스크린샷%205_1.png)

서버 시작 시 모델(`SimpleClassifier`)을 1회만 로드하여 메모리에 상주시켰고, 헬스체크에서 `model_loaded: True`를 확인했습니다.

### 실제 MNIST 이미지 추론

![추론 테스트 — 정답 7 예측](스크린샷%205_2.png)

실제 MNIST 테스트 이미지(정답 7)를 추론하여 `label: 7, confidence: 1.0`을 반환받았습니다.

### 10개 이미지 연속 추론

![10개 이미지 연속 추론 — 정확도 10/10](스크린샷%205_3.png)

10개 이미지 연속 추론 결과 정확도 10/10 (100%)를 확인했습니다.

### 에러 상황 테스트

![에러 1·2 — 픽셀 개수 위반 / 타입 오류](스크린샷%20error_1.png)

![에러 3·4 — 필수 필드 누락 / 빈 JSON](스크린샷%20error_2.png)

모든 잘못된 요청(길이 위반, 타입 오류, 필드 누락, 빈 JSON)에 대해 서버가 죽지 않고 적절한 상태 코드(422)와 메시지를 반환했습니다. 검증 로직을 직접 작성하지 않았으며, Pydantic 스키마 선언만으로 처리되었습니다.

---

## 5. 각 섹션 체크포인트 답변

### 섹션 1 체크포인트 (FastAPI 개요)

**1. FastAPI가 Flask보다 모델 배포에 적합한 이유 세 가지**
- **자동 입력 검증:** Pydantic 기반으로 타입·제약(길이, 범위 등)을 *선언*만 하면 검증이 자동 수행됩니다. `if/else` 검증 코드를 직접 작성할 필요가 없습니다.
- **자동 API 문서화:** 코드(타입 힌트, Pydantic 모델, docstring)로부터 Swagger UI(`/docs`)와 ReDoc(`/redoc`)이 자동 생성되며, 코드와 문서가 항상 동기화됩니다.
- **네이티브 비동기 처리:** `async/await`를 기본 지원하여, 모델 추론처럼 대기 시간이 있는 I/O 작업에서 동시 처리 효율이 높습니다.

**2. Uvicorn의 역할, 왜 FastAPI와 함께 사용하는가**
FastAPI는 ASGI *애플리케이션(프레임워크)*일 뿐, 스스로 네트워크에서 HTTP 요청을 받지 못합니다. Uvicorn은 **ASGI 서버**로서 실제 소켓에서 HTTP 요청을 받아 ASGI 규약에 따라 FastAPI 앱에 전달하고 응답을 돌려줍니다. 즉 **FastAPI = 앱, Uvicorn = 그 앱을 구동하는 서버**이며, FastAPI의 async 기능을 살리려면 ASGI 서버가 필요하므로 Uvicorn을 사용합니다.

**3. `@app.get("/health")`에서 `get`과 `"/health"`의 의미**
- `get` → HTTP 메서드(GET)
- `"/health"` → 요청 경로(path)
즉 "`GET /health` 요청이 들어오면 이 함수를 실행하라"는 라우팅 선언입니다.

**4. dict를 반환하면 자동으로 일어나는 일**
FastAPI가 dict를 **JSON으로 자동 직렬화**하고, `Content-Type: application/json` 헤더를 설정하며, 상태 코드 200을 붙여 응답합니다. `json.dumps()`를 수동 호출할 필요가 없습니다.

---

### 섹션 2 체크포인트 (Path / Query / Body)

**1. `/models/sentiment-v1`에서 `sentiment-v1`의 종류** → **Path 파라미터** (URL 경로에 포함된 값)

**2. `/models?status=running&limit=5`에서 `status`, `limit`의 종류** → **Query 파라미터** (URL 뒤 `?key=value` 형태)

**3. 모델 추론 요청에 Request Body를 사용하는 이유**
추론 입력은 크고 구조화된 데이터(텍스트, 784개 픽셀 배열 등)입니다. URL은 길이 제한이 있고, URL에 대용량·민감 데이터를 넣으면 서버 로그·브라우저 히스토리에 남는 문제가 있습니다. Request Body는 JSON으로 복잡한 중첩 구조를 담을 수 있고 Pydantic 스키마로 검증할 수 있어, 추론 입력 전달에 적합합니다.

**4. FastAPI가 파라미터 출처를 판별하는 방법**
규칙 기반으로 자동 판별합니다.
- 경로 문자열의 `{중괄호}`에 선언된 이름 → **Path**
- 함수 인자 중 경로에 없고 단순 타입(str, int, bool 등)이면 → **Query**
- 인자 타입이 Pydantic `BaseModel`이면 → **Request Body**

---

### 섹션 3 체크포인트 (Swagger UI)

**1. Swagger UI 접속 URL** → `/docs` (예: `http://localhost:8000/docs`)

**2. Swagger UI가 코드와 항상 동기화될 수 있는 이유**
문서를 수동 작성하는 것이 아니라, FastAPI가 런타임에 코드(타입 힌트, Pydantic 모델, docstring, 데코레이터)로부터 **OpenAPI 스펙(`openapi.json`)을 자동 생성**하고 Swagger UI가 그 스펙을 렌더링하기 때문입니다. 코드가 곧 단일 진실 원천(single source of truth)이라, 코드를 수정하면 문서가 자동으로 갱신됩니다.

**3. `Field(description=, examples=)`가 Swagger UI에 반영되는 위치**
- `description` → 해당 필드/파라미터의 설명란
- `examples` → "Try it out" 시 요청 본문 입력창에 자동으로 채워지는 예시 값
두 값 모두 JSON Schema에 실려 Swagger UI가 렌더링합니다.

**4. Swagger UI와 ReDoc의 핵심 차이**
- **Swagger UI:** 인터랙티브. 브라우저에서 직접 API를 호출·테스트할 수 있습니다("Try it out"). → 개발·테스트에 적합
- **ReDoc:** 읽기 전용, 정적인 문서. 가독성 좋은 레퍼런스 문서화에 강점이 있으나 직접 호출은 불가 → 공개 API 레퍼런스에 적합

---

### 섹션 4 체크포인트 (Pydantic)

**1. `text: str`과 `text: str = "기본값"`의 차이**
- `text: str` → **필수 필드** (없으면 검증 에러)
- `text: str = "기본값"` → **선택 필드**, 생략 시 기본값 사용

**2. `Field(..., min_length=1, max_length=5000)`에서 `...`의 의미**
`...`(Ellipsis)은 **"필수 필드"임을 명시적으로 표시**합니다. 기본값 자리에 `...`을 넣으면 "기본값이 없다 = 반드시 제공해야 한다"는 뜻이며, `Field()`에 검증 옵션은 주되 필수임을 유지할 때 사용합니다.

**3. 422 에러 응답에서 `loc` 필드가 담는 정보**
에러가 발생한 **위치(location)** 를 담습니다. 어디서(`body`/`query`/`path`) 어떤 필드에서 문제가 났는지를 배열로 표시합니다. 예: `['path', 'prediction_id']`, `['body', 'pixel_values']`.

**4. `response_model`을 지정하면 얻는 이점**
- 응답 스키마가 Swagger UI에 자동 문서화됩니다.
- 반환 데이터가 스키마에 맞게 **검증·필터링**되어, 스키마에 없는 필드는 응답에서 제거됩니다(민감 필드 유출 방지).
- 응답 자동 직렬화 및 타입 보장이 이루어집니다.

---

### Day 2 종합 체크포인트 (섹션 5)

**1. 모델을 서버 시작 시 한 번만 로드해야 하는 이유**
모델 로드는 파일 I/O와 파라미터 역직렬화가 포함된 무거운 작업(수 초 소요)입니다. 요청마다 로드하면 매 응답이 수 초씩 느려지고 자원이 낭비됩니다. 모듈 레벨(서버 시작 시)에 1회 로드하여 메모리에 상주시키면, 이후 요청은 추론만 수행하므로 응답이 빠릅니다.

**2. `pixel_values`가 784개가 아닌 요청이 오면 발생하는 일 / 직접 처리 코드를 작성했는가**
Pydantic `Field(min_length=784, max_length=784)` 선언에 의해, 요청이 라우트 함수에 **도달하기 전에** FastAPI가 자동으로 422를 반환합니다. 실제 출력:
```
List should have at least 784 items after validation, not 100
```
`if len(pixel_values) != 784`와 같은 검증 코드를 **직접 작성하지 않았으며**, 스키마 선언만으로 처리되었습니다. 이것이 Pydantic + FastAPI 조합의 핵심 이점입니다.

**3. `HTTPException(status_code=503)`은 어떤 상황에서 사용했는가 / 왜 500이 아니라 503인가**
**모델이 로드되지 않은 상황**(서비스가 일시적으로 요청을 처리할 수 없는 상태)에서 사용했습니다.
- **503 Service Unavailable:** 서버가 현재 일시적으로 서비스 불가하며, 재시도하면 처리될 수 있음을 의미합니다. 모델 미로드는 "지금은 준비되지 않음"이라는 예견된 상태이므로 의미상 정확합니다.
- **500 Internal Server Error:** 예기치 못한 서버 내부 오류(버그성)를 의미합니다.
503을 반환하면 클라이언트가 "재시도하면 될 수 있는 상황"인지 "서버가 고장 난 상황"인지 구분할 수 있습니다.

**4. Swagger UI에서 `PredictRequest`의 `description`과 `examples`가 표시되는 위치**
`/predict` 엔드포인트를 펼쳤을 때,
- `description` → 요청 본문(Request body) 각 필드의 설명으로 표시됩니다.
- `examples` → "Try it out" 시 요청 본문 입력창에 자동으로 채워지는 예시 값(및 스키마 examples)으로 표시됩니다.

Field에 지정한 `description`, `examples`가 그대로 문서에 반영됩니다.

---

*Day 2 학습 목표 달성: 주피터 노트북에서만 동작하던 MNIST 모델을 HTTP 요청으로 호출 가능한 API로 배포하고, Pydantic 기반 입력 검증과 에러 처리를 갖춘 추론 엔드포인트를 완성했습니다.*
