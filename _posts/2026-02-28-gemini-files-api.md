---
title: "[인턴 활동] Gemini Files API — 큰 파일을 모델에게 넘기는 방법"
date: 2026-02-28 00:00:00 +0900
categories: [인턴 활동]
tags: [Gemini, FilesAPI, 멀티모달, 파일업로드]
---

> 이 글은 **2026년 2월 28일 기준** Google Gemini Files API 공식 문서를 참고해 작성했습니다.
> API 스펙은 변경될 수 있으니 최신 정보는 [공식 문서](https://ai.google.dev/gemini-api/docs/files?hl=ko)에서 확인하세요.

---

Gemini에 영상 파일을 넣어서 분석을 시키려고 하면 금방 벽에 부딪힌다. 파일이 크면 그냥 요청 본문에 넣을 수 없다. 그때 쓰는 게 **Files API**다.

## 왜 Files API가 필요한가

Gemini API 요청에는 크기 제한이 있다. 요청 전체가 **100MB를 넘으면** 인라인으로 파일을 넣는 방식은 쓸 수 없다. 드라마 한 편은 수 GB가 넘으니, Files API 없이는 영상 분석 자체가 불가능하다.

Files API를 쓰면 파일을 먼저 Google 서버에 올려두고, 이후 Gemini 요청에서 그 파일을 참조하는 방식으로 처리한다.

## 제한사항 먼저

| 항목 | 제한 |
|------|------|
| 프로젝트당 저장 용량 | 최대 **20GB** |
| 파일 하나의 크기 | 최대 **2GB** (PDF는 50MB) |
| 파일 보관 기간 | 업로드 후 **48시간** |
| 비용 | **무료** (모든 Gemini API 지원 지역) |

48시간이 지나면 자동으로 삭제된다. 영구 저장이 필요하면 Cloud Storage를 따로 써야 한다.

## 기본 흐름

```
파일 업로드 → URI 획득 → Gemini 요청에 URI 포함 → 분석 결과 수신
```

코드로 보면 이렇다.

```python
from google import genai

client = genai.Client()

# 1. 파일 업로드
video_file = client.files.upload(file="drama_episode.mp4")

# 2. 업로드 완료 상태 확인 (영상은 처리 시간이 걸림)
import time
while video_file.state.name == "PROCESSING":
    time.sleep(5)
    video_file = client.files.get(name=video_file.name)

# 3. Gemini 요청에 파일 포함
response = client.models.generate_content(
    model="gemini-3.1-pro-preview",
    contents=[video_file, "이 영상에서 감정이 가장 고조되는 구간을 찾아줘"],
)
print(response.text)
```

## 주요 메서드

**`files.upload()`** — 파일 업로드

```python
video_file = client.files.upload(file="path/to/file.mp4")
print(video_file.uri)  # gs://... 형태의 URI 반환
```

**`files.get()`** — 파일 상태 및 메타데이터 조회

```python
file_info = client.files.get(name=video_file.name)
print(file_info.state)  # PROCESSING / ACTIVE / FAILED
```

영상 파일은 업로드 직후 바로 쓸 수 없다. `state`가 `ACTIVE`가 될 때까지 기다려야 한다.

**`files.list()`** — 업로드된 파일 목록 조회

```python
for f in client.files.list():
    print(f.name, f.state)
```

**`files.delete()`** — 파일 수동 삭제

```python
client.files.delete(name=video_file.name)
```

48시간 안에 지워지긴 하지만, 용량 관리나 보안이 필요하면 직접 삭제하는 게 낫다.

## 영상 처리 시 주의할 점

영상 파일은 이미지나 텍스트와 달리 업로드 후 **서버 측 처리 시간**이 추가로 필요하다. 상태가 `PROCESSING`인 동안 Gemini 요청에 넣으면 에러가 난다.

폴링 루프를 넣어 `ACTIVE` 상태를 확인한 뒤 요청을 보내야 안전하다.

```python
while video_file.state.name == "PROCESSING":
    time.sleep(5)
    video_file = client.files.get(name=video_file.name)

if video_file.state.name == "FAILED":
    raise ValueError("파일 처리 실패")
```

## 정리

Files API는 단순하다. 올리고, 확인하고, 쓰고, 지운다. 복잡한 게 없는 대신 **48시간 보관 기간**과 **상태 확인 필수**라는 두 가지만 잘 기억하면 된다. 대용량 영상 분석 파이프라인을 짤 때 이 두 가지를 빠뜨리면 예상치 못한 곳에서 막힌다.
