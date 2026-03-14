---
title: "[인턴 일다] 5일차 - 가로 영상을 세로로 — 9:16 letterbox와 오디오 처리"
date: 2026-03-14 00:00:00 +0900
categories: [인턴 일다]
tags: [ffmpeg, 쇼츠, 세로영상, letterbox, ffprobe]
---

드라마 원본은 16:9 가로 영상이다. 쇼츠는 9:16 세로다. 단순히 크기를 바꾸면 영상이 찌그러진다. 올바른 변환 방법을 찾아야 했다.

## letterbox vs 단순 scale

영상 원본의 비율에 따라 처리 방식이 달라진다.

| 원본 비율 | 처리 방식 | ffmpeg 필터 |
|-----------|-----------|-------------|
| 세로 (9:16, 4:5 등) | 단순 스케일 | `scale=1080:1920` |
| 가로 (16:9 등) | letterbox | `scale=1080:608,pad=1080:1920:0:656:black` |

letterbox는 가로 영상을 1080px 너비로 줄인 뒤 (→ 높이 608px), 위아래에 검은 여백을 패딩해 1920px 높이를 맞추는 방식이다. 영상은 세로 656~1264 구간에 위치하게 된다.

## 영상 방향 자동 감지

어떤 영상이 들어와도 올바른 필터를 쓰려면, ffprobe로 해상도를 먼저 읽어야 한다.

```python
def _get_scale_filter(video_path: str) -> str:
    result = subprocess.run(
        ["ffprobe", "-v", "quiet", "-select_streams", "v:0",
         "-show_entries", "stream=width,height", "-of", "csv=p=0", video_path],
        capture_output=True, text=True,
    )
    w, h = map(int, result.stdout.strip().split(","))
    if h >= w:
        return "scale=1080:1920"
    else:
        return "scale=1080:608,pad=1080:1920:0:656:black"
```

높이가 너비보다 크거나 같으면 세로형, 아니면 가로형으로 판단한다.

## 오디오 없는 영상의 함정

`-map 0:a`는 오디오 스트림이 없으면 ffmpeg이 에러를 낸다. 처음에 이것 때문에 특정 클립에서 계속 실패했다.

```bash
# 오디오 없으면 에러
-map 0:a

# 오디오 있으면 포함, 없으면 무시
-map 0:a?
```

`?`를 붙이면 스트림이 없을 때 무시하고 넘어간다. 옵션 하나의 차이다.

## 클립 인코딩 함수

최종적으로 만든 `encode_clip_vertical` 함수는 다음 세 가지를 담당한다.

1. 세로 포맷 변환 (scale/letterbox)
2. 텍스트 이미지 overlay 합성
3. 30fps 강제 지정 (concat 시 프레임레이트 충돌 방지)

```python
def encode_clip_vertical(video_path, start, end, output_path, overlays=[]):
    scale_filter = _get_scale_filter(video_path)
    cmd = ["ffmpeg", "-y", "-ss", str(start), "-to", str(end), "-i", video_path]

    for img_path, _ in overlays:
        cmd += ["-i", img_path]

    filter_parts = [f"[0:v]{scale_filter}[v0]"]
    prev = "v0"
    for i, (_, y) in enumerate(overlays):
        nxt = "out" if i == len(overlays) - 1 else f"v{i+1}"
        filter_parts.append(f"[{prev}][{i+1}:v]overlay=0:{y}[{nxt}]")
        prev = nxt

    cmd += ["-filter_complex", ";".join(filter_parts),
            "-map", "[out]", "-map", "0:a?", "-r", "30",
            "-c:v", "libx264", "-c:a", "aac", "-preset", "fast", output_path]

    subprocess.run(cmd, check=True, stdout=subprocess.DEVNULL, stderr=subprocess.PIPE)
```

세그먼트를 모두 만든 뒤 `-c copy`로 concat하면 쇼츠 완성본이 나온다. 인코딩은 세그먼트 단계에서 끝났으므로 concat은 재인코딩 없이 빠르게 완료된다.
