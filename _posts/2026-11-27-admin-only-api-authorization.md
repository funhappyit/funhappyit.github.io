---
layout: post
title: "인증 예외 목록에 들어간 계정 API - 관리자 전용 권한을 서버에서 막기"
date: 2026-11-27 11:00:00 +0900
categories: [오류해결, Nuxt]
tags: [Nuxt, Nitro, 권한, 인가, 보안, 미들웨어]
---

Nuxt(Nitro) 서버 미들웨어 하나로 인증을 처리하는 프로젝트에서, 관리 기능에 **권한 검사가 거의 없었다**는 걸 발견하고 막은 내용이다.
메뉴 숨김만으로는 권한이 되지 않는다는, 알고 있지만 자주 놓치는 이야기다.

## 문제 상황

세 가지가 겹쳐 있었다.

1. 계정 생성·수정 API가 **로그인 없이** 호출됐다.
2. 로그인한 일반 사용자가 설정 화면의 저장·삭제 API를 직접 호출하면 **그대로 실행**됐다.
3. 설정 메뉴는 일부 등급에서만 숨겨져 있어서, 등급 값이 비어 있는 사용자에게는 보였다.

## 원인

### 1. 인증 예외(public) 목록에 계정 API가 들어 있었다

```ts
// 인증 없이 통과시키는 경로
const publicPrefixes = [
  '/api/login',
  '/api/session/refresh',
  '/api/users/register', // ← 누구나 계정 생성
  '/api/users/edit',     // ← 누구나 아무 계정의 비밀번호·등급 수정
];
```

초기 개발 중 회원가입 화면을 열어 두려고 넣었던 것으로 보인다.
결과적으로 **로그인하지 않은 사람이 스스로 관리자 계정을 만들거나, 기존 계정의 등급을 올리는 것**이 가능했다.

### 2. 인증(authentication)만 있고 인가(authorization)가 없었다

미들웨어는 "세션이 있는가"만 확인했다. "이 사용자가 이 API를 호출해도 되는가"는 아무도 검사하지 않았다.

### 3. 메뉴 필터가 블랙리스트 방식이었다

```ts
// 일반 사용자(1)일 때만 관리자 메뉴 숨김
.filter(item => !(item.role === 'ADMIN' && grade === 1));
```

등급이 `undefined`이거나 새 등급(2, 3…)이 생기면 관리자 메뉴가 보인다.

## 해결 방법

### 1. 계정 API를 public 목록에서 뺀다

```diff
 const publicPrefixes = [
   '/api/login',
   '/api/session/refresh',
-  '/api/users/register',
-  '/api/users/edit',
 ];
```

### 2. 관리자 전용 경로 목록을 두고 미들웨어에서 403

```ts
// 관리자만 호출 가능한 쓰기 API. 조회(GET)는 일반 화면에서도 쓰므로 제외
const adminPrefixes = [
  '/api/users/register',
  '/api/accounts/',
  '/api/sites/save', '/api/sites/delete',
  '/api/devices/save', '/api/devices/delete',
  // ...
];

export default defineEventHandler(async event => {
  const pathname = event.path.split('?')[0]; // 쿼리스트링 제거 후 비교
  if (isPublic(pathname)) return;

  const session = await requireUserSessionOr401(event);

  const isAdminRoute = adminPrefixes.some(p => pathname.startsWith(p));
  if (isAdminRoute && Number(session.user?.grade ?? -1) !== 0) {
    throw createError({ statusCode: 403, message: '관리자만 사용할 수 있는 기능입니다.' });
  }
});
```

- 등급을 `?? -1`로 받아서 **값이 없으면 관리자가 아닌 것**으로 처리한다.
- 경로 비교 전에 `?` 뒤를 잘라 둔다.

### 3. 본인 정보 수정은 핸들러 안에서 검사

일반 사용자도 자기 이름·비밀번호는 바꿀 수 있어야 하므로, 수정 API는 관리자 전용 목록에 넣지 않고 핸들러에서 나눈다.

```ts
const session = await requireUserSession(event);
const isAdmin = Number(session.user?.grade ?? -1) === 0;

if (!isAdmin && session.user?.id !== body.userId) {
  throw createError({ statusCode: 403, message: '본인 정보만 수정할 수 있습니다.' });
}

// 권한·담당 범위 같은 필드는 관리자만 반영
if (isAdmin && typeof body.grade === 'number') update.grade = body.grade;
if (isAdmin && typeof body.scope === 'string') update.scope = body.scope.trim();
```

본인 계정이라도 **등급·담당 범위 필드를 같이 보내 권한을 올리는 것**을 막는 부분이 중요하다.

### 4. 메뉴 필터를 화이트리스트로

```diff
-.filter(item => !(item.role === 'ADMIN' && grade === 1));
+.filter(item => !(item.role === 'ADMIN' && grade !== 0));
```

관리자만 보이고 나머지는 전부 숨긴다. 이건 UI 편의일 뿐이고, 실제 차단은 2·3번의 서버 검사가 한다.

## 결과

| 요청 | 이전 | 이후 |
| --- | --- | --- |
| 비로그인 계정 생성/수정 | 성공 | 401 |
| 일반 사용자의 설정 저장·삭제 | 성공 | 403 |
| 일반 사용자의 본인 비밀번호 변경 | 성공 | 성공 |
| 일반 사용자의 본인 등급 변경 | 성공 | 무시됨 |
| 등급 미확인 사용자의 관리자 메뉴 | 보임 | 숨김 |

### 정리

- **인증 예외 목록은 주기적으로 다시 본다.** 개발 편의로 넣은 경로가 그대로 남기 쉽다.
- 권한 판정은 "아니면 막는다"(`!== ADMIN`)로 쓴다. "이것이면 막는다"(`=== USER`)는 새 값이 생길 때 뚫린다.
- 같은 API라도 **필드 단위 권한**이 필요한 경우가 있다(본인 수정은 허용, 등급 필드는 관리자만).
- 이 변경 이후 계정 생성은 관리자만 할 수 있다. 자가 회원가입 화면이 있었다면 같이 정리해야 한다.
