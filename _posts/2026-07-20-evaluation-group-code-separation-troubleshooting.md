---
layout: post
title: "평가 기능 확장에서 만난 공통코드·응답 계약·캐시 fallback 문제"
date: 2026-07-20 18:30:00 +0900
categories: [개발작업, 트러블슈팅]
tags: [Vue, Spring, QueryDSL, 공통코드, API 보안]
---

평가 기능을 확장하면서 겉으로는 별개처럼 보이는 오류를 함께 다뤘다. 공통코드 목록이 섞이는 문제, 개인 화면에서 점수를 숨기는 문제, 캐시 누락 시 목록이 사라지는 문제가 모두 있었다.

프로젝트와 데이터 이름은 일반화했지만, 해결 과정은 다른 업무 시스템에도 그대로 적용할 수 있다.

## 1. 이름이 비슷한 그룹은 코드 체계부터 분리한다

사용자 정보에는 실제 업무 배정을 위한 그룹과 평가 규칙을 위한 그룹이 함께 있었다. 두 값은 모두 숫자 ID였지만 해석하는 코드 목록이 달랐다.

| 역할 | 예시 사용처 | 필요한 코드 목록 |
| --- | --- | --- |
| 업무 그룹 | 업무 배정, 응대 분류 | 업무용 그룹 |
| 평가 그룹 | 대상자 선정, 회차 규칙 | 평가용 그룹 |

문제는 화면마다 드롭다운, 검색 조건, 그리드 lookup이 따로 구현되어 있다는 점이다. 코드맵 요청만 바꿔서는 충분하지 않다.

```js
const codeMap = await loadCodeMap(['assessment-group']);

const assessmentGroupOptions = getLeafCodes(
  codeMap['assessment-group'],
);

grid.column('assessmentGroupId').lookup = assessmentGroupOptions;
```

선택 목록·검색 값·결과 표시명을 함께 바꿔야 저장된 ID와 화면의 이름이 같은 의미를 유지한다. 특히 회차 예외처럼 업무 규칙이 걸린 값은 숫자 ID를 직접 비교하기보다 **평가용 코드 체계 안에서** 해석해야 한다.

## 2. 점수를 숨길 때는 화면만 가리면 안 된다

개인 평가 화면에서 숫자 점수와 배점을 숨기고 등급만 보여 주는 요구가 있었다. 처음에는 컬럼을 제거하는 것으로 충분해 보이지만, 브라우저 Network 탭에 점수가 남아 있으면 실제로는 숨긴 것이 아니다.

그래서 개인 화면용 조회 응답을 운영자용 조회와 분리했다.

```java
// 운영자용 응답: 점수, 배점, 답변 정보 포함
// 개인용 응답: 등급명과 선택 상태만 포함
public class PersonalAnswerView {
    private Long questionId;
    private Long answerRecordId; // 저장에 필요한 내부 식별자
    private Long selectedOptionId;
    private String answerGradeName;
    private List<OptionGradeView> options;
}
```

핵심은 다음 두 가지다.

1. 합산 점수는 서버에서 기존 등급 기준표로 환산한다.
2. 문항 보기의 점수 대신 보기별로 등록한 `우수·양호·보통·미흡` 등급을 내려준다.

기존 평가표처럼 보기 등급이 없는 데이터는 `미설정`으로 표시하되, 최종 등급은 기존 기준표를 이용해 계속 계산할 수 있게 했다.

## 3. 민감한 값은 빼도 저장에 필요한 식별자는 남겨야 한다

전용 응답을 만들 때 모든 필드를 제거하면 안전해 보인다. 하지만 피드백 저장 기능에는 각 답변 레코드를 가리키는 내부 ID가 필요하다.

코드 리뷰에서 발견한 문제는 다음과 같았다.

```js
// 화면은 저장 시 이 ID가 필요하다.
answers.map(answer => ({
  id: answer.answerRecordId,
  feedbackOptionId: answer.feedbackOptionId,
}));
```

응답 DTO에서 이 ID까지 제거하면 저장 요청의 `id`가 `undefined`가 된다. 점수는 민감한 정보지만 답변 레코드 ID는 저장을 위한 식별자다. 따라서 전용 응답에는 **점수는 제외하고 식별자만 유지**해야 한다.

또한 기존 운영자 API는 의견을 중첩 객체로 반환했는데, 전용 API는 평면 필드로 반환했다. 화면 모델로 변환하는 코드를 명시적으로 두어 계약 차이로 인한 바인딩 오류를 막았다.

```js
// API 응답을 화면 모델로 변환
commentModel.value = {
  evaluationComment: response.evaluationComment ?? '',
  feedbackComment: response.feedbackComment ?? '',
  finalComment: response.finalComment ?? '',
};
```

## 4. 액션 기반 API 호출은 라우팅 등록도 기능의 일부다

일부 프런트엔드 구조는 URL을 직접 호출하지 않고 액션 이름으로 API를 호출한다.

```js
callApi({ actionName: 'PERSONAL_ASSESSMENT_DETAIL' });
```

이 경우 공통 라우팅 테이블에 아래 정보가 있어야 한다.

| 액션명 | HTTP 메서드 | 실제 경로 |
| --- | --- | --- |
| 개인 평가 상세 조회 | GET | `/api/personal/assessment/detail` |
| 개인 피드백 저장 | POST | `/api/personal/assessment/feedback` |

컨트롤러와 화면 코드를 모두 작성했는데도 호출 URL을 찾지 못한다면, 라우팅 메타데이터 등록 여부를 확인해야 한다. 이는 Vue Router와는 다른, 서버 API 호출용 라우팅이다.

## 5. 캐시 오류를 막는 fallback도 검색 조건을 고려해야 한다

평가표 목록에서는 평가 구분의 상위 ID가 필요했다. 성능 개선 과정에서 코드 테이블 조인을 제거하고 공통코드 캐시로 상위 ID를 보완하도록 바뀐 상태였다.

캐시 초기화 전에는 메타데이터를 찾지 못해 목록 API가 시스템 오류가 됐다. 처음에는 예외를 잡고 상위 ID를 `null`로 두어 목록을 반환하도록 바꿨다.

하지만 평가표 선택 팝업은 상위 ID를 검색 조건으로 보낸다.

```java
items = items.stream()
    .filter(item -> requestedParentId.equals(item.getParentCategoryId()))
    .toList();
```

캐시가 없어서 모든 상위 ID를 `null`로 만들면, 이번에는 오류 대신 **목록 전체가 빈 결과**가 된다. 장애 형태만 바뀐 셈이다.

올바른 fallback은 캐시가 없을 때 DB 코드 뷰를 다시 조인해 상위 ID를 얻는 방식이다.

```java
leftJoin(codeView)
    .on(sheet.categoryId.eq(codeView.id))
// codeView.parentId를 함께 projection
```

이제 캐시가 정상일 때는 캐시 값을 활용하고, 캐시가 없을 때도 DB 조회값으로 같은 검색 조건을 유지한다.

## 확인한 항목

- 평가용 그룹과 업무용 그룹의 코드맵·lookup·검색 조건이 같은 의미를 쓰는지
- 개인용 API 응답에 숫자 점수·배점이 포함되지 않는지
- 보기 등급과 최종 등급이 각각 올바르게 표시되는지
- 피드백 저장에 필요한 답변 식별자가 남아 있는지
- 캐시 누락 상태에서도 평가표 검색 조건이 정상 동작하는지
- 백엔드 컴파일과 프런트엔드 빌드가 통과하는지

## 마무리

이번 작업의 공통점은 "값을 숨기거나 최적화하는 변경"이 데이터의 의미와 계약을 함께 바꾼다는 점이었다.

> 공통코드는 값보다 해석 체계가 중요하고, API 보안은 화면이 아니라 응답 계약에서 완성되며, fallback은 원래의 검색 의미를 보존해야 한다.

이 세 가지를 함께 점검하면 비슷한 기능 확장에서도 오류를 훨씬 일찍 잡을 수 있다.
