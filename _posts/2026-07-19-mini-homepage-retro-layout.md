---
layout: post
title: "참고 이미지로 미니홈피 레트로 레이아웃 다듬기"
date: 2026-07-19 19:00:00 +0900
categories: [사이드프로젝트, mini-homepage]
tags: [Vue, CSS, 레트로 UI, mini-homepage]
---

mini-homepage의 화면을 예전 미니홈피 느낌에 더 가깝게 다듬었다. 이번 작업의 기준은 단순히 파란색을 입히는 것이 아니라, 참고 이미지에 있는 **프레임·패널·탭·우측 위젯의 상대적인 크기와 위치**를 맞추는 것이었다.

## 목표

- 회색 격자 배경 위에 하늘색 미니홈피 프레임을 배치한다.
- 좌측 프로필, 가운데 본문, 우측 세로 탭을 책장처럼 보이게 구성한다.
- 우측 `마이홈` 위젯은 본문 패널과 분리해 프레임 바깥에 놓는다.
- 홈 화면은 `Updated news → Miniroom → What friends say` 흐름을 유지한다.

## 1. 화면 전체를 고정 비율로 다루기

브라우저 크기에 따라 각 요소가 따로 변하면 참고 이미지의 비율이 쉽게 무너진다. 그래서 `#app`에 디자인 기준 크기를 두고, 뷰포트 크기에 따라 전체를 한 번에 확대·축소하도록 구성했다.

```css
#app {
  width: 1060px;
  height: 560px;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%)
    scale(min(calc(98vw / 1060px), calc(94vh / 560px)));
}
```

이 방식은 개별 패널의 폭을 반응형으로 다시 계산하지 않아도, 데스크톱 화면에서 레트로 UI 특유의 고정된 캔버스 느낌을 유지할 수 있다. 작은 화면에서는 기존 미디어 쿼리로 일반 세로 레이아웃으로 전환한다.

## 2. 배경과 외곽 프레임

본문 밖은 두 개의 선형 그라디언트로 회색 격자를 만들었다. 메인 프레임은 하늘색 바탕, 둥근 모서리, 안쪽 흰 점선 테두리로 구성했다.

```css
body {
  background-color: #727777;
  background-image:
    linear-gradient(rgba(255, 255, 255, 0.14) 1px, transparent 1px),
    linear-gradient(90deg, rgba(255, 255, 255, 0.14) 1px, transparent 1px);
  background-size: 16px 16px;
}

.cy-box::before {
  content: '';
  position: absolute;
  inset: 16px;
  border: 1px dashed rgba(255, 255, 255, 0.9);
  border-radius: 10px;
}
```

## 3. 프로필과 본문 비율 맞추기

참고 이미지에서는 좌측 프로필이 너무 좁지 않고, 본문은 미니룸을 넉넉히 보여줄 정도로 넓다. 그 비율을 위해 레이아웃을 `프로필 205px / 스프링 10px / 본문`으로 잡았다.

```css
.cy-layout {
  display: grid;
  grid-template-columns: 205px 10px minmax(0, 1fr);
  gap: 8px;
}
```

가운데의 점 형태 스프링은 `radial-gradient()`를 세로로 반복해 구현했다. 별도 이미지가 없어도 책 제본처럼 보이면서 좌측과 본문을 자연스럽게 분리한다.

## 4. 메뉴 탭과 우측 위젯은 본문 밖으로

처음에는 세로 메뉴와 `마이홈` 위젯을 그리드 컬럼으로 넣었는데, 이 경우 파란 프레임 오른쪽에 큰 빈 공간이 생기고 위젯이 메뉴와 겹치기 쉬웠다.

현재는 메뉴 탭과 위젯을 절대 위치로 배치했다. 메뉴는 본문 오른쪽에 붙이고, 위젯은 프레임 바깥 오른쪽 상단으로 이동시켰다. 선택된 탭은 흰 패널과 짙은 테두리를 적용하고, 모든 모서리를 둥글게 처리했다.

```css
.nav-tabs {
  position: absolute;
  right: -52px;
  width: 54px;
}

.friend-widget {
  position: absolute;
  top: -84px;
  right: -236px;
  width: 165px;
}
```

절대 위치는 반응형 화면에서 주의가 필요하다. 모바일 구간에서는 위젯을 다시 일반 흐름으로 되돌려 콘텐츠가 겹치지 않도록 했다.

## 5. 홈 화면의 미니룸 영역

홈 화면은 세 개의 섹션이 좁은 본문 안에 들어간다. 미니룸이 작은 카드처럼 보이지 않도록 `flex`를 사용해 남는 높이와 본문 폭을 사용하게 했다.

```css
.room {
  flex: 1 1 auto;
  min-height: 0;
}

.room :deep(.room-scene) {
  width: 100%;
  max-width: none;
  flex: 1 1 auto;
  aspect-ratio: 16 / 9;
}
```

이후 미니룸 일러스트 자체를 더 풍부하게 만들거나 실제 사용자 이미지로 교체하더라도, 카드의 크기와 위치는 유지된다.

## 검증과 배포

프론트엔드 빌드로 타입 검사와 번들 생성을 확인했다.

```bash
cd frontend
npm.cmd run build
```

마지막으로 변경 사항은 `Redesign mini homepage layout and profile widgets` 커밋으로 정리했고, `main` 브랜치가 GitHub 원격 저장소와 동기화된 상태를 확인했다.

이번 작업에서 얻은 결론은, 레퍼런스 기반 UI는 색상 하나보다도 **고정된 캔버스 비율과 패널 간의 상대 위치**를 먼저 맞추는 편이 훨씬 빠르게 원하는 분위기에 도달한다는 점이다.
