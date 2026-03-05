---
title: "[인턴 활동] 2GB를 넘는 영상을 Gemini에 넣는 법 — 청크 분할 전략"
date: 2026-03-05 00:00:00 +0900
categories: [인턴 활동]
tags: [Gemini, 청크, 영상분할, ffmpeg, FilesAPI]
---

> 이 글은 **2026년 3월 5일 기준** Gemini Files API 및 ffmpeg을 활용한 청크 분할 방식을 정리한 글입니다.

---

전 글에서 Files API의 파일 하나당 한도가 2GB라는 것을 확인했다. 드라마 원본처럼 그보다 큰 파일을 Gemini에 넣으려면 분할이 필요하다. 어떻게 나누고, 어떻게 올리고, 결과를 어떻게 합치는지 전략을 정리한다.

## 1단계: ffmpeg으로 시간 기준 분할

파일 크기 기준이 아니라 **시간 기준**으로 나누는 게 낫다. 크기는 화질마다 다르지만, 시간은 일정하게 제어할 수 있다.

```bash
# 30분 단위로 분할 (-c copy: 재인코딩 없이 빠르게)
ffmpeg -i input.mp4 -c copy -map 0 -segment_time 1800 -f segment chunk_%02d.mp4
```

`-segment_time 1800`은 1800초(30분) 단위로 자른다는 뜻이다. `-c copy`를 쓰면 재인코딩 없이 빠르게 분할된다.

분할된 파일이 2GB 이하인지 확인한 뒤 업로드로 넘어간다.

```python
import os

def check_chunk_sizes(chunk_paths: list[str], limit_gb: float = 1.8) -> None:
    limit_bytes = limit_gb * 1024 ** 3
    for path in chunk_paths:
        size = os.path.getsize(path)
        if size > limit_bytes:
            raise ValueError(f"{path}: {size / 1024**3:.2f}GB — 한도 초과")
    print("모든 청크 크기 정상")
```

여유 있게 1.8GB를 기준으로 잡는 게 안전하다.

## 2단계: 청크별 업로드

```python
import time
from google import genai
from google.genai.types import FileData, Part

client = genai.Client()

def upload_chunk(path: str) -> genai.types.File:
    print(f"업로드 중: {path}")
    video_file = client.files.upload(file=path)

    # ACTIVE 상태가 될 때까지 대기
    while video_file.state.name == "PROCESSING":
        time.sleep(5)
        video_file = client.files.get(name=video_file.name)

    if video_file.state.name == "FAILED":
        raise RuntimeError(f"업로드 실패: {path}")

    print(f"완료: {video_file.name}")
    return video_file

chunk_paths = ["chunk_00.mp4", "chunk_01.mp4", "chunk_02.mp4"]
uploaded_chunks = [upload_chunk(p) for p in chunk_paths]
```

## 3단계: 청크별 Gemini 분석

각 청크를 순서대로 분석한다. 청크마다 독립적으로 요청을 보내고, 결과를 리스트로 모은다.

```python
def analyze_chunk(video_file, chunk_index: int, total: int) -> str:
    prompt = f"""
    이 영상은 전체 {total}개 청크 중 {chunk_index + 1}번째입니다.
    감정이 고조되는 구간의 타임스탬프(MM:SS)와 해당 장면을 설명해주세요.
    JSON 형식으로 출력하세요.
    """
    response = client.models.generate_content(
        model="gemini-3.1-pro-preview",
        contents=[video_file, prompt],
    )
    return response.text

results = [
    analyze_chunk(chunk, i, len(uploaded_chunks))
    for i, chunk in enumerate(uploaded_chunks)
]
```

## 4단계: 청크 경계 문제 — 오버랩으로 보완

가장 큰 문제는 청크 경계에서 장면이 잘리는 것이다. 30분 단위로 자르면 29분 50초~30분 10초 구간의 장면은 두 청크에 걸쳐 나뉜다.

해결책은 **오버랩 구간**을 두는 것이다.

```bash
# Python으로 오버랩 분할 (각 청크 앞뒤 2분씩 겹침)
import subprocess

def split_with_overlap(input_path: str, chunk_duration: int = 1800, overlap: int = 120):
    """chunk_duration초 단위, overlap초 겹침으로 분할"""
    # 전체 길이 확인
    result = subprocess.run(
        ["ffprobe", "-v", "quiet", "-show_entries", "format=duration",
         "-of", "csv=p=0", input_path],
        capture_output=True, text=True,
    )
    total = float(result.stdout.strip())

    chunks = []
    start = 0
    while start < total:
        end = min(start + chunk_duration + overlap, total)
        out = f"chunk_{len(chunks):02d}.mp4"
        subprocess.run([
            "ffmpeg", "-y", "-ss", str(max(0, start - overlap)),
            "-to", str(end), "-i", input_path, "-c", "copy", out
        ], check=True, stdout=subprocess.DEVNULL, stderr=subprocess.PIPE)
        chunks.append(out)
        start += chunk_duration

    return chunks
```

오버랩 구간이 겹치므로 결과를 합칠 때 타임스탬프 중복을 제거하는 로직이 필요하다.

## 5단계: 사용 후 청크 파일 삭제

48시간이 지나면 자동 삭제되지만, 용량 관리를 위해 분석이 끝나면 바로 지우는 게 낫다.

```python
for chunk in uploaded_chunks:
    client.files.delete(name=chunk.name)
    print(f"삭제 완료: {chunk.name}")
```

## 정리

| 단계 | 핵심 |
|------|------|
| 분할 | 시간 기준, 1.8GB 이하 확인 |
| 업로드 | ACTIVE 상태 확인 후 다음 단계 |
| 분석 | 청크 번호/전체 수를 프롬프트에 명시 |
| 경계 처리 | 오버랩 구간으로 장면 잘림 방지 |
| 정리 | 분석 후 즉시 삭제 |

2GB 제한은 우회할 수 없지만, 제대로 나누면 아무 문제가 없다.
