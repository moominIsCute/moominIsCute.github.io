---
title: "[인턴 활동] AI가 드라마를 보고 편집점을 잡는다"
date: 2026-03-12 00:00:00 +0900
categories: [인턴 활동]
tags: [AI, Gemini, 멀티모달, 쇼츠, 영상분석]
---

숏폼 편집자가 드라마 한 편을 보고 하는 일이 있다. 어디서 웃기고, 어디서 긴장되고, 어디서 "이거다" 싶은 장면이 나오는지 감으로 포착하는 것. 그걸 AI한테 시킬 수 있을까?

이번 프로젝트의 출발점은 그 질문이었다.

## 프로젝트 구조

툴은 단순하다. Gemini API에 영상 파일을 올리면, 멀티모달 모델이 시각/오디오/대사를 통합 분석해서 편집점 타임스탬프를 뽑아준다. 그 JSON을 받아 ffmpeg으로 영상을 잘라 붙이면 쇼츠 완성.

```
영상 파일 → Gemini API (멀티모달 분석) → clip_json → ffmpeg → 쇼츠
```

의존성은 세 줄이다.

```
google-genai
python-dotenv
Pillow
```

## Gemini에게 "편집점 뽑아줘"를 가르치는 법

단순히 "재밌는 장면 알려줘"라고 하면 안 된다. 모델이 무엇을 기준으로 판단하는지 명확히 정의해줘야 한다.

프롬프트에 넣은 분석 항목들:

| 항목 | 설명 |
|------|------|
| video_summary | 핵심 서사 80자 이내 요약 |
| emotion_curve | 타임스탬프별 감정 변화 (intensity 0~1) |
| tension_score | 평균/피크 긴장도 및 해당 시간대 |
| highlights | 바이럴 가능성 높은 구간 3~5개 |
| humor_points | 유머 발생 지점 및 웃음 강도 |
| love_points | 플러팅 발생 지점 및 멘트 |

핵심은 "느낌"이 아니라 수치와 타임스탬프로 받는 것이다. 그래야 ffmpeg이 실제로 자를 수 있다.

## clip_json — AI와 ffmpeg 사이의 계약

Gemini 응답 마지막에 반드시 아래 형식의 JSON 블록을 출력하도록 프롬프트에 강제했다.

```json
{
  "title": "자기야를 부르던 도라미\n다정한 안녕으로 끝났다",
  "clips": [
    {"start": 0.0, "end": 5.0, "subtitle": "말까하고 있다?", "caption": "안녕은 다정했고, 결국 무희는 울었다"},
    {"start": 667.0, "end": 690.0, "subtitle": "나 진짜 갈 거야", "caption": "돌아서는 그의 등이 낯설었다"}
  ]
}
```

시간 단위는 반드시 순수 초(seconds)로만 받는다. 모델이 "11분 7초"를 `11.07`로 출력하는 경우가 있어서, 변환 로직도 넣었다.

```python
def mmss_to_seconds(ts: float) -> float:
    """11.07 형식(분.초)을 순수 초(667.0)로 변환."""
    minutes = int(ts)
    centiseconds = round((ts - minutes) * 100)
    if 0 <= centiseconds <= 59:
        return minutes * 60 + centiseconds
    return ts
```

## 핵심 호출부

```python
client = genai.Client(api_key=API_KEY)
MODEL = os.getenv("GEMINI_MODEL_NAME", "gemini-2.0-flash")

def ask_about_video(video_file, question: str, system_prompt: str) -> str:
    response = client.models.generate_content(
        model=MODEL,
        contents=[video_file, question],
        config={"system_instruction": system_prompt},
    )
    return response.text
```

영상 파일을 Gemini Files API로 업로드한 뒤, 멀티모달 입력으로 넘기면 된다. 분석 결과가 텍스트로 돌아오고, 그 안에서 clip_json 블록을 파싱해 편집점 리스트를 뽑는다.

AI가 드라마를 보고, 어디서 잘라야 할지 판단하는 파이프라인의 첫 번째 뼈대가 완성됐다.
