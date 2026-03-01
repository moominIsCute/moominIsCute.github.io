---
title: "[인턴 활동] Gemini가 영상을 이해하는 방식 — 동영상 이해 API"
date: 2026-03-01 00:00:00 +0900
categories: [인턴 활동]
tags: [Gemini, 동영상이해, 멀티모달, 타임스탬프, 영상분석]
---

> 이 글은 **2026년 3월 1일 기준** Google Gemini 동영상 이해 API 공식 문서를 참고해 작성했습니다.
> 최신 정보는 [공식 문서](https://ai.google.dev/gemini-api/docs/video-understanding?hl=ko)에서 확인하세요.

---

Files API로 영상을 올리는 법은 알겠는데, 그러면 Gemini는 그 영상을 어떻게 "보는" 걸까. 프레임을 어떻게 읽고, 오디오는 어떻게 처리하고, 타임스탬프는 어떻게 이해하는지 정리해봤다.

## 영상을 넣는 방법 4가지

상황에 따라 입력 방식이 다르다.

| 방식 | 최대 크기 | 언제 쓰나 |
|------|-----------|-----------|
| Files API | 2GB | 100MB 이상 대용량 파일 |
| 인라인 데이터 | 100MB | 1분 이하 짧은 클립 |
| Cloud Storage | 2GB | 재사용이 잦은 파일 |
| YouTube URL | 제한 없음 | 공개 유튜브 영상 |

드라마 원본처럼 수 GB짜리 파일은 Files API, 분석 결과로 잘라낸 짧은 클립 테스트는 인라인 데이터로 처리하면 된다.

## 지원 포맷

```
mp4, mpeg, mov, avi, flv, mpg, webm, wmv, 3gpp
```

대부분의 일반적인 영상 포맷은 다 된다.

## Gemini가 영상을 읽는 방식

영상을 통째로 분석하는 게 아니다. 내부적으로 이렇게 처리한다.

- **프레임 샘플링**: 기본값 **초당 1프레임(1FPS)**
- **오디오**: **1Kbps 단일 채널**로 처리
- **토큰**: 기본 해상도 기준 **초당 약 300토큰** 소비

30분짜리 드라마라면 약 1,800프레임을 보는 셈이다. 빠른 액션 장면이 많다면 FPS를 높여서 샘플링 밀도를 올릴 수 있다.

```python
from google.genai.types import VideoMetadata

# FPS 조정 예시
video_part = Part(
    file_data=FileData(file_uri=video_file.uri),
    video_metadata=VideoMetadata(fps=2)  # 기본값 1 → 2로 상향
)
```

## 타임스탬프로 특정 구간 질문하기

Gemini는 `MM:SS` 형식의 타임스탬프를 이해한다. 프롬프트에 시간대를 명시해서 특정 구간에 대해 질문할 수 있다.

```python
response = client.models.generate_content(
    model="gemini-3.1-pro-preview",
    contents=[
        video_file,
        "00:42에서 01:10 구간에서 두 인물 사이의 감정 변화를 설명해줘"
    ],
)
```

반대로 Gemini가 타임스탬프를 직접 출력하게 할 수도 있다.

```python
response = client.models.generate_content(
    model="gemini-3.1-pro-preview",
    contents=[
        video_file,
        "감정이 급격히 변하는 구간을 MM:SS 형식으로 모두 알려줘"
    ],
)
```

편집점 탐지에서 이 방식을 쓰면 Gemini가 직접 타임스탬프를 뽑아준다. 이걸 파싱해서 ffmpeg에 넘기는 구조다.

## 처리 가능한 영상 길이

| 조건 | 최대 길이 |
|------|-----------|
| 기본 해상도 (1M 컨텍스트) | 약 **1시간** |
| 낮은 해상도 | 약 **3시간** |

드라마 1화(보통 60~70분)는 기본 설정으로 처리 가능하다.

## 동일 영상을 반복 분석할 때 — 컨텍스트 캐싱

같은 영상 파일로 여러 번 다른 질문을 던질 때마다 매번 토큰을 소비하면 비용이 커진다. **컨텍스트 캐싱**을 쓰면 영상 처리 결과를 캐시해두고 재사용할 수 있다.

편집점 탐지, 감정 곡선 분석, 대사 추출을 같은 영상에 순서대로 돌린다면 컨텍스트 캐싱을 고려할 만하다.

## 정리

Gemini는 영상을 1FPS로 샘플링해서 프레임과 오디오를 같은 컨텍스트에서 처리한다. 타임스탬프를 이해하고, 직접 출력도 한다. 입력 방식은 파일 크기에 따라 고르면 된다. 100MB 미만이면 인라인, 그 이상이면 Files API가 기본이다.
