---
title: "[인턴 활동] ffmpeg drawtext가 안 된다 — Pillow로 우회한 이야기"
date: 2026-03-13 00:00:00 +0900
categories: [인턴 활동]
tags: [ffmpeg, Pillow, 한국어, 자막, 영상편집]
---

AI가 편집점을 잡아줬다. 이제 ffmpeg으로 자르고 붙이기만 하면 쇼츠가 나올 것 같았다. 그런데 막히는 건 항상 예상 못한 곳에서 나온다.

## 오류: No such filter 'drawtext'

자막을 영상에 넣으려고 ffmpeg의 drawtext 필터를 썼다.

```bash
ffmpeg -vf "drawtext=text='말까하고 있다?':fontsize=52:fontcolor=white" ...
```

돌아온 에러:

```
No such filter 'drawtext'
```

Homebrew로 설치한 ffmpeg 8.0.1이 `--enable-libfreetype` 없이 컴파일된 것이다. drawtext는 freetype 라이브러리가 있어야 동작하는데, 기본 brew 빌드에는 포함되지 않는다.

ffmpeg을 직접 다시 빌드하거나 다른 tap을 쓸 수도 있었지만, 배포 환경에서도 같은 문제가 반복될 수 있다. 그래서 방향을 바꿨다.

## 우회: 텍스트 → PNG → overlay

Pillow로 텍스트 이미지를 만들고, ffmpeg overlay 필터로 합성하는 방식이다.

```python
from PIL import Image, ImageDraw, ImageFont

def _make_text_image(text, width, height, font_size, out_path, ...):
    lines = text.replace("\\n", "\n").split("\n")
    # 각 줄의 크기 측정
    line_bboxes = [d.textbbox((0, 0), line, font=font) for line in lines]
    # 이미지 생성 후 텍스트 렌더링
    img.save(out_path)
```

이렇게 만든 PNG를 ffmpeg overlay 필터로 영상 위에 올린다.

```python
filter_parts = [f"[0:v]{scale_filter}[v0]"]
for i, (img_path, y) in enumerate(overlays):
    nxt = "out" if i == len(overlays) - 1 else f"v{i + 1}"
    filter_parts.append(f"[{prev}][{i + 1}:v]overlay=0:{y}[{nxt}]")
```

## 한국어 폰트 자동 감지

Pillow 기본 폰트는 한글을 지원하지 않는다. OS별로 폰트 경로가 다르므로, 실행 환경에 따라 자동으로 찾도록 했다.

```python
def _get_korean_font() -> str:
    candidates = {
        "Darwin": ["/System/Library/Fonts/AppleSDGothicNeo.ttc"],
        "Linux": [
            "/usr/share/fonts/truetype/nanum/NanumGothic.ttf",
            "/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc",
        ],
    }
    for path in candidates.get(platform.system(), []):
        if Path(path).exists():
            return path
    return ""
```

macOS는 시스템 폰트, Linux는 Nanum 또는 Noto를 순서대로 시도한다.

## ffmpeg 에러 메시지를 숨기지 말 것

초기에 ffmpeg 출력을 `subprocess.DEVNULL`로 버리고 있었다. 덕분에 오류가 나도 아무 메시지가 없어서 디버깅이 불가능했다.

```python
# 전: 에러를 삼켜버림
subprocess.run(cmd, check=True, stderr=subprocess.DEVNULL)

# 후: 에러를 캡처해서 출력
except subprocess.CalledProcessError as e:
    stderr_msg = e.stderr.decode(errors="replace") if e.stderr else ""
    print(f"오류: ffmpeg 실행 실패.\n{stderr_msg[-1000:]}")
```

`stderr=subprocess.PIPE`로 받고, 실패 시 마지막 1000자를 출력하도록 바꿨다. 이후 디버깅 속도가 확연히 달라졌다.

ffmpeg은 에러를 매우 장황하게 출력하는데, 핵심 원인은 항상 마지막 몇 줄에 있다.
