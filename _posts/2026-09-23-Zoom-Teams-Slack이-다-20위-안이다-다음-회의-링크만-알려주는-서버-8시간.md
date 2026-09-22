---
layout: post
title: "Zoom, Teams, Slack이 다 20위 안이다 — 다음 회의 링크만 알려주는 서버 8시간"
description: "회의 도구는 세 개인데 '지금 어디로 들어가야 하나'에 답하는 것은 없다. ICS 구독 하나로 다음 회의 링크를 5분 전에 밀어 주는 개인용 서버를 하루에 만든다."
categories: [Idea]
tags: [idea, sidejob, backend]
---

## 순위에서 읽은 것

Zoom Workplace, Microsoft Teams, Slack이 나란히 20위 안에 있다. 한 조직이 셋 중 하나만 쓰는 경우는 드물고, 외부 미팅까지 섞이면 접속 링크는 메일, 캘린더 초대 본문, 채널 메시지로 흩어진다. 회의를 여는 도구는 이렇게 검증됐는데, "지금 들어갈 방이 어디냐"에 한 줄로 답해 주는 것은 이 목록에 없다. 리멤버나 사람인처럼 정보를 모아 주는 앱은 있어도, 흩어진 회의 링크를 모으는 쪽은 비어 있다.

## 만들 것

캘린더 ICS 구독 URL만 등록하면 다음 회의의 제목, 시각, 접속 링크를 시작 5분 전에 Slack DM으로 보내 주는 개인용 서버다. 하루에 회의가 서너 개 잡히고 도구가 섞여 있는 사람이 대상이다. 화면은 없어도 되고, 만들더라도 다음 회의 한 건과 링크만 보여 주는 페이지면 충분하다.

## 8시간 범위

- ICS 구독 URL 파싱, 앞으로 24시간 안의 이벤트만 추출해 정규화
- location과 description 필드에서 Zoom, Teams, Meet 링크를 정규식으로 골라내기
- 5분 주기 스케줄러, 시작 5분 전 한 번만 발송하고 중복은 이벤트 UID로 차단
- Slack Incoming Webhook 발송, 슬래시 명령 /next로 다음 회의 즉시 조회

## 스택과 시간 배분

Python, ics 라이브러리, APScheduler, SQLite, Slack Incoming Webhook.

| 시간 | 할 일 |
| --- | --- |
| 0~2h | ICS 구독 가져오기, 이벤트 정규화, SQLite 스키마와 발송 이력 테이블 |
| 2~5h | 회의 링크 추출, 5분 주기 스케줄러, UID 기준 중복 발송 방지 |
| 5~8h | Slack 웹훅 발송 포맷, /next 슬래시 명령, 파싱 실패 로그와 재시도 |

```python
MEET = re.compile(
    r"https://\S*(?:zoom\.us/j/|teams\.microsoft\.com/l/meetup-join/|meet\.google\.com/)\S*"
)

def meeting_link(event):
    for field in (event.location or "", event.description or ""):
        m = MEET.search(field)
        if m:
            return m.group(0)
    return None
```


## 구현

아이디어를 실제로 만들면 여기에 남깁니다. 이 표는 사람이 직접 채웁니다.

| 항목 | 내용 |
| --- | --- |
| 저장소 | — |
| 메모 | — |


---

> **이 글의 본문은 자동 생성되었습니다.**
> 학습 기록을 남기기 위해 매일 정해진 시각에 스케줄러가 Claude Code를 호출해 작성하고, 사람의 검수 없이 그대로 게시합니다.
> 따라서 사실과 다른 내용이나 오래된 정보가 포함될 수 있습니다. 명령어와 설정은 실제 환경에서 검증한 뒤 사용하세요.
