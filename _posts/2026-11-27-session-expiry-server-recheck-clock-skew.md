---
layout: post
title: "클라이언트 만료 타이머가 멀쩡한 세션을 끊던 문제 - 서버 재확인과 시계 차이 보정"
date: 2026-11-27 10:00:00 +0900
categories: [오류해결, Nuxt]
tags: [Nuxt, nuxt-auth-utils, 세션, 인증, 시계차이, Vue]
---

세션 쿠키 기반 인증을 쓰는 Nuxt SPA에서, 사용자가 **계속 작업 중인데도** 만료 시각이 되면 로그인 화면으로 튕기는 문제가 있었다.
고치다 보니 PC 시계가 서버와 다를 때 생기는 문제까지 같이 엮여 있어서 둘을 함께 정리한다.

## 문제 상황

- 세션 유휴 만료는 1시간.
- 클라이언트는 세션 정보의 `expiresAt`을 보고 `setTimeout`으로 만료 시점에 로그아웃 처리를 한다.
- 화면에서 조회·차트 조작을 계속하고 있었는데, 로그인 1시간 뒤 정확히 로그아웃된다.
- 로그아웃 직후 다시 API를 쳐 보면 **서버 세션은 살아 있다.**

## 원인

### 1. 서버는 연장했는데 클라이언트는 모른다

서버 미들웨어는 인증이 필요한 API가 호출될 때마다 세션을 연장한다.

```ts
// server/middleware/auth.ts (예시)
export default defineEventHandler(async event => {
  if (!needsAuth(event.path)) return;

  const session = await requireUserSession(event);
  if (isExpired(session)) throw createError({ statusCode: 401 });

  // API 호출마다 expiresAt, 쿠키 Max-Age 연장
  await touchSession(event, session);
});
```

그런데 클라이언트의 `useUserSession().session`은 **`fetch()`를 다시 호출해야만** 갱신된다.
일반 API 응답으로는 바뀌지 않는다.

```
로그인        API 호출들(서버 만료 계속 연장)          클라 타이머
  |──────────────────────────────────────────────────────|
  expiresAt = 로그인+1h (클라는 이 값 그대로)          → 즉시 로그아웃 ✗
```

라우트 미들웨어도 같은 구조였다. 클라이언트 기준으로 남은 시간이 0 이하면 서버에 묻지 않고 바로 `clear()` 후 로그인으로 보냈다.

### 2. 만료 계산에 PC 시계를 쓴다

`expiresAt`은 **서버 시계** 기준 시각인데, 남은 시간은 `expiresAt - Date.now()`, 즉 **PC 시계**로 계산했다.
현장 PC는 시계가 몇 분씩 틀린 경우가 흔하다.

| PC 시계 | 결과 |
| --- | --- |
| 서버보다 느림 | 타이머가 늦게 울림 → 서버가 401을 주므로 큰 문제는 없음 |
| 서버보다 빠름 | 타이머가 **서버 만료 전에** 울림 |

## 해결 방법

### 1. 타이머가 울리면 서버에 먼저 물어본다

즉시 로그아웃하지 않고 세션 갱신 API를 호출해 본다.

```ts
const RETRY_MS = 30_000;

const confirmWithServer = async (isRetry = false) => {
  clearLogoutTimer();
  try {
    await refreshOnServer();          // POST /api/session/refresh
    await fetchSession();             // 클라 세션 상태 동기화 → watch가 타이머 다시 건다
  }
  catch (e: any) {
    const status = e?.statusCode ?? e?.response?.status;
    // 네트워크 오류·5xx는 한 번만 재시도
    if (status !== 401 && !isRetry) {
      logoutTimer = setTimeout(() => confirmWithServer(true), RETRY_MS);
      return;
    }
    await logout();
  }
};
```

- 서버 세션이 유효하면 연장하고 계속 사용한다.
- 401이면 진짜 만료이므로 로그아웃한다.
- 일시적인 네트워크 끊김으로 쫓겨나지 않도록 1회 재시도한다.

**방치된 세션은 여전히 정상 만료된다.** 재확인은 클라이언트 만료 시각에 약간의 여유(5초)를 더한 뒤에 하므로,
그 사이 API 호출이 없었다면 서버 세션도 이미 만료되어 401이 온다.

라우트 미들웨어도 "이미 만료" 분기를 없애고, "남은 시간이 절반 이하" 분기에 합쳐 서버 갱신을 시도하게 했다. 실패하면 catch에서 로그인으로 보낸다.

### 2. 갱신 응답으로 서버 시각을 추정한다

1번만 넣으면 오히려 더 나빠지는 경우가 있다. **PC 시계가 빠른 경우**다.

1. 타이머가 서버 만료보다 먼저 울린다.
2. 서버에 확인하면 아직 유효하므로 성공한다.
3. 그런데 갱신 호출 **자체가** 세션을 연장한다.
4. 다음 타이머도 또 일찍 울리고, 또 연장된다.

결국 아무도 안 쓰는 세션이 **영원히 만료되지 않는다.**

그래서 갱신 API가 `expiresAt`과 `maxAgeSec`를 함께 돌려주게 하고, 이 둘로 응답 시점의 서버 시각을 역산했다.

```ts
let clockOffset = 0; // 서버 시각 - PC 시각 (ms)

const refreshOnServer = async () => {
  const res = await $fetch<{ expiresAt?: string; maxAgeSec?: number }>(
    '/api/session/refresh', { method: 'POST' },
  );
  const exp = res?.expiresAt ? new Date(res.expiresAt).getTime() : NaN;
  const maxAge = Number(res?.maxAgeSec);
  if (!Number.isNaN(exp) && maxAge > 0) {
    const serverNowAtResponse = exp - maxAge * 1000;
    clockOffset = serverNowAtResponse - Date.now();
  }
  return res;
};

const serverNow = () => Date.now() + clockOffset;
```

타이머는 서버 시각 기준으로 건다.

```ts
const GRACE_MS = 5_000;
const MIN_INTERVAL_MS = 60_000;
let lastOkAt = 0;

watch(expiresAtMs, exp => {
  clearLogoutTimer();
  if (!exp) return;

  let delay = exp - serverNow() + GRACE_MS;
  // 보정이 안 된 상태에서 재확인 직후 또 재확인하는 루프 방지
  if (Date.now() - lastOkAt < MIN_INTERVAL_MS) {
    delay = Math.max(delay, MIN_INTERVAL_MS);
  }
  logoutTimer = setTimeout(() => confirmWithServer(), Math.max(0, delay));
}, { immediate: true });
```

주기적으로 보내는 heartbeat도 같은 `refreshOnServer()`를 쓰게 해서 offset이 자주 갱신되게 했다.

## 결과

| 항목 | 이전 | 이후 |
| --- | --- | --- |
| 만료 판단 | 클라이언트 단독 | 클라이언트가 계기를 주고 서버가 최종 판단 |
| API 호출로 연장된 세션 | 잘못 로그아웃됨 | 유지 |
| 네트워크 순단 | 즉시 로그아웃 | 1회 재시도 |
| PC 시계 차이 | 오판 | 서버 시각 기준으로 보정 |
| 방치된 세션 | 만료 | 그대로 만료 |

### 남은 한계

- 첫 갱신 전까지는 `clockOffset`이 0이라 보정되지 않는다.
- 라우트 미들웨어는 아직 `Date.now()`를 그대로 쓴다. PC 시계가 빠르면 갱신 요청을 조금 더 자주 보낼 뿐이고, 판단은 서버가 하므로 기능상 문제는 없다.

### 정리

- **클라이언트 타이머는 "확인할 시점"을 알려 주는 용도로만 쓰고, 만료 판정은 서버에 맡긴다.**
- 서버가 준 절대 시각을 클라이언트 시계로 비교할 때는 시계 차이를 의심한다.
- 확인 요청이 상태를 바꾸는 요청(여기서는 세션 연장)이면, 확인 시점이 틀렸을 때 무한 연장이 될 수 있다.
