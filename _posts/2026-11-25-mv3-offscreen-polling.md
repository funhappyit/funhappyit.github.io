---
layout: post
title: "Manifest V3에서 4초 폴링이 불가능한 이유 - offscreen document로 루프 살리기"
date: 2026-11-25 14:50:00 +0900
categories: [오류해결, Chrome Extension]
tags: [Chrome Extension, Manifest V3, Service Worker, JavaScript]
---

Chrome 확장에서 몇 초 간격으로 페이지를 조회하는 기능을 만들려 했다.
Manifest V2 시절이라면 `setInterval` 한 줄로 끝날 일인데, **V3에서는 그 루프가 30초쯤 뒤에 조용히 멈춘다.**

## 문제 상황

MV2의 background page는 브라우저가 켜져 있는 동안 계속 살아 있는 문서였다. 타이머를 걸어 두면 그대로 돌았다.

MV3에서 background는 **service worker**로 바뀌었다. service worker는 이벤트에 반응해 깨어났다가, 할 일이 없으면 종료된다. 유휴 상태가 **약 30초** 지속되면 내려간다.

문제는 `setTimeout` / `setInterval`이 service worker를 살려 두는 근거가 되지 못한다는 점이다. 타이머가 걸려 있어도 워커는 종료되고, 타이머는 워커와 함께 사라진다.

```js
// service worker 안에서 — 동작하지 않는다
setInterval(async () => {
  await check();   // 처음 몇 번은 돌다가 워커가 내려가면서 멈춘다
}, 4000);
```

더 나쁜 건 **조용히** 멈춘다는 것이다. 에러도 없고, 콘솔에도 아무것도 안 남는다. 확장을 열어 보면 멀쩡해 보인다.

## chrome.alarms로는 왜 안 되나

MV3에서 주기 작업의 공식 답은 `chrome.alarms`다. 알람이 울리면 워커가 깨어나므로 타이머와 달리 종료에 영향을 받지 않는다.

```js
chrome.alarms.create('poll', { periodInMinutes: 1 });
chrome.alarms.onAlarm.addListener(check);
```

그런데 **`periodInMinutes`의 최소값이 1분**이다. 더 짧게 주면 1분으로 올라간다.

1분 간격이면 되는 작업에는 이게 정답이다. 하지만 수 초 단위로 봐야 하는 작업에는 쓸 수 없다.

## 선택지 비교

| 방법 | 최소 주기 | 창을 닫아도 유지 | 비고 |
| --- | --- | --- | --- |
| service worker + `setInterval` | — | ✗ | 약 30초 후 멈춤 |
| `chrome.alarms` | 1분 | ✓ | 짧은 주기 불가 |
| 확장 페이지(탭)에서 폴링 | 제한 없음 | ✗ | 탭을 닫으면 끝 |
| **offscreen document** | 제한 없음 | ✓ | MV3의 사실상 표준 해법 |

## 해결: offscreen document

`chrome.offscreen`은 **화면에 보이지 않는 DOM 문서**를 띄우는 API다. 원래는 DOM이 필요한 작업(클립보드, 오디오 재생, HTML 파싱 등)을 service worker 대신 처리하라고 만든 것인데, 이 문서는 **service worker와 생존 주기가 분리되어 있다.** 워커가 내려가도 offscreen 문서는 남는다.

즉 폴링 루프를 여기에 넣으면 된다.

### manifest

```json
{
  "manifest_version": 3,
  "minimum_chrome_version": "116",
  "permissions": ["offscreen", "alarms", "storage"],
  "background": {
    "service_worker": "src/background/service-worker.js",
    "type": "module"
  }
}
```

### service worker — 생성과 감시

```js
const OFFSCREEN_PATH = 'src/offscreen/offscreen.html';

async function ensureOffscreen() {
  if (await chrome.offscreen.hasDocument()) {
    return;
  }

  await chrome.offscreen.createDocument({
    url: OFFSCREEN_PATH,
    reasons: ['DOM_PARSER'],
    justification: '주기적 조회와 HTML 응답 파싱',
  });
}
```

`reasons`에는 실제로 하는 일을 적는다. 응답이 HTML이라 `DOMParser`로 파싱해야 한다면 `DOM_PARSER`가 그대로 명분이 된다. 명분 없이 생존 수단으로만 쓰는 건 피하는 게 좋다.

offscreen 문서도 브라우저 사정으로 사라질 수 있으므로, **알람으로 생존을 확인하고 없으면 다시 만든다.** 여기서는 1분 주기면 충분하다. 폴링 자체가 아니라 감시만 하면 되기 때문이다.

```js
chrome.alarms.create('offscreen-watchdog', { periodInMinutes: 1 });

chrome.alarms.onAlarm.addListener(async (alarm) => {
  if (alarm.name !== 'offscreen-watchdog') {
    return;
  }

  const targets = await loadTargets();

  // 감시할 대상이 없으면 문서를 닫아 자원을 돌려준다
  if (targets.length === 0) {
    if (await chrome.offscreen.hasDocument()) {
      await chrome.offscreen.closeDocument();
    }
    return;
  }

  await ensureOffscreen();
});
```

### offscreen document — 실제 루프

```html
<!doctype html>
<html lang="ko">
  <head><meta charset="utf-8" /></head>
  <body>
    <script type="module" src="./poller.js"></script>
  </body>
</html>
```

```js
let timer = null;
let stopped = false;

async function tick() {
  if (stopped) {
    return;
  }

  await runOnce();
  timer = setTimeout(tick, nextDelay());
}
```

여기서는 `setInterval`이 아니라 **`setTimeout` 재귀**를 쓴다. 한 번의 조회가 간격보다 오래 걸려도 요청이 겹쳐 쌓이지 않고, 매번 간격을 다르게 줄 수 있기 때문이다.

## 폴링을 붙일 때 같이 넣어야 하는 것

루프가 도는 것과 **써도 되는 루프**인 것은 다르다. 세 가지를 함께 넣었다.

### 간격에 지터를 섞는다

정확히 4.000초마다 날아가는 요청은 패턴이 또렷하다. 매 요청마다 간격을 흔들어 준다.

```js
function nextDelay(base = 4000, jitter = 1500) {
  const offset = (Math.random() * 2 - 1) * jitter;
  return clamp(base + offset);
}
```

### 대상이 여럿이어도 직렬로 보낸다

감시 대상이 5개라고 해서 동시에 5개를 쏘면 안 된다. 전역 큐를 두고 한 번에 하나씩, 대상 사이에 최소 간격을 둔다. 그 결과 대상이 많아지면 **대상별 실제 주기가 늘어나므로**, 화면에는 설정값이 아니라 실측 주기를 보여 주는 편이 정직하다.

### 실패하면 물러난다

연속 실패마다 간격을 2배씩 늘리고 상한을 둔다. 성공하면 초기화한다.

```js
function backoffInterval(base, failures) {
  if (failures <= 0) {
    return base;
  }
  return Math.min(base * 2 ** failures, MAX_BACKOFF);
}
```

백오프에는 지터를 섞지 않았다. 지터는 패턴을 숨기려는 것이고 백오프는 물러나려는 것이라, 목적이 다르다.

인증이 풀린 경우(로그인 필요 응답)는 백오프 대상이 아니라 **중단** 대상이다. 재시도해도 상황이 바뀌지 않고, 반복하면 차단 사유만 쌓인다.

## 정리

- MV3 service worker는 유휴 30초 후 종료되므로 그 안의 타이머는 믿을 수 없다. 조용히 멈춘다는 점이 특히 위험하다.
- `chrome.alarms`는 최소 1분이라 초 단위 주기에 못 쓴다.
- `chrome.offscreen`으로 띄운 문서는 워커와 생존 주기가 분리되어 있어 긴 루프를 담을 수 있다. 단, 문서가 사라질 경우를 대비해 알람 watchdog을 함께 둔다.
- 폴링을 만들 때 지터·직렬 큐·백오프는 선택이 아니라 기본 구성으로 본다.
