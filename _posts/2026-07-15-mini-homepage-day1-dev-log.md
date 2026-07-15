---
layout: post
title: "오늘 한 개발 작업 정리 (2026-07-15)"
date: 2026-07-15 20:00:00 +0900
categories: [개발작업, 일일로그]
tags: [Spring Boot, Vue, mini-homepage, Docker]
---

## 오늘 한 일

### mini-homepage 사이드 프로젝트 1단계 착수
- 사이월드 감성 개인 포트폴리오 사이트(`mini-homepage`) 저장소가 빈 상태여서, 미리 정리해둔 1단계(기반 구축 + 관리자 인증) 스코프를 처음부터 구현
  - Spring Initializr로 `backend/` 스캐폴딩 생성 (Spring Boot, Java 21, Gradle Kotlin DSL) — Spring Boot 3.x가 이미 Initializr에서 지원 종료돼 최신 안정 버전으로 올라감
  - Flyway 마이그레이션으로 `admin` 테이블 생성
  - JWT Access/Refresh 토큰 발급·검증 로직과 `SecurityConfig` 구현 (회원가입 API 없이, 환경변수 기반으로 관리자 계정 1개만 자동 시딩)
  - 로그인 / 토큰 재발급 API 구현
  - Vite로 `frontend/` 스캐폴딩 생성 (Vue 3 + TypeScript + Pinia), 로그인 페이지와 라우터 인증 가드 구현
  - 로컬 개발용 `docker-compose.yml`(MySQL) 추가

### 로컬 환경 이슈 두 가지 정리
- Docker Desktop이 꺼져 있어서 실행한 뒤, MySQL 컨테이너용으로 잡아둔 포트를 이미 다른 사이드 프로젝트 컨테이너가 쓰고 있어서 정작 우리 컨테이너 포트는 게시되지 않고 있던 걸 발견 → 다른 포트로 변경
- 로그인 API가 실패 케이스에서 401 대신 403 빈 응답을 내려주는 문제 발견/수정 — 자세한 원인과 해결은 [바로 이전 포스트]({{ site.baseurl }}{% post_url 2026-07-15-spring-security-error-403-override-fix %}) 참고

### 검증 및 마무리
- 실제로 백엔드를 띄운 상태에서 로그인 성공/실패, 토큰 재발급, JWT 인증 필터 통과 여부까지 curl로 전체 시나리오 검증
- 회사 프로젝트 커밋과 계정이 섞이지 않도록, 이 저장소만 로컬 git 계정을 별도 GitHub 계정으로 설정하고 커밋

## 이슈 / 특이사항

- 로컬에 여러 사이드 프로젝트의 도커 컨테이너가 함께 떠 있다 보니 포트 충돌이 에러 없이 조용히 발생할 수 있다는 걸 다시 확인했다. `docker compose up -d` 이후엔 `docker port <컨테이너>`로 실제 게시된 포트를 한 번 확인하는 습관을 들여야겠다.
