---
title: "[인턴 활동] Gemini API 세팅 — Vertex AI냐, Google AI Studio냐"
date: 2026-02-27 00:00:00 +0900
categories: [인턴 활동]
tags: [Gemini, API, VertexAI, GoogleAIStudio, 환경설정]
---

Gemini를 실제로 쓰려면 먼저 API 연결부터 해야 한다. 생각보다 선택지가 있어서 정리해뒀다.

---

## 패키지는 두 개

```bash
pip install "google-genai>=1.49.0" "pandas[output-formatting]"
```

`google-genai`가 핵심이다. Gemini 요청을 몇 줄로 처리해주는 공식 SDK다. `pandas`는 결과 시각화용.

---

## API 경로: Vertex AI vs Google AI Studio

Gemini에 요청을 보내는 방법이 두 가지다.

**Option A — Vertex AI**
Google Cloud 프로젝트 기반. 엔터프라이즈 환경에 적합하다.

```bash
GOOGLE_GENAI_USE_VERTEXAI="True"
GOOGLE_CLOUD_PROJECT="<PROJECT_ID>"
GOOGLE_CLOUD_LOCATION="<LOCATION>"
```

**Option B — Google AI Studio**
API 키 하나면 된다. 빠르게 실험하거나 소규모 프로젝트에 적합.

```bash
GOOGLE_GENAI_USE_VERTEXAI="False"
GOOGLE_API_KEY="<API_KEY>"
```

두 경로 모두 `google-genai` SDK의 통합 인터페이스로 동일하게 쓸 수 있다. 환경변수만 바꾸면 된다.

클라이언트 생성은 한 줄이다.

```python
from google import genai

client = genai.Client()
```

---

## 모델 선택: gemini-3.1-pro-preview

```python
GEMINI_3_1_PRO_PREVIEW = "gemini-3.1-pro-preview"
```

현재 프로젝트에서는 `gemini-3.1-pro-preview`를 쓰고 있다. 멀티모달 이해 성능이 높아서 영상 분석처럼 시각·오디오·대사를 동시에 처리해야 하는 작업에 적합하다.

---

## 재현성을 위한 파라미터 설정

데이터 추출 목적으로 쓸 때는 출력이 일정해야 한다. 랜덤성을 최대한 제거하기 위해 세 가지를 고정한다.

```python
from google.genai.types import GenerateContentConfig

DEFAULT_CONFIG = GenerateContentConfig(
    temperature=0.0,
    top_p=0.0,
    seed=42,
)
```

- `temperature=0.0`: 가장 확률 높은 토큰만 선택
- `top_p=0.0`: 샘플링 범위를 최소화
- `seed=42`: 같은 입력에 같은 출력

창의적인 글쓰기가 아니라 영상에서 정보를 뽑는 작업이라면 이 설정이 기본값이다.

---

## 영상 입력 방식

Gemini에 영상을 넣는 방법은 세 가지다.

| 방식 | 예시 |
|------|------|
| YouTube URL | `https://www.youtube.com/watch?v=...` |
| Cloud Storage URI | `gs://bucket/video.mp4` |
| 업로드한 파일 | Files API로 업로드 후 참조 |

YouTube URL을 넣으면 Gemini가 자막 데이터 없이 실제 오디오/비디오 스트림을 직접 읽는다. YouTube가 제공하는 자동 자막은 전달되지 않는다. 모델이 영상 자체를 보고 판단하는 것이다.

---

## 요청 실패 시 재시도 로직

API 쿼터나 일시적 오류로 요청이 실패할 수 있다. `tenacity`로 재시도를 처리한다.

```python
import tenacity

def get_retrier() -> tenacity.Retrying:
    return tenacity.Retrying(
        stop=tenacity.stop_after_attempt(7),
        wait=tenacity.wait_incrementing(start=10, increment=1),
        retry=should_retry_request,
        reraise=True,
    )
```

7번까지 재시도하고, 간격은 10초부터 1초씩 늘어난다. 429(쿼터 초과)나 특정 400 오류에서만 재시도한다. 실패 원인을 숨기지 않고 출력한 뒤 재시도 여부를 결정하는 구조다.

ffmpeg 에러를 `DEVNULL`로 버렸다가 디버깅이 막혔던 것처럼, API 에러도 삼켜버리면 나중에 원인을 찾기 어렵다. 에러는 항상 보여야 한다.
