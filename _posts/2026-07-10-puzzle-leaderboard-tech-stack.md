---
layout: post
title: "실시간 랭킹 기능 하나 때문에 건드려야 했던 것들 — puzzle-leaderboard 기술 스택 정리"
date: 2026-07-10 21:00:00 +0900
categories: [사이드프로젝트]
tags: [Spring Boot, Redis, MySQL, Redisson, Terraform, GitHub Actions, React]
---

퍼즐 게임에 랭킹 기능을 붙이려고 하면 처음엔 다들 "점수 저장하고 정렬해서 보여주면 되지" 정도로 생각한다. 근데 막상 만들어보면 얘기가 좀 다르다. 동시에 수백 명이 점수를 갱신하는 상황에서 데이터가 안 꼬여야 하고, 내 순위가 몇 등인지 바로 알 수 있어야 하고, 트래픽이 몰리는 순간에도 응답이 느려지면 안 된다. `puzzle-leaderboard`는 이 문제를 풀기 위해 백엔드, 캐시, 인프라, CI/CD까지 여러 기술을 엮어서 만든 프로젝트다. 각각을 왜 골랐는지, 실제로 어떻게 붙였는지 정리해 본다.

## 1. Java 21 + Spring Boot 3.4.3

백엔드는 Java 21, Spring Boot 3.4.3으로 짰다. Java 17이 아니라 21을 쓴 이유는 가상 스레드(Virtual Thread) 때문인데, `spring.threads.virtual.enabled=true` 옵션 하나만 켜면 톰캣이 요청마다 가상 스레드를 붙여서 처리한다. 랭킹 조회 API처럼 요청은 많은데 각 요청 자체는 짧게 끝나는 케이스에서는 플랫폼 스레드 풀 크기에 발목 잡히는 일이 확 줄어든다. 대신 Lombok, QueryDSL 같은 라이브러리 조합이 최신 JDK와 궁합이 안 맞는 경우가 종종 있어서, 그 부분은 버전 잡을 때 좀 신경 썼다.

Spring Boot는 굳이 설명할 필요 없이 익숙해서 선택한 것도 있지만, `@Transactional`로 트랜잭션 경계 잡는 거랑 Actuator로 헬스체크 붙이는 게 익숙한 패턴이라 초기 세팅 시간을 많이 아꼈다.

## 2. MySQL (Aiven)

사용자 정보, 게임 기록처럼 없어지면 안 되는 데이터는 MySQL에 넣었다. `users`, `game_result` 두 테이블이 핵심이고, `game_result`에는 `user_id`, `score`, `played_at` 정도만 있는 단순한 구조다. 랭킹 자체는 Redis가 담당하니까 MySQL 쪽 스키마는 최대한 가볍게 가져갔다.

서버는 직접 안 띄우고 Aiven 매니지드 MySQL을 붙였다. 무료 플랜 기준으로 스토리지가 1GB 정도로 넉넉하진 않지만, 개인 프로젝트에서 백업 스케줄 짜고 장애 대응하는 데 시간 쓰는 것보다는 이쪽이 훨씬 남는 장사였다.

## 3. Redis (Upstash) — Sorted Set

랭킹의 핵심은 여기다. 처음엔 "MySQL에서 `ORDER BY score DESC LIMIT 100` 하면 되잖아" 했는데, 문제는 "내 순위가 몇 등인지"를 구하려면 전체를 정렬한 다음 내 위치를 찾아야 한다는 거였다. 사용자가 늘어날수록 이 쿼리가 점점 무거워질 게 뻔했다.

그래서 Redis Sorted Set으로 갈아탔다. `ZADD leaderboard:global {score} {userId}`로 점수를 넣고, 내 순위는 `ZREVRANK leaderboard:global {userId}`, 상위 100명은 `ZREVRANGE leaderboard:global 0 99 WITHSCORES` 한 번으로 끝난다. 둘 다 O(log N) 근처라 사용자가 늘어도 체감 속도가 거의 그대로다. 동점자 처리는 Redis가 score 같으면 member 값(여기선 userId) 사전순으로 정렬해 주는 걸 그대로 썼는데, 나중에 "먼저 그 점수를 찍은 사람이 위로 가야 한다"는 요구사항이 생기면 score에 타임스탬프를 미세하게 섞는 트릭을 써야 할 것 같다. 시즌 초기화는 키를 지우는 대신 `leaderboard:season:2024-06` 식으로 시즌별 키를 따로 두는 쪽으로 정리했다. Redis는 직접 안 띄우고 Upstash 서버리스로 붙였다.

## 4. Redisson 분산락

서버 인스턴스가 여러 개 뜨는 순간부터 동시성 문제가 진짜가 된다. 같은 유저의 점수를 두 요청이 동시에 갱신하려고 하면 레이스 컨디션이 난다. 로컬 `synchronized`로는 인스턴스 간 문제를 못 막길래 Redisson의 `RLock`을 붙였다.

`lock:score:{userId}` 형태로 키를 잡고, waitTime 3초, leaseTime 5초로 설정해서 락을 오래 붙잡고 있는 요청이 있어도 시스템 전체가 멈추지 않게 했다. `try-finally`로 unlock을 강제하는 걸 놓쳐서 초반에 락이 안 풀리는 버그를 한 번 겪었는데, 그 이후로는 락 획득/해제 로직을 별도 유틸로 빼서 실수할 여지를 줄였다.

## 5. Flyway

스키마 변경은 `V1__init.sql`, `V2__add_index.sql` 식으로 파일을 쌓아가면서 관리한다. 로컬에서 스키마 손으로 바꾸고 나중에 "이거 운영 DB에 반영했나?" 헷갈리는 걸 예전 프로젝트에서 겪은 적이 있어서, 이번엔 처음부터 넣었다. CI 파이프라인에서 앱이 뜨기 전에 `flyway migrate`가 먼저 돌도록 순서를 잡아뒀다.

## 6. JMeter 부하 테스트

분산락까지 넣고 나니 "진짜 버티는지"를 확인하고 싶었다. JMeter로 Thread Group 500명, ramp-up 30초로 잡고 점수 갱신 API를 두들겨봤다. 처음 돌렸을 때는 HikariCP 커넥션 풀 기본값(10)이 너무 작아서 커넥션 타임아웃이 우수수 났고, 풀 사이즈를 늘리고 나서야 p95 응답시간이 어느 정도 안정적인 수준으로 내려왔다. 부하 테스트를 안 해봤으면 이 병목을 배포하고 나서야 발견했을 거다.

## 7. 인프라: Render / Aiven / Upstash

인프라는 최대한 관리형 서비스로 몰아넣었다. 배포는 Render, DB는 Aiven, 캐시는 Upstash다. Render 무료 플랜은 15분 정도 요청이 없으면 인스턴스가 슬립 상태로 들어가서 콜드 스타트가 몇 초 걸리는 게 단점인데, 개인 프로젝트 단계에서는 감수할 만한 트레이드오프라고 판단했다.

## 8. Terraform (AWS ECS, 참고용)

지금 운영 방식과는 별개로, AWS ECS Fargate + ALB 구성을 Terraform 코드로 짜둔 게 저장소에 같이 들어있다. 아직 실제로 적용한 상태는 아니고, 트래픽이 늘었을 때 옮겨갈 수 있는 경로를 미리 코드로 남겨둔 것에 가깝다.

## 9. GitHub Actions

`.github/workflows/ci.yml`에 gradle build → test → Render 배포 훅 호출까지 이어지는 파이프라인을 넣었다. 브랜치에 푸시하면 자동으로 테스트가 돌고, main에 머지되면 배포 훅이 호출되는 식이라 수동으로 배포하는 일이 거의 없다.

## 10. React 18 + TypeScript + Vite → Vercel

프론트는 React 18, TypeScript, Vite 조합이다. 랭킹 데이터는 몇 초 간격으로 값이 바뀔 수 있어서 TanStack Query로 polling을 걸어뒀다. Vite는 로컬 개발 서버 속도가 체감상 확실히 빠르고, PR을 올리면 Vercel이 자동으로 프리뷰 배포를 만들어줘서 리뷰할 때 코드만 보지 않고 실제 화면을 같이 보고 판단할 수 있는 게 꽤 유용했다.

## 마무리

정리하고 보니 "실시간 랭킹"이라는 요구사항 하나 때문에 건드려야 했던 영역이 생각보다 많았다. 캐시 자료구조 선택, 분산 환경에서의 락 처리, 부하 테스트로 병목 찾기, 관리형 서비스로 운영 부담 줄이기까지. 다음에 손댈 부분은 시즌별 랭킹 히스토리를 보여주는 기능이랑, 관리자용 대시보드 정도로 생각하고 있다.

**저장소**: [github.com/amiesoft-hyk/puzzle-leaderboard](https://github.com/amiesoft-hyk/puzzle-leaderboard)
