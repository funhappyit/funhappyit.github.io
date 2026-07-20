---
layout: post
title: "mini-homepage 관리자 로그인 점검 기록 — 토큰 갱신과 포트 충돌"
date: 2026-07-20 14:00:00 +0900
categories: [사이드프로젝트, mini-homepage]
tags: [Spring Boot, Vue, JWT, Pinia, Axios, 관리자 인증, 트러블슈팅]
---

`mini-homepage`의 1단계 목표인 관리자 인증을 다시 점검했다. 로그인 화면과 JWT 발급 API는 이미 있었지만, 브라우저에서 실제로 로그인하는 과정에서는 아직 해결해야 할 문제가 남아 있다. 이번 글은 완료 보고가 아니라, 현재까지 반영한 내용과 막힌 지점을 남기는 작업 기록이다.

## 이번에 보강한 인증 흐름

관리자 계정은 회원가입 없이 환경 변수로만 생성한다.

```powershell
$env:ADMIN_SEED_USERNAME = "admin"
$env:ADMIN_SEED_PASSWORD = "admin1234"
```

로그인에 성공하면 Access/Refresh 토큰을 Pinia 스토어와 브라우저 저장소에 보관한다. 여기에 Axios 응답 인터셉터를 추가해, 일반 API 요청이 `401`을 받았을 때 Refresh 토큰으로 토큰을 한 번 재발급한 뒤 원래 요청을 재시도하도록 만들었다.

중요한 점은 여러 요청이 동시에 `401`을 받아도 Refresh API가 여러 번 호출되지 않도록, 하나의 갱신 Promise를 공유하게 한 것이다. 갱신까지 실패하면 저장된 토큰을 지우고 `/login`으로 이동한다.

```ts
// 핵심 흐름: 동시에 발생한 401은 하나의 갱신 요청을 공유한다.
refreshPromise ??= authStore.refresh()
await refreshPromise

return apiClient(originalRequest)
```

`/admin` 라우터 가드는 유지했다. 비로그인 사용자는 `/login`으로 보내고, 로그인 성공 시에는 관리자 화면으로 이동한다.

## 백엔드 포트 충돌 대응

기존 `8080` 포트가 이미 사용 중이라 Spring Boot 애플리케이션이 시작하지 못했다.

```text
Web server failed to start. Port 8080 was already in use.
```

백엔드 기본 포트를 `8081`로 옮기고, 프런트 Axios 기본 API 주소도 함께 변경했다.

```yaml
server:
  port: ${SERVER_PORT:8081}
```

```ts
baseURL: import.meta.env.VITE_API_BASE_URL ?? 'http://localhost:8081'
```

따라서 로컬 실행은 같은 PowerShell 창에서 다음처럼 진행한다.

```powershell
$env:ADMIN_SEED_USERNAME = "admin"
$env:ADMIN_SEED_PASSWORD = "admin1234"
$env:JWT_SECRET = "32자 이상인-개발용-JWT-비밀키를-입력한다"

cd backend
.\gradlew.bat bootRun
```

프런트는 별도 터미널에서 실행한다.

```powershell
cd frontend
npm.cmd run dev
```

## 검증한 범위

- 프런트엔드 프로덕션 빌드는 통과했다.
- `AuthServiceTest`에 로그인 성공, 잘못된 비밀번호, Refresh 재발급, Access 토큰으로 Refresh 시도 거부를 추가했다.
- 테스트를 경로가 단순한 임시 복사본에서 실행했을 때 인증 서비스 테스트 4개는 통과했다.

다만 전체 백엔드 테스트는 로컬 MySQL/Docker 연결 상태에 영향을 받는다. 애플리케이션 컨텍스트 테스트는 DB 연결이 가능한 환경에서 다시 확인해야 한다.

## 아직 해결되지 않은 부분

현재 `admin` / `admin1234`로 로그인했을 때 “아이디 또는 비밀번호가 올바르지 않습니다.”라는 응답이 남아 있다. 이 메시지는 프런트 화면만의 문제라기보다, 실행 중인 백엔드가 새 환경 변수로 시딩된 인스턴스인지와 DB에 기존 `admin` 계정이 남아 있는지를 함께 확인해야 하는 상황이다.

특히 시더는 같은 사용자명이 이미 존재하면 새 비밀번호로 덮어쓰지 않는다. 즉, 이전에 생성된 `admin` 계정이 있다면 환경 변수를 바꿔도 기존 비밀번호가 그대로 남는다. 다음 점검에서는 실행 중인 프로세스의 포트, 실제 연결 DB, `admin` 레코드 생성 시점과 비밀번호 해시를 순서대로 확인할 예정이다.

인증의 코드 구조는 갖춰졌지만, 로컬 실행 환경과 데이터 상태까지 맞아야 비로소 로그인 완료라고 말할 수 있다. 다음 글에서는 이 연결 문제를 해결한 결과를 이어서 정리할 계획이다.
