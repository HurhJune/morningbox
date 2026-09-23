---
name: morning-board
description: 연합뉴스 RSS에서 관심 뉴스 3건, Open-Meteo에서 오늘 날씨를 가져와 news.md/weather.md/weather_alert.md에 저장하고, DESIGN.md 기준으로 dashboard.html을 다시 만들어 브라우저로 띄운다. 사용자가 /morning-board 를 칠 때만 실행한다.
disable-model-invocation: true
---

# morning-board

아래 네 단계를 순서대로 모두 수행한다. 작업 폴더는 `E:\workspace\seintu\morningbox`이다.
1·2단계는 서로 독립이므로 한 메시지에서 동시에 호출한다.

## 비밀값 규칙

- 이 스킬의 1~4단계는 **키가 필요 없다** (연합뉴스 RSS, Open-Meteo 모두 무료·무인증).
- 키가 필요한 일(텔레그램 전송 등)을 덧붙여 시키면, 값을 **`E:\workspace\seintu\morningbox\.env`에서 읽는다** (`TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`).
- 토큰 값을 화면, 파일, 로그, 이 스킬 문서 어디에도 출력하거나 적지 않는다. 오류 메시지에 토큰이 섞이면 가린 뒤 보여준다.

## 1단계: 뉴스 → news_raw.md, news.md

`https://www.yna.co.kr/rss/news.xml`을 가져온다. 응답은 UTF-8로 읽고, `pubDate`는 `ddd, dd MMM yyyy HH:mm:ss zzz` 형식이다.

1. 전체 기사를 **최신순**으로 정렬해 `news_raw.md`에 표로 저장한다 (`# | 시각 | 제목 | 링크`). 파일 맨 위에 가져온 시각과 출처 URL을 적는다.
2. 제목에 **경제 관련 낱말**이 든 기사를 고른다. 낱말 목록은 다음과 같다.
   `경제|주식|부동산|증시|코스피|코스닥|주가|금리|환율|물가|집값|아파트|전세|분양|주택|투자|기업|수출|반도체|GDP|한은|은행`
3. 제목이 `[부고]`, `[인사]`로 시작하는 기사는 뺀다. (회사 이름 때문에 걸리는 기사다)
4. 걸린 기사 중 **최신순 3건**을 고른다. 3건이 안 되면 가장 최근 기사로 채운다.
5. 고른 3건의 **본문을 직접 열어 읽고** 한 줄 정리를 쓴다. 제목만 보고 짐작해서 쓰지 않는다.
6. `news.md`를 아래 형식으로 **덮어쓴다**.

```markdown
# 관심 뉴스 3건

- 원본: 연합뉴스 RSS (https://www.yna.co.kr/rss/news.xml), <수집 시각> KST 수집
- 고른 기준: 제목에 경제 관련 낱말이 들어간 기사 중 최신순 3건. 부고·인사 기사는 뺐다.
- 한 줄 정리는 기사 본문을 읽고 썼다.

## 1. <제목>
- 한 줄 정리: <본문을 읽고 쓴 한 줄>
- 링크: <URL>

## 2. …
## 3. …
```

기사 시각(`MM-DD HH:mm`)은 4단계에서 쓰므로 따로 기억해 둔다.

## 2단계: 날씨 → weather.md

Open-Meteo에서 오늘 예보를 가져온다. 기본 지역은 **서울특별시**(시청 좌표)다. 사용자가 다른 지역을 말하면 그 좌표로 바꾼다.

```
https://api.open-meteo.com/v1/forecast?latitude=37.5665&longitude=126.9780&daily=temperature_2m_max,temperature_2m_min,precipitation_probability_max&timezone=Asia%2FSeoul&forecast_days=1
```

- 호출한 시각(KST, 초 단위)을 확인 시각으로 기록한다.
- `weather.md`를 아래 형식으로 **덮어쓴다**.

```markdown
# 오늘 날씨 예보: <지역>

| 항목 | 값 |
|---|---|
| 날짜 | YYYY-MM-DD (요일) |
| 최고기온 | ○ °C |
| 최저기온 | ○ °C |
| 비 올 확률 (하루 중 최대) | ○ % |

- 확인 시각: YYYY-MM-DD HH:mm:ss (KST, UTC+9)
- 출처: Open-Meteo Forecast API (https://open-meteo.com)
  - 요청: `<위 URL>`
  - 요청 좌표는 <지역>(<lat>, <lon>)이고, API가 가장 가까운 격자점(<응답 latitude>, <응답 longitude>, 해발 <elevation>m)으로 맞춰 응답했다.
```

호출이 실패하면 날씨 부분은 멈추고, 뉴스만으로 3·4단계를 진행하되 실패 사실을 알린다.

## 3단계: 기준과 견주기 → weather_alert.md

| 기준 | 조건 |
|---|---|
| 비 올 확률 | 60% 이상 |
| 최저기온 | 10℃ 이하 |

하나라도 해당하면 넘은 것이다. 넘든 안 넘든 **반드시 한 줄을 만든다**.

```
[YYYY-MM-DD HH:mm] <지역> · 최고 ○℃ 최저 ○℃ · 비 ○% · <판정> · 확인함
```

- `<판정>`은 넘으면 `⚠ 기준 넘음`, 아니면 `기준 미달`이다. 숫자는 API 값 그대로 쓴다.
- `weather_alert.md`를 덮어쓰되 **첫 줄은 반드시 위의 한 줄**이어야 한다. 그 아래에 기준 비교표와 원본 값 출처를 적는다 (기존 파일 형식 그대로).

## 4단계: dashboard.html 다시 만들기 → 화면 띄우기

- 디자인 기준 파일: `docs/stitch_main_dashboard/DESIGN.md`와 `code.html` (색 토큰, 글꼴, 레이아웃 규칙)
- **기존 `dashboard.html`이 있으면 구조와 CSS는 그대로 두고 데이터만 갈아끼운다.** 없으면 DESIGN.md를 읽고 새로 만든다. (w2 폴더의 `dashboard.html`을 본보기로 삼아도 된다)

갈아끼우는 값은 다음과 같다.

| 위치 | 값 |
|---|---|
| 상단 날짜 | 오늘 날짜 |
| `.weather` 의 `data-rain`, `data-min` | 비 올 확률, 최저기온 (강조 판정에 쓰인다) |
| `.place` | 지역 이름 |
| `.temps`, `.rain .value`, 막대 `width` | 최고/최저 기온, 비 올 확률 |
| `.alert-line .text` | weather_alert.md 첫 줄 그대로 |
| `#verdict` | `기준 미달` (넘은 날은 스크립트가 `⚠ 기준 넘음`으로 바꾼다) |
| `.news` 목록 | 기사 3건의 시각·제목(링크)·한 줄 정리 |
| `footer` | 날씨 확인 시각, 뉴스 수집 시각 |

지켜야 할 디자인 규칙:

- 색 토큰은 DESIGN.md 값을 쓴다. 보조 글자색은 `--outline` (#6c7b6c), 기준 넘음 강조는 error 계열(`--error-container` #ffdad6, `--error` #ba1a1a)이다.
- 1px 실선 테두리와 구분선을 쓰지 않는다. 경계는 배경색 차이로만 만든다.
- 제목은 링크(`<a>`)로 걸고 `target="_blank" rel="noopener"`를 붙인다.
- 휴대폰(600px 이하)에서는 날씨 상자가 한 칸으로 쌓이게 한다.
- 기사 제목·요약에 `&`, `<`, `>`가 있으면 HTML 이스케이프한다.

만든 뒤:

1. headless Edge로 확인한다. Edge는 창을 500px보다 좁게 만들지 못하므로 휴대폰 확인은 500px로 한다.
   ```powershell
   $e="${env:ProgramFiles(x86)}\Microsoft\Edge\Application\msedge.exe"; $sp=$env:TEMP; Start-Process -FilePath $e -ArgumentList '--headless=new','--disable-gpu','--hide-scrollbars','--virtual-time-budget=4000','--window-size=500,1250',"--screenshot=$sp\mb_m.png",'file:///E:/workspace/seintu/morningbox/dashboard.html' -Wait -NoNewWindow
   ```
   찍은 png를 직접 읽어 레이아웃이 깨지지 않았는지 눈으로 확인한다.
2. 기본 브라우저로 띄운다.
   ```powershell
   Start-Process 'E:\workspace\seintu\morningbox\dashboard.html'
   ```

## 끝나고 보고할 것

- 고른 기사 3건의 제목과 시각, 그리고 몇 건이 낱말에 걸렸는지 (최근 기사로 채웠다면 그 사실)
- 날씨 값과 판정, 판정 근거 (비 ○% / 60%, 최저 ○℃ / 10℃)
- 화면을 띄웠다는 것과, headless 확인에서 본 문제
- 어느 단계가 실패했다면 그 단계와 이유

## 알아둘 것

- `/weather-check` 스킬은 **성북구 돈암동** 기준으로 `w2` 폴더의 `weather.md`, `weather_alert.md`를 덮어쓴다. 이 스킬(morningbox·서울)과는 폴더가 다르므로 서로 덮어쓰지 않는다. 실행 뒤 어느 폴더·지역 기준인지 보고에 밝힌다.
- `dashboard.html`의 값은 실행 시점에 박아 넣은 것이다. 파일을 나중에 다시 열어도 내용은 갱신되지 않는다.
- 강조 화면을 확인하려면 주소 끝에 `#preview-over`를 붙인다.
