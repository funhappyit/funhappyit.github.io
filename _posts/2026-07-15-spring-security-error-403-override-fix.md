---
layout: post
title: "Spring Security 컨트롤러 예외가 401 대신 403 빈 응답으로 뒤바뀌던 문제"
date: 2026-07-15 19:00:00 +0900
categories: [오류해결, Spring Security]
tags: [Spring Boot, Spring Security, JWT]
---

## 문제 상황

개인 사이드 프로젝트에 관리자 전용 JWT 로그인 API(`POST /api/v1/auth/login`)를 구현하고 curl로 시나리오별 테스트를 하는데, 이상한 현상이 나타났다.

- 정상 로그인(아이디/비밀번호 일치): `200 OK` + 토큰 정상 발급
- 잘못된 비밀번호: 컨트롤러에서 `ResponseStatusException(HttpStatus.UNAUTHORIZED, ...)`를 던지도록 만들어뒀는데, 실제 응답은 401이 아니라 **403 Forbidden + 빈 바디**
- 로그인 엔드포인트를 지원하지 않는 메서드(GET)로 호출해도 405 대신 403
- 인증 없이 보호된 엔드포인트를 호출해도 403 — 이건 기대한 그대로였지만, 결국 모든 실패 케이스가 403 하나로 뭉개지고 있다는 게 문제였다

## 원인

Spring Boot는 컨트롤러(또는 필터 체인) 처리 중 예외가 발생하면, 예외를 곧바로 클라이언트에 응답하는 게 아니라 서블릿 컨테이너 차원에서 `GET /error` 경로로 **내부 forward**를 한다. 이 forward된 요청을 `BasicErrorController`가 받아서, 원래 발생한 예외의 상태 코드(이 경우 401)를 읽어 응답을 만든다.

문제는 이 `/error` forward도 "새 요청"으로 취급되어 Spring Security의 필터 체인을 다시 통과한다는 점이다. `SecurityConfig`에서는 인증 API 경로만 `permitAll()`로 열어뒀고 `/error`는 따로 허용하지 않았기 때문에 `anyRequest().authenticated()`에 걸려버렸다. 인증되지 않은 상태로 `/error`에 접근하니, 원래 만들려던 401 응답이 완성되기도 전에 Spring Security의 `Http403ForbiddenEntryPoint`가 먼저 끼어들어 403(빈 바디)으로 막아버린 것이다.

```java
// 문제가 있던 설정
http
    .authorizeHttpRequests(auth -> auth
        .requestMatchers("/api/v1/auth/**").permitAll()
        .anyRequest().authenticated());
```

`logging.level.org.springframework.security=DEBUG`로 필터 체인 로그를 켜보니 흐름이 명확히 보였다.

```
Securing POST /api/v1/auth/login
Set SecurityContextHolder to anonymous SecurityContext
Secured POST /api/v1/auth/login
Securing GET /error                     # 컨트롤러 예외 이후 내부 forward
Set SecurityContextHolder to anonymous SecurityContext
Http403ForbiddenEntryPoint : Pre-authenticated entry point called. Rejecting access
```

`POST /api/v1/auth/login` 자체는 permitAll을 정상적으로 통과했지만(`Secured`), 바로 뒤이어 들어온 `GET /error`가 인증 요구에 걸려 거부됐고, 그 403이 클라이언트에게 최종 응답으로 나간 것이었다.

## 해결 방법

`/error` 경로를 명시적으로 `permitAll()`에 추가했다.

```java
http
    .authorizeHttpRequests(auth -> auth
        .requestMatchers("/api/v1/auth/**", "/error").permitAll()
        .anyRequest().authenticated());
```

## 결과

같은 시나리오를 다시 돌려보니 각 케이스가 의도한 상태 코드를 그대로 돌려줬다.

| 시나리오 | 수정 전 | 수정 후 |
|---|---|---|
| 잘못된 비밀번호 | 403 (빈 바디) | 401 |
| 유효하지 않은 refresh token | 403 | 401 |
| access token으로 refresh 시도 | 403 | 401 |
| 지원하지 않는 메서드(GET) | 403 | 405 |
| 인증 없이 보호된 리소스 접근 | 403 | 403 (그대로, 정상 동작) |

### 배운 점

Spring Security의 `authorizeHttpRequests` 규칙은 "내가 만든 API 경로"만 생각하고 채우기 쉽지만, 예외 발생 시 Spring Boot가 내부적으로 forward하는 `/error` 같은 경로도 같은 필터 체인을 통과한다는 걸 놓치면 모든 실패 응답이 403 하나로 뭉개져 버린다. 커스텀 시큐리티 설정을 쓸 때는 `/error`(필요하면 `/actuator/health` 같은 경로도) permitAll 여부를 체크리스트에 넣어두는 게 안전하다.
