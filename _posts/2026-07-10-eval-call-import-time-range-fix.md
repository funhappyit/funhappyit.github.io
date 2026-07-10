---
layout: post
title: "일정 기반 데이터 가져오기 화면의 시간대 입력 범위 제한 누락 문제 해결"
date: 2026-07-10 18:00:00 +0900
categories: [오류해결]
tags: [Vue, DevExtreme, QueryDSL]
---

## 문제 상황

일정(회차) 관리 기반으로 외부 데이터를 가져오는 관리 화면에는 "데이터 가져오기" 모달이 있고, 그 안에 "추출 대상 시간대"라는 시간 범위 입력창(시작시간 ~ 종료시간)이 있다. 이 값은 모달을 열 때 상위 일정 관리 화면에서 설정한 해당 회차의 추출 시작~종료 시간으로 초기화된다.

문제는 이 입력창이 아무런 제약 없이 편집 가능한 date-box로 열려 있었다는 점이다. 사용자가 회차에서 정한 범위와 무관하게 임의의 시간대를 입력해도 그대로 검색이 수행되어, "회차의 추출 대상 시간대"라는 원래 의미가 무색해지는 구조였다.

같은 회차 데이터인 최소/최대 조건 값(예: 대상 길이의 최소·최대값)은 애초에 화면에 입력창 자체가 없어 수정이 불가능한데, 유독 시간대 입력만 자유롭게 바뀔 수 있어 일관성도 없었다.

## 원인

프론트엔드의 검색 로직을 추적해보면:

```js
// 검색 파라미터 조립부 (요약)
searchParams.fromTime = moment(searchForm.fromTime).format('HHmmSS');
searchParams.toTime = moment(searchForm.toTime).format('HHmmSS');
```

사용자가 입력창 값을 바꾸면 그 값이 그대로 목록 조회 API 파라미터로 전송된다. 백엔드(QueryDSL 기반 레포지토리)도 이 값을 실제 WHERE 조건에 사용한다.

```java
// 검색 조건 빌더 (요약)
record.startTime.between(searchDto.getFromTime(), searchDto.getToTime()),
record.endTime.between(searchDto.getFromTime(), searchDto.getToTime()),
```

즉 백엔드는 정상적으로 필터링을 수행하고 있었지만, 프론트엔드에서 입력값 자체를 회차 범위 내로 제한하지 않아 "회차에서 정한 시간대"라는 전제가 UI 단에서 지켜지지 않는 상태였다.

## 해결 방법

date-box 컴포넌트의 `min`/`max` prop을 이용해 입력 가능 범위를 회차의 추출 시작~종료 시간으로 제한했다.

1. 회차 데이터 조회 시 시작/종료 시간을 Date 객체로 변환해 별도 상태(`timeBounds.min`/`timeBounds.max`)에 저장한다.

```js
const timeBounds = reactive({
  min: null,
  max: null,
});
```

```js
// 회차 데이터 조회 후 처리 (요약)
const startDate = new Date(today);
startDate.setHours(startHours);
startDate.setMinutes(startMinutes);

const endDate = new Date(today);
endDate.setHours(endHours);
endDate.setMinutes(endMinutes);

timeBounds.min = startDate;
timeBounds.max = endDate;
```

기존 코드는 `today`라는 Date 객체 하나를 `setHours`/`setMinutes`로 재사용(mutate)하면서 시작/종료 시각을 순차적으로 계산했다. 문자열로 즉시 스냅샷을 뜨는 용도로만 쓸 때는 문제가 없었지만, `min`/`max`처럼 Date 객체 참조 자체를 보관해야 하는 상황에서는 두 값이 같은 객체를 가리키게 되는 문제가 생긴다. 그래서 시작/종료 시각을 별도의 Date 객체로 분리했다.

2. 템플릿의 시간대 입력창 2개(시작/종료)에 `min`/`max`를 바인딩한다.

```vue
<date-box
  v-model="searchForm.fromTime"
  type="time"
  display-format="HH:mm"
  :min="timeBounds.min"
  :max="timeBounds.max"
/>
<date-box
  v-model="searchForm.toTime"
  type="time"
  display-format="HH:mm"
  :min="timeBounds.min"
  :max="timeBounds.max"
/>
```

## 결과

데이터 가져오기 모달의 시간대 입력값이 해당 회차의 추출 시작~종료 시간 범위 밖으로는 선택/입력되지 않도록 제한되었다. 백엔드의 검색 필터 로직은 그대로 유지했고, 프론트엔드 입력 단계에서 회차 설정 범위를 벗어나지 않도록 막는 방식으로 정리했다.
