---
title: "LLM Serving 아키텍처 이해 및 로컬 환경 구축"
date: 2026-03-02 00:00:00 +0900
categories: [인턴 활동]
tags: [LLM, Ollama, vLLM, AI, 백엔드, 인프라]
---

| 항목 | 내용 |
|------|------|
| 주제 | LLM Serving 아키텍처 이해 및 로컬 환경 구축 |
| 학습 방법 | 개념 학습 + 실습 (Ollama 로컬 서버 구축) |

---

## 1. 학습 배경 및 목적

백엔드 개발자로서 AI 기능을 서비스에 통합하는 수요가 증가함에 따라, 외부 API 의존 방식과 자체 LLM 서버 운용 방식의 차이를 이해하고 실무에서의 기술 선택 판단력을 키우기 위해 학습을 진행했다.

---

## 2. 핵심 개념 정리

### 2-1. LLM Serving이란?

LLM(Large Language Model)을 HTTP API 서버 형태로 배포하여, 다양한 클라이언트가 네트워크를 통해 모델을 호출할 수 있도록 만드는 것을 말한다.

```
모델 파일만 존재  →  로컬에서만 사용 가능
LLM Serving 적용  →  HTTP 요청으로 어디서든 호출 가능
```

### 2-2. 외부 API vs 내부 LLM 서버

| 구분 | 외부 API (OpenAI, Claude 등) | 내부 LLM 서버 |
|------|-------------------------------|----------------|
| 과금 방식 | 토큰 기반 종량제 | GPU 서버 월정액 |
| 초기 비용 | 없음 | 높음 |
| 트래픽 소량 | 유리 | 불리 |
| 트래픽 대량 | 불리 | 유리 |
| 데이터 보안 | 외부 전송 발생 | 내부 망에서만 처리 |
| 모델 커스터마이징 | 불가 | 파인튜닝 가능 |

> **핵심 판단 기준**: 보안 요구사항과 트래픽 규모에 따라 기술 선택이 달라진다.

### 2-3. 오픈소스 LLM (로컬 LLM)

내부 LLM 서버 구축은 반드시 **오픈소스 LLM**을 사용해야 한다. Claude, GPT-4 등 상용 모델은 모델 가중치가 비공개이므로 설치가 불가능하고 API로만 접근 가능하다.

**대표적인 오픈소스 LLM**

| 모델 | 개발사 | 특징 |
|------|--------|------|
| Llama 3 | Meta | 범용 고성능 |
| Mistral | Mistral AI | 경량 고성능 |
| Gemma | Google | 다양한 크기 제공 |
| Qwen | Alibaba | 다국어 지원 |
| EXAONE | LG AI연구원 | 한국어 특화 |

**용어 구분**

- **오픈소스 LLM**: 코드 + 가중치 전부 공개
- **오픈웨이트(Open Weight)**: 가중치만 공개 (Llama, Gemma 해당)
- **로컬 LLM**: 내 서버/PC에 직접 설치해서 운용하는 LLM

### 2-4. GPU 인프라

LLM 서버는 CPU만으로는 응답 속도가 수십 초 이상 소요된다. 실용적인 운용을 위해 GPU가 필수다.

```
NVIDIA (GPU 제조)
  └─ AWS / GCP / Azure (GPU 클라우드 렌탈)
       └─ 우리 서비스 (EC2 GPU 인스턴스 사용)
```

**GPU 조달 방식**

| 방식 | 비용 | 적합한 상황 |
|------|------|------------|
| AWS EC2 GPU 인스턴스 | 월 수백만원 | 프로덕션 서비스 |
| 자체 GPU 서버 구매 (A100 등) | 수천만원 (일회성) | 대규모 장기 운용 |
| 맥북 M 시리즈 (로컬) | 추가 비용 없음 | 개발/테스트 |

---

## 3. 대표 LLM Serving 도구

### 3-1. Ollama

가장 간단하게 로컬 LLM 서버를 구축할 수 있는 도구. 개발 및 테스트 환경에 적합하다.

```bash
# 서버 실행 (터미널 1)
ollama serve

# 모델 다운로드 (터미널 2)
ollama pull llama3

# 대화 테스트
ollama run llama3

# API 호출
curl http://localhost:11434/api/generate \
  -d '{"model": "llama3", "prompt": "안녕?"}'
```

### 3-2. vLLM

프로덕션 환경에 적합한 고성능 LLM 서빙 프레임워크. OpenAI 호환 API를 제공하므로 기존 코드 변경 없이 전환 가능하다.

```bash
python -m vllm.entrypoints.openai.api_server \
  --model mistralai/Mistral-7B-v0.1 \
  --port 8000
```

```python
from openai import OpenAI

# base_url만 변경하면 외부 API → 내부 LLM 전환 완료
client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-needed"
)
```

---

## 4. 실습 - 맥북 로컬 LLM 서버 구축

### 4-1. 환경

- 장비: MacBook (Apple Silicon)
- 도구: Ollama
- 모델: Llama 3 (4.7GB)

### 4-2. 실습 과정

```bash
# Step 1. Ollama 설치
brew install ollama

# Step 2. 서버 실행 (백그라운드 유지 필요)
ollama serve
# → Listening on 127.0.0.1:11434

# Step 3. 모델 다운로드 (새 터미널 탭에서)
ollama pull llama3

# Step 4. 대화 테스트
ollama run llama3
# >>> 프롬프트에서 직접 입력 가능
```

### 4-3. 트러블슈팅

| 에러 | 원인 | 해결 |
|------|------|------|
| `could not connect to ollama server` | `ollama serve` 실행 전에 `pull` 시도 | serve 먼저 실행 후 새 탭에서 pull |
| `Generating new private key` 메시지 | 최초 실행 시 인증키 자동 생성 | 정상 동작, 무시 |

### 4-4. Spring Boot 연동 예시

```java
@Service
public class LlmService {
    private final RestTemplate restTemplate;

    public String ask(String prompt) {
        String url = "http://localhost:11434/api/generate";
        Map<String, String> body = Map.of(
            "model", "llama3",
            "prompt", prompt
        );
        return restTemplate.postForObject(url, body, String.class);
    }
}
```

기존 Gemini API 연동 방식과 구조가 동일하며, URL만 localhost로 변경된다.

---

## 5. LLM Serve vs LangChain

두 개념은 역할이 다르며 함께 사용한다.

| 구분 | LLM Serve | LangChain |
|------|-----------|-----------|
| 역할 | LLM을 API 서버로 제공 | LLM 활용 애플리케이션 개발 프레임워크 |
| 담당 영역 | 모델 로딩, 엔드포인트 제공, 배치 처리 | 프롬프트 관리, RAG, 에이전트, 체인 구성 |
| 비유 | 엔진 (동력 제공) | 자동차 설계도 (엔진 활용) |

```python
# LangChain + vLLM 조합 예시
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(
    base_url="http://localhost:8000/v1",  # LLM Serve
    model="llama3"
)
chain = prompt | llm | output_parser      # LangChain
```

---

## 6. 기술 선택 가이드

| 상황 | 권장 선택 |
|------|-----------|
| 개인 학습 / 테스트 | 맥북 + Ollama |
| 빠른 프로토타입 | 외부 API (OpenAI 등) |
| 의료/금융 등 보안 필수 서비스 | EC2 GPU + 오픈소스 LLM |
| 대규모 트래픽 프로덕션 | EC2 GPU + vLLM |
| 복잡한 AI 파이프라인 | LangChain + LLM Serve 조합 |

---

## 7. 학습 소감 및 적용 방향

외부 API와 내부 LLM 서버의 차이를 단순히 "유료 vs 무료"로 오해하기 쉬웠는데, 실제로는 **보안 요구사항, 트래픽 규모, 모델 커스터마이징 필요 여부**에 따른 아키텍처 선택의 문제임을 이해했다.

의료 데이터를 다루는 DentalLink 같은 서비스의 경우, 개인 건강 정보가 외부 API 서버로 전송되는 리스크가 있기 때문에 내부 LLM 서버 구축이 실질적인 대안이 될 수 있다는 점을 인식했다.

향후 vLLM 기반 프로덕션 서버 구축 및 오픈소스 모델 파인튜닝까지 학습을 확장할 계획이다.
