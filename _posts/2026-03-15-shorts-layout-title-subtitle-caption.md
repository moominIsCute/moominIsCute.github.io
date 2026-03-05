---
title: "[인턴 활동] YouTube Shorts 레이아웃 — 제목 바, 대사 자막, 캡션 바"
date: 2026-03-15 00:00:00 +0900
categories: [인턴 활동]
tags: [쇼츠, 레이아웃, Pillow, ffmpeg, UI]
---

영상은 나왔다. 그런데 뭔가 이상했다. 검은 화면만 나오고, 자막도 없고, "구독좋아요"라는 글씨가 갑자기 뜨다가 사라졌다. 실제 YouTube Shorts처럼 보이려면 레이아웃을 처음부터 다시 설계해야 했다.

## 참고 레이아웃 — 실제 Shorts 채널 분석

드라마 쇼츠 채널들을 보면 공통적인 레이아웃 패턴이 있다.

```
┌─────────────────────────┐
│  [제목 바 — 어두운 배경]  │  ← 노란 텍스트, 상단 고정
│  2줄로 작품명/훅 표시    │
├─────────────────────────┤
│                         │
│       [ 영상 ]          │  ← 실제 드라마 장면
│                         │
│  "말까하고 있다?"        │  ← 대사 자막 (반투명 박스)
│                         │
├─────────────────────────┤
│  [핑크 캡션 바]          │  ← 장면 요약 한 줄
└─────────────────────────┘
```

이 3레이어 구조를 코드로 만드는 것이 목표였다.

## 두 가지 텍스트 스타일

텍스트 이미지 생성 함수에 `style` 파라미터를 추가해 두 가지 모드를 지원한다.

```python
def _make_text_image(..., style: str = "overlay"):
    if style == "bar":
        # 전체 너비를 solid 색상으로 채움
        img = Image.new("RGBA", (width, img_h), bg_rgba)
    else:
        # 투명 배경 + 텍스트 뒤에만 반투명 박스
        img = Image.new("RGBA", (width, img_h), (0, 0, 0, 0))
        draw_tmp.rectangle([box_x, 0, box_x + max_w + pad*2, total_h + pad*2], fill=bg_rgba)
```

| 레이어 | style | 색상 | 폰트 크기 |
|--------|-------|------|-----------|
| 제목 바 (상단) | bar | 텍스트 노란(255,230,0), 배경 거의 검정 | 72px |
| 대사 자막 (중간) | overlay | 흰색, 반투명 검정 박스 | 52px |
| 캡션 바 (하단) | bar | 흰색, 핑크(180,50,120) | 44px |

## 영상 방향에 따른 y좌표 분기

letterbox(가로 원본)와 세로 원본은 영상이 배치되는 위치가 다르다. y좌표를 하드코딩하면 한쪽에서 레이아웃이 틀어진다.

```python
scale_filter = _get_scale_filter(video_path)
if "pad" in scale_filter:
    # 가로 영상 → letterbox: 영상이 y=656~1264에 위치
    title_y, subtitle_y, caption_y = 0, 900, 1290
else:
    # 세로 영상 → 전체가 영상
    title_y, subtitle_y, caption_y = 0, 1400, 1700
```

## 전체 완성본 생성 흐름

```python
def build_shorts_complete(video_path, clip_data, base):
    clips = clip_data.get("clips", [])
    title = clip_data.get("title", "")

    with tempfile.TemporaryDirectory() as tmp:
        # 1. 제목 이미지 생성 (모든 클립 공통)
        title_img = _make_text_image(title, ..., style="bar")

        # 2. 각 클립마다 세그먼트 생성
        for i, clip in enumerate(clips):
            subtitle = clip.get("subtitle", "")
            caption = clip.get("caption", "")
            overlays = [(title_img, title_y)]
            if subtitle: overlays.append((sub_img, subtitle_y))
            if caption:  overlays.append((cap_img, caption_y))
            encode_clip_vertical(video_path, clip["start"], clip["end"], seg, overlays)

        # 3. 세그먼트 concat → 완성본
        subprocess.run(["ffmpeg", "-f", "concat", "-c", "copy", ...])
```

Gemini가 분석한 clip_json 하나로, 제목·대사·캡션이 모두 들어간 세로 쇼츠 완성본이 자동으로 만들어진다.

## 프롬프트와 코드가 계약하는 방식

clip_json 형식은 Gemini 프롬프트와 Python 파싱 코드가 함께 유지해야 하는 인터페이스다.

```
프롬프트 (v1.py)         →  clip_json 형식 정의
parse_clip_data()        →  JSON 파싱 + MM.SS 변환
build_shorts_complete()  →  실제 영상 생성
```

프롬프트를 바꾸면 JSON 구조가 바뀌고, 파싱 코드도 바뀌어야 한다. 이 세 레이어의 일관성을 유지하는 것이 이 프로젝트에서 가장 신경 써야 할 부분이다.

AI가 분석하고, 코드가 편집한다. 사람이 할 일은 그 둘 사이의 계약을 잘 정의하는 것이다.
