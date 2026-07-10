---
layout: post
title: "평가 콜가져오기 화면의 추출통화시간 입력 범위 제한 누락 문제 해결"
date: 2026-07-10 18:00:00 +0900
categories: [오류해결]
tags: [Vue, DevExtreme, esp-ui, esp-ewm-api]
---

## 문제 상황

평가관리 > 콜가져오기(녹취콜 리스트) 모달에는 "추출통화시간"이라는 시간 범위 입력창(`frRecTime` ~ `toRecTime`)이 있다. 이 값은 모달을 열 때 회차관리(`schedule/detail`) 화면에서 설정한 해당 회차의 추출시작시간~추출종료시간(`recStartTime`/`recEndTime`)으로 초기화된다.

문제는 이 입력창이 아무런 제약 없이 편집 가능한 `dx-date-box`로 열려 있었다는 점이다. 사용자가 회차에서 정한 범위와 무관하게 임의의 시간대를 입력해도 그대로 검색이 수행되어, "회차의 추출 대상 시간대"라는 원래 의미가 무색해지는 구조였다.

같은 회차 데이터인 최소상담시간/최대상담시간(`recMinTime`/`recMaxTime`)은 애초에 화면에 입력창 자체가 없어 수정이 불가능한데, 유독 추출통화시간만 자유롭게 바뀔 수 있어 일관성도 없었다.

## 원인

프론트엔드(`modal-eval-record-list.vue`)의 검색 로직을 추적해보면:

```js
// selectDataList() 발췌
searchType.paramsData['frRecTime'] = moment(searchType.customTypes.frRecTime).format('HHmmSS');
searchType.paramsData['toRecTime'] = moment(searchType.customTypes.toRecTime).format('HHmmSS');
```

사용자가 입력창 값을 바꾸면 그 값이 그대로 `EWM_EVALUATION_STATUS_RECORD_LIST` API 파라미터로 전송된다. 백엔드(`EvalRecordRepositorySupport.java`)도 이 값을 실제 WHERE 조건에 사용한다.

```java
evalRecord.recStartTime.between(searchDto.getFrRecTime(), searchDto.getToRecTime()),
evalRecord.recEndTime.between(searchDto.getFrRecTime(), searchDto.getToRecTime()),
```

즉 백엔드는 정상적으로 필터링을 수행하고 있었지만, 프론트엔드에서 입력값 자체를 회차 범위 내로 제한하지 않아 "회차에서 정한 시간대"라는 전제가 UI 단에서 지켜지지 않는 상태였다.

## 해결 방법

`dx-date-box`의 `min`/`max` prop을 이용해 입력 가능 범위를 회차의 추출시작~종료시간으로 제한했다.

1. 회차 데이터 조회 시 `recStartTime`/`recEndTime`을 Date 객체로 변환해 별도 상태(`timeBounds.min`/`timeBounds.max`)에 저장한다.

```js
const timeBounds = reactive({
  min: null,
  max: null,
});
```

```js
// selectRoundData() 발췌
const startDate = new Date(today);
startDate.setHours(startHours);
startDate.setMinutes(startMinutes);
const recStartDateTime = moment(startDate).format('YYYY-MM-DD HH:mm:SS');

const endDate = new Date(today);
endDate.setHours(endHours);
endDate.setMinutes(endMinutes);
const recEndDateTime = moment(endDate).format('YYYY-MM-DD HH:mm:SS');

timeBounds.min = startDate;
timeBounds.max = endDate;
```

기존 코드는 `today`라는 Date 객체 하나를 `setHours`/`setMinutes`로 재사용(mutate)하면서 시작/종료 시각을 순차적으로 계산했다. 문자열(`moment().format()`)로 즉시 스냅샷을 뜨는 용도로만 쓸 때는 문제가 없었지만, `min`/`max`처럼 Date 객체 참조 자체를 보관해야 하는 상황에서는 두 값이 같은 객체를 가리키게 되는 문제가 생긴다. 그래서 `startDate`/`endDate`로 객체를 분리했다.

2. 템플릿의 추출통화시간 `dx-date-box` 2개에 `min`/`max`를 바인딩한다.

```vue
<dx-date-box
  v-model="searchType.customTypes.frRecTime"
  type="time"
  display-format="HH:mm"
  :min="timeBounds.min"
  :max="timeBounds.max"
/>
<dx-date-box
  v-model="searchType.customTypes.toRecTime"
  type="time"
  display-format="HH:mm"
  :min="timeBounds.min"
  :max="timeBounds.max"
/>
```

## 결과

콜가져오기 모달의 추출통화시간 입력값이 해당 회차의 추출시작~종료시간 범위 밖으로는 선택/입력되지 않도록 제한되었다. 백엔드의 검색 필터 로직은 그대로 유지했고, 프론트엔드 입력 단계에서 회차 설정 범위를 벗어나지 않도록 막는 방식으로 정리했다.
