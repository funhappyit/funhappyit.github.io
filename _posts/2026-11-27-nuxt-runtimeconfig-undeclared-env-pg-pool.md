---
layout: post
title: ".env에 적은 커넥션 풀 크기가 적용되지 않던 이유 - Nuxt runtimeConfig와 pg 타임아웃"
date: 2026-11-27 12:00:00 +0900
categories: [오류해결, Nuxt]
tags: [Nuxt, runtimeConfig, PostgreSQL, node-postgres, 커넥션풀, 환경변수]
---

DB 커넥션 획득 타임아웃을 조정하다가, `.env`에 적어 둔 풀 최대/최소 개수가 **처음부터 한 번도 적용되지 않았다**는 걸 알게 됐다.
Nuxt의 `runtimeConfig`가 환경변수를 어떻게 덮어쓰는지와, `pg` Pool의 `connectionTimeoutMillis`가 정확히 무엇을 재는지 정리한다.

## 문제 상황

- 모니터링 화면에서 가끔 `timeout exceeded when trying to connect` 오류가 났다. DB 자체는 정상이었다.
- `.env`에는 이렇게 적혀 있었다.

```env
NUXT_DB_POOL_MAX=3
NUXT_DB_POOL_MIN=0
```

- 풀 생성 코드는 runtimeConfig에서 값을 읽었다.

```ts
const config = useRuntimeConfig();
const pool = new Pool({
  // ...
  max: config.dbPoolMax,
  min: config.dbPoolMin,
  connectionTimeoutMillis: 5000,
  statement_timeout: 30000,
});
```

## 원인

### 1. `connectionTimeoutMillis`는 "풀 대기 시간"도 포함한다

이름만 보면 "TCP 연결을 맺는 데 걸리는 시간"처럼 보이지만, node-postgres에서는
**풀에서 커넥션을 하나 얻을 때까지의 전체 시간**이다. 풀이 다 차서 빈 커넥션을 기다리는 시간도 여기에 들어간다.

이 앱은 다음 조건이 겹쳤다.

- 모니터링 화면이 여러 API를 주기적으로 동시에 호출한다.
- 무거운 조회는 `statement_timeout`(30초)까지 커넥션을 쥐고 있을 수 있다.

그러면 풀이 금방 찬다. 뒤에 온 요청은 5초를 기다린 뒤 타임아웃으로 실패한다.

### 2. runtimeConfig에 선언하지 않은 키는 환경변수로 채워지지 않는다

Nuxt는 런타임에 `NUXT_` 접두사 환경변수로 runtimeConfig를 덮어쓰는데, **`nuxt.config.ts`에 이미 선언된 키만** 덮어쓴다.

```ts
// nuxt.config.ts
runtimeConfig: {
  dbHost: '',
  dbPort: '',
  // dbPoolMax, dbPoolMin 선언 없음 → NUXT_DB_POOL_MAX 무시
}
```

그래서 `config.dbPoolMax`은 `undefined`였다. node-postgres는 `max`가 없으면 기본값 10을 쓴다.
즉 `.env`의 `3`은 **장식**이었고, 실제로는 내내 10으로 돌고 있었다.

## 해결 방법

### 1. 타임아웃을 늘리고 환경변수로 뺀다

```ts
// nuxt.config.ts
runtimeConfig: {
  // 커넥션 획득 대기 시간(ms). 풀이 가득 찼을 때 기다리는 시간 포함
  dbAcquireTimeout: 15000,
  dbPoolMax: 10,
  dbPoolMin: 0,
}
```

```ts
const pool = new Pool({
  max: Number(config.dbPoolMax) || 10,
  min: Number(config.dbPoolMin) || 0,
  // 쿼리 타임아웃(30초)보다 짧게 유지
  connectionTimeoutMillis: Number(config.dbAcquireTimeout) || 15000,
  statement_timeout: 30000,
});
```

- 잠깐 몰리는 요청은 기다렸다가 처리하도록 15초로 늘렸다.
- 쿼리 타임아웃보다는 짧게 둔다. 커넥션을 기다리는 시간이 쿼리 실행 시간보다 길면 요청 하나가 지나치게 오래 걸린다.
- `Number(...) || 기본값`으로 감싸서, 값이 비었거나 숫자가 아니어도 기본값을 쓰게 했다.

### 2. 풀 크기 키를 runtimeConfig에 선언한다

위처럼 `dbPoolMax`, `dbPoolMin`을 선언하자 비로소 `NUXT_DB_POOL_MAX`이 적용된다.

## 결과

- 환경변수로 풀 크기와 타임아웃을 현장마다 조정할 수 있게 됐다.
- 순간적인 요청 몰림으로 생기던 커넥션 획득 실패가 줄었다.

### ⚠️ 주의: 풀 크기가 실제로는 줄어든다

2번을 고치는 순간 `.env`의 `3`이 **처음으로** 적용된다. 실질적인 풀 크기가 10에서 3으로 줄어든다는 뜻이다.
풀 고갈 대응으로 타임아웃을 늘렸는데, 풀 크기가 줄어서 고갈이 오히려 잘 생길 수 있다.
배포 전에 `.env` 값이 의도한 값인지 확인해야 한다. DB 서버의 `max_connections` 때문에 일부러 작게 잡은 게 아니라면 올리는 것을 검토한다.

### 정리

- Nuxt runtimeConfig: **선언한 키만 환경변수로 덮어쓴다.** `.env`에 추가할 때 `nuxt.config.ts`에도 기본값과 함께 선언한다.
- node-postgres `connectionTimeoutMillis`: 연결 수립 + 풀 대기 전체 시간이다.
- "설정이 안 먹던 걸 고치는" 변경은 실제 동작값이 바뀌는 변경이다. 그동안 돌던 값(기본값)과 새로 적용될 값을 비교해 본다.
