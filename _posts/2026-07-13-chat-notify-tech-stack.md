---
layout: post
title: "채팅 하나 만들려다 인증 경로가 세 갈래로 갈라진 이야기 — chat-notify 기술 스택 정리"
date: 2026-07-13 20:00:00 +0900
categories: [사이드프로젝트, chat-notify]
tags: [Spring Boot, WebSocket, STOMP, Redis, SSE, JWT, React, Docker, Render, Vercel]
---

실시간 채팅을 만든다고 하면 보통 "WebSocket 붙이면 되지" 정도로 생각하기 쉽다. 근데 인증 하나만 놓고 봐도 얘기가 달라진다. 일반 REST 요청은 헤더에 토큰 넣으면 끝인데, WebSocket 핸드셰이크와 SSE 구독은 헤더를 마음대로 못 넣는 경우가 많아서 각각 다른 방식으로 토큰을 받아야 한다. `chat-notify`는 이 인증 문제부터 시작해서, 다중 서버 환경에서 WebSocket 메시지를 어떻게 모든 클라이언트에 전달할지, 트랜잭션이 커밋되기도 전에 알림이 나가버리는 문제까지 하나씩 부딪히면서 만든 프로젝트다. 겪은 순서대로 정리해본다.

## 1. Java 21 + Spring Boot 3.x

백엔드는 Java 21, Spring Boot 3.x로 짰다. WebSocket/STOMP, SSE, JWT 인증을 한 애플리케이션에서 같이 다뤄야 해서 각 진입점마다 인증 처리를 다르게 넣어야 했는데, Spring Security의 필터 체인과 STOMP 인터셉터를 분리해서 구성할 수 있다는 점이 이번 프로젝트의 핵심 요구사항과 잘 맞았다.

## 2. JWT 인증 — 경로 세 개, 방식 세 개

Access/Refresh 이중 토큰 구조 자체는 흔한 패턴이라 어렵지 않았다. 문제는 토큰을 "어디서 어떻게 받을지"가 진입점마다 다르다는 거였다.

- **일반 HTTP API**: `Authorization: Bearer {token}` 헤더를 그대로 읽으면 된다. `JwtAuthenticationFilter`가 이 케이스를 담당한다.
- **STOMP(WebSocket)**: 핸드셰이크 이후 STOMP CONNECT 프레임의 커스텀 헤더로 토큰을 실어 보내고, `StompAuthChannelInterceptor`에서 CONNECT 프레임을 가로채 검증한다. 일반 HTTP 필터 체인을 안 타기 때문에 별도 인터셉터가 필요했다.
- **SSE 구독**: `EventSource`는 브라우저 표준 API라 커스텀 헤더를 못 넣는다. 그래서 SSE 구독 엔드포인트(`/api/v1/notify/subscribe`)에 한해서만 쿼리 파라미터로 토큰을 받도록 예외를 뒀다. 처음엔 이 쿼리 파라미터 폴백을 모든 API 경로에 열어놨는데, 코드 리뷰 과정에서 "SSE 구독 경로 말고는 열어둘 이유가 없다"는 지적을 받고 특정 경로 하나로 좁혔다. 토큰이 URL에 남는 방식은 로그나 리퍼러로 새어나갈 여지가 있어서, 꼭 필요한 곳에만 열어두는 게 맞다.

## 3. Spring WebSocket + STOMP — 실시간 채팅

채팅 메시지 발행은 `/pub/message`, 구독은 `/sub/room/{roomId}` 패턴으로 잡았다. STOMP 자체는 익숙한 pub/sub 패턴이라 붙이는 데는 오래 안 걸렸는데, 진짜 문제는 다음 항목이었다.

## 4. Redis Pub/Sub — 서버가 여러 대일 때 브로드캐스팅

WebSocket은 기본적으로 "그 요청을 받은 서버"에 연결이 묶인다. 서버 A에 연결된 클라이언트에게 서버 B가 메시지를 직접 밀어줄 방법이 없다. 인스턴스가 하나뿐이면 문제가 안 되지만, 스케일아웃을 하는 순간 다른 서버에 붙어 있는 사람은 메시지를 못 받는다.

```
[서버 A] ── WebSocket ── [클라이언트 1]
[서버 B] ── WebSocket ── [클라이언트 2]

서버 A로 메시지 전송 → 서버 B의 클라이언트 2는 수신 불가
```

그래서 메시지를 저장할 때 STOMP로 바로 브로드캐스팅하지 않고, 일단 Redis 채널에 발행(publish)하도록 바꿨다. 모든 서버 인스턴스가 같은 채널을 구독하고 있다가, 메시지가 들어오면 각자 자기한테 붙어있는 클라이언트에게 STOMP로 전달한다. 이렇게 하면 어느 서버가 메시지를 받았든 상관없이 모든 클라이언트에 도달한다.

이걸 로컬에서 검증하려고 같은 애플리케이션을 포트만 다르게 해서 두 개 띄웠다(8080, 8081). 클라이언트 1은 8080에, 클라이언트 2는 8081에 WebSocket으로 붙인 다음, 8080 쪽으로 메시지를 보내고 8081에 붙은 클라이언트가 실제로 메시지를 받는지 Node.js 스크립트로 확인했다. 단일 서버에서만 테스트했으면 이 브로드캐스팅 로직이 우연히 동작하는 건지 제대로 동작하는 건지 구분이 안 됐을 거다.

## 5. SSE — 안읽음 알림

채팅방에 새 메시지가 오면 그 방에 없는 사용자에게도 "안읽은 메시지가 있다"는 걸 알려줘야 한다. 이건 양방향 통신이 필요 없고 서버 → 클라이언트 단방향 푸시만 있으면 되는 케이스라 WebSocket 대신 SSE(Server-Sent Events)를 썼다. `SseEmitter`로 연결을 열어두고, 메시지 발행 시점에 알림 채널로 안읽음 카운트를 흘려보내는 구조다. 알림도 결국 Redis Pub/Sub을 한 번 거쳐서 나가기 때문에, 채팅 메시지 브로드캐스팅과 같은 인프라를 재사용할 수 있었다.

## 6. `@Transactional` 자기 호출 함정, 그리고 커밋 순서 버그

두 가지를 이 프로젝트에서 제대로 겪었다.

첫 번째는 흔한 함정인데, 같은 클래스 안에서 `@Transactional`이 붙은 메서드를 `this.method()` 형태로 호출하면 프록시를 안 거치기 때문에 트랜잭션이 안 걸린다. Spring AOP가 프록시 기반이라 외부에서 호출될 때만 트랜잭션이 적용된다는 걸 알고는 있었지만, 메시지 조회 로직을 리팩터링하다가 실제로 한 번 빠뜨렸다. 조회 메서드에 트랜잭션 어노테이션을 따로 명시하는 걸로 해결했다.

두 번째는 좀 더 미묘했다. 메시지를 저장하는 로직이 대략 이런 흐름이었다.

```java
@Transactional
public MessageResponse saveMessage(Long roomId, Long senderId, String content) {
    Message message = messageRepository.save(new Message(roomId, senderId, content));
    MessageResponse response = MessageResponse.from(message);

    publishMessage(response);           // Redis에 발행 → 다른 서버로 브로드캐스팅
    notifyOtherMembers(roomId, senderId); // 안읽음 알림 발송

    return response;
}
```

문제는 `publishMessage`와 `notifyOtherMembers`가 트랜잭션이 실제로 커밋되기 *전에* 실행된다는 점이었다. `@Transactional` 메서드 안에서 DB 저장과 Redis 발행을 순서대로 적었다고 해서 "DB에 반영된 다음 발행된다"는 보장이 안 된다. 트랜잭션은 메서드가 끝나야 커밋되는데, 그 안에서 이미 Redis로 메시지를 쏴버린 거다. 만약 이후 로직에서 예외가 나서 롤백되면? 클라이언트는 이미 화면에 메시지를 받았는데 DB에는 그 메시지가 없는, "보이는데 존재하지 않는 메시지"가 생길 수 있는 상황이었다.

고치는 방법은 `TransactionSynchronizationManager`로 커밋 이후 실행되는 콜백을 등록하는 거였다.

```java
@Transactional
public MessageResponse saveMessage(Long roomId, Long senderId, String content) {
    Message message = messageRepository.save(new Message(roomId, senderId, content));
    MessageResponse response = MessageResponse.from(message);

    TransactionSynchronizationManager.registerSynchronization(
        new TransactionSynchronization() {
            @Override
            public void afterCommit() {
                publishMessage(response);
                notifyOtherMembers(roomId, senderId);
            }
        }
    );

    return response;
}
```

이렇게 하면 트랜잭션이 실제로 커밋된 다음에만 Redis 발행이 나간다. 롤백되면 애초에 콜백 자체가 실행되지 않으니 "DB엔 없는데 화면엔 보이는 메시지" 문제가 원천적으로 막힌다. PR 리뷰를 받으면서 지적받은 부분이었는데, 혼자 코드를 봤으면 한동안 놓쳤을 것 같다.

## 7. React 18 + TypeScript + Vite

프론트는 React 18, TypeScript, Vite 조합으로 붙였다. STOMP 클라이언트는 `@stomp/stompjs` + `sockjs-client`를 썼고, SSE는 브라우저 기본 `EventSource`를 그대로 썼다. 하나 걸렸던 건 SSE 알림 콜백 안에서 방 이름을 표시해야 하는데, `useEffect`가 마운트 시점의 `rooms` 상태를 클로저로 캡처해버려서 알림 토스트에 항상 기본값("채팅방")만 뜨는 버그가 있었다. `setRooms`의 함수형 업데이터 안에서 방 이름을 조회하도록 바꿔서, 최신 상태를 기준으로 이름을 찾도록 고쳤다.

## 8. Docker 멀티스테이지 빌드 — gradlew가 갑자기 실행이 안 될 때

배포용 이미지는 빌드 스테이지(JDK)와 실행 스테이지(JRE)를 나눠서 이미지 크기를 줄였다.

```dockerfile
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app
COPY gradlew .
COPY gradle gradle
COPY build.gradle.kts settings.gradle.kts ./
RUN chmod +x gradlew
RUN ./gradlew dependencies --no-daemon
COPY src src
RUN ./gradlew bootJar --no-daemon

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-Xmx256m", "-jar", "app.jar"]
```

로컬에서 처음 빌드를 돌렸을 때 `./gradlew: not found`라는, 파일이 분명히 있는데 없다는 에러가 났다. 원인은 Windows의 `core.autocrlf` 설정 때문에 워킹 트리의 `gradlew` 파일이 CRLF 줄바꿈으로 바뀌어 있었던 거였다. 셔뱅 라인이 `#!/bin/sh\r`가 되면 리눅스 컨테이너 안에서는 `\r`까지 파일명의 일부로 인식돼서 인터프리터를 못 찾는다. `.gitattributes`에 `gradlew text eol=lf`를 추가해서 체크아웃할 때 항상 LF로 강제하는 걸로 해결했다.

## 9. Render / Vercel / Aiven / Upstash — 무료 조합 배포

배포는 백엔드를 Render, 프론트엔드를 Vercel에 올렸고, DB는 Aiven MySQL, 캐시는 Upstash Redis를 붙였다. 둘 다 다른 오리진이라 프론트엔드에서 API 주소를 환경변수(`VITE_API_BASE_URL`)로 주입받도록 바꿔야 했고, CORS도 `CorsConfigurationSource` 빈을 따로 등록해서 `/api/**`, `/ws/**` 양쪽에 적용했다. Spring Security를 쓰는 프로젝트에서는 `WebMvcConfigurer`의 CORS 설정만으로는 Security 필터가 먼저 막아버려서, `.cors(Customizer)`로 필터 체인에도 명시적으로 연결해줘야 한다는 걸 다시 확인했다.

Render 무료 플랜은 일정 시간 요청이 없으면 인스턴스가 슬립 상태로 들어가서, 배포 후 첫 요청이 콜드 스타트로 30~50초 정도 걸린다. 실제로 배포 직후 회원가입 요청을 날렸더니 CORS preflight(OPTIONS) 요청이 한동안 pending 상태로 멈춰 있었는데, 콜드 스타트가 끝나자마자 정상적으로 200이 돌아왔다. 무료 인프라를 쓰는 이상 감수할 트레이드오프라고 보고 있다.

## 마무리

인증 하나만 세 갈래로 나뉘는 것부터 시작해서, 다중 서버 브로드캐스팅, 트랜잭션 커밋 순서, Windows-리눅스 줄바꿈 문제까지 — 각각은 흔한 이슈들이지만 한 프로젝트 안에서 다 겹치니까 생각보다 손이 많이 갔다. 다음엔 GitHub Actions로 빌드/테스트 파이프라인을 붙이고, 로그인부터 채팅·알림까지 이어지는 E2E 테스트를 자동화하는 걸 남겨두고 있다.

**저장소**: [github.com/funhappyit/chat-notify](https://github.com/funhappyit/chat-notify)
