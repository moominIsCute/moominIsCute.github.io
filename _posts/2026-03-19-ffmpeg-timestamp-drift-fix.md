---
title: "[인턴 활동] FFmpeg 롱폼 분할 시 타임스탬프 밀림 원인과 해결"
date: 2026-03-19 00:00:00 +0900
categories: [인턴 활동]
tags: [ffmpeg, VFR, CFR, 청크, 타임스탬프, 영상처리]
---

1시간짜리 영상을 AI에게 그대로 던지면 비용과 속도 모두 감당이 안 된다. 그래서 480p 프록시로 줄이고 5분(300초) 단위 청크로 잘라 분석하는 구조를 만들었다. 첫 번째 청크는 자막이 칼같이 맞았다. 두 번째 청크부터 1~2초씩 밀리더니, 뒤로 갈수록 3~4초까지 벌어졌다.

## 원인 1 — 가변 프레임(VFR)의 누적 오차

프록시를 만들 때 속도를 위해 `fps=15`로 강제 압축했다. 원본이 정확히 30.00fps면 문제가 없다. 문제는 29.97fps 같은 가변 프레임(VFR) 영상이다. 프레임이 미세하게 생략되면서 청크가 뒤로 갈수록 오차가 야금야금 누적된다.

해결책은 FFmpeg 옵션 하나였다.

```bash
# 기존
ffmpeg -i input.mp4 -vf scale=854:480 -r 15 proxy.mp4

# 수정
ffmpeg -i input.mp4 -vf scale=854:480 -r 15 -vsync cfr proxy.mp4
```

`-vsync cfr`은 프레임을 건너뛰지 않고 일정한 간격(Constant Frame Rate)으로 고정한다. 이것만으로 후반부 누적 오차가 0에 수렴했다.

## 원인 2 — 오프셋 덧셈 로직 버그

청크를 자를 때 자연스럽게 이어붙이기 위해 3초 오버랩을 뒀다. 두 번째 청크는 300초가 아니라 297초부터 잘렸다.

그런데 파이썬 코드는 이랬다.

```python
# 문제 코드
for i, chunk in enumerate(chunks):
    offset = i * 300.0  # 실제 분할 위치와 무관하게 300씩 더함
    adjusted_time = chunk_relative_time + offset
```

코드는 300.0을 더하고, 영상은 297초부터 시작된다. 3초 간극이 매 청크마다 쌓였다.

```python
# 수정 코드
for chunk in chunks:
    actual_cut_offset = max(0, chunk.start_sec)  # 실제 분할된 위치 사용
    adjusted_time = chunk_relative_time + actual_cut_offset
```

`chunk.start_sec`에는 오버랩이 반영된 실제 분할 시작 시간이 들어있다. 이 값을 그대로 쓰면 코드와 영상의 물리적 위치가 정확히 일치한다.

## 결과

두 수정을 동시에 적용하자 1시간짜리 영상의 마지막 청크까지 일치했다. 오차의 원인은 AI가 아니었다. 전처리 옵션과 오프셋 계산 로직이었다.
