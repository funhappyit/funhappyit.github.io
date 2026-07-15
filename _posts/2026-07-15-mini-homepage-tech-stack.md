---
layout: post
title: "가입 기능 없는 나만의 미니홈피, 하루 만에 로그인까지 붙인 이야기 — mini-homepage 기술 스택 정리"
date: 2026-07-15 21:00:00 +0900
categories: [사이드프로젝트, mini-homepage]
tags: [Spring Boot, Java, MySQL, Flyway, JWT, Spring Security, Vue, TypeScript, Pinia, Vite, Docker]
---

사이월드 감성의 개인 포트폴리오를 만들기로 했다. 회원가입/로그인이 필요한 다중 사용자 SNS가 아니라, 관리자(나) 한 명만 로그인해서 대문/프로필/다이어리/미니룸을 관리하고, 방문자는 로그인 없이 둘러보고 방명록에 익명으로 글만 남기는 구조다. 오늘은 완전히 빈 저장소에서 시작해서, 1단계인 "기반 구축 + 관리자 인증"까지 끝냈다. 뭘 왜 골랐고 실제로 어떻게 붙였는지 정리해본다.

## 1. Java 21 + Spring Boot 4.1.0

백엔드는 Spring Initializr로 스캐폴딩을 잡았다. 원래 계획서엔 "Spring Boot 3.x"라고 적어놨었는데, 막상 Initializr에 들어가보니 3.x는 이미 지원 범위에서 빠져 있고 최신 안정 버전인 4.1.0이 기본값이었다. 계획 문서 쓸 때랑 실제로 만드는 시점 사이에 생태계가 한 세대 넘어가 있었던 셈이라, README도 실제 버전에 맞춰 다시 정리했다.

패키지는 기능이 아니라 도메인 기준으로 나눴다. `domain/admin`(엔티티·레포지토리·시딩), `security`(JWT), `auth`(로그인 API), `config`(Security 설정) — 아직 도메인이 하나뿐이라 크게 체감은 안 되지만, 2단계부터 다이어리·방명록·미니룸이 붙기 시작하면 이 구조가 빛을 볼 거라 생각한다.

## 2. JWT — Access/Refresh, 근데 회원가입 API는 없다

이 프로젝트에서 가장 특이한 지점은 "회원가입 API 자체가 없다"는 거다. 관리자 계정은 딱 1개, 그것도 코드로 만드는 게 아니라 앱이 처음 뜰 때 환경변수(`ADMIN_SEED_USERNAME`, `ADMIN_SEED_PASSWORD`)를 읽어서 없으면 한 번만 생성한다.

```java
// AdminSeeder — 부팅 시 1회 체크 후 없으면 생성
@Override
public void run(String... args) {
    if (seedPassword == null || seedPassword.isBlank()) {
        log.warn("ADMIN_SEED_PASSWORD가 설정되지 않아 관리자 계정 시딩을 건너뜁니다.");
        return;
    }
    if (adminRepository.existsByUsername(seedUsername)) {
        return;
    }
    adminRepository.save(new Admin(seedUsername, passwordEncoder.encode(seedPassword)));
}
```

토큰은 Access/Refresh 이중 구조로, JWT claim에 `tokenType`(`ACCESS`/`REFRESH`)을 넣어서 refresh 토큰을 access 토큰 대신 들이미는 걸 막았다. `/api/v1/auth/refresh`에서 넘어온 토큰이 실제로 refresh 타입인지 한 번 더 확인하는 식이다.

Spring Security는 세션을 아예 안 쓰는 stateless 구조로 잡고, 커스텀 `JwtAuthenticationFilter`를 `UsernamePasswordAuthenticationFilter` 앞에 끼워 넣었다. `/api/v1/auth/**`만 permitAll로 열어두면 끝날 줄 알았는데, 실제로는 그렇지 않았다 — 컨트롤러 예외가 401 대신 403 빈 응답으로 뒤바뀌는 문제를 하나 더 만났고, 원인과 해결은 [바로 이전 포스트]({{ site.baseurl }}{% post_url 2026-07-15-spring-security-error-403-override-fix %})에 따로 정리했다.

## 3. Flyway

스키마는 `V1__init.sql` 하나로 시작했다. 아직 1단계라 `admin` 테이블 하나뿐이지만, 앞으로 다이어리·방명록·미니룸 테이블이 붙을 때마다 버전 파일을 쌓아가는 방식으로 관리할 생각이다. `ddl-auto`는 처음부터 `none`으로 박아두고 스키마 변경은 전부 마이그레이션 파일로만 하기로 했다 — 나중에 로컬이랑 배포 환경 스키마가 슬쩍 어긋나는 걸 막기 위해서다.

## 4. Vue 3 + TypeScript + Vite + Pinia

프론트도 Vite로 스캐폴딩했다. 아직 대문/프로필 화면은 없고, 로그인 페이지와 `/admin` 라우트 가드만 붙였다.

```ts
// router/index.ts — 인증 안 된 상태로 /admin 접근 시 로그인으로 리다이렉트
router.beforeEach((to) => {
  const authStore = useAuthStore()
  if (to.meta.requiresAuth && !authStore.isAuthenticated) {
    return { name: 'login' }
  }
  return true
})
```

토큰은 Pinia 스토어에서 관리하면서 localStorage에도 같이 저장해 새로고침해도 로그인이 풀리지 않게 했고, axios 인터셉터가 요청마다 `Authorization` 헤더를 자동으로 붙여준다.

## 5. Docker Compose — 근데 포트를 이미 다른 프로젝트가 쓰고 있었다

로컬 개발용 MySQL은 docker-compose로 띄웠다. 계획했던 포트(3307)로 `docker compose up -d`를 돌렸는데, 에러 하나 없이 "Started"라고 뜨길래 그대로 믿고 넘어갔었다. 그런데 실제로 백엔드를 붙여보니 계속 "Access denied" 에러가 났다.

알고 보니 그 포트는 이미 다른 사이드 프로젝트의 MySQL 컨테이너가 쓰고 있었고, docker compose는 포트를 게시하지 못한 채로 컨테이너만 "Started" 상태로 조용히 떠 있었다. `docker port <컨테이너>`로 확인해보고 나서야 우리 컨테이너의 3306 포트가 호스트 어디에도 연결돼 있지 않다는 걸 알았다. 로컬에 사이드 프로젝트 컨테이너가 여러 개 떠 있다 보니 벌어진 일 — 포트를 바꾸고 나서야 정상적으로 붙었다. `docker compose up -d`가 성공했다고 해서 포트까지 정상 게시됐다는 보장은 없다는 걸 이번에 확실히 배웠다.

## 오늘까지 한 일 / 다음 단계

1단계(기반 구축 + 관리자 인증)는 로그인 성공/실패, 토큰 재발급, 인증 필터 통과 여부까지 curl로 시나리오별 검증까지 마쳤다. 다음은 2단계 — 대문/프로필 화면 붙이는 것부터 시작할 예정이다.
