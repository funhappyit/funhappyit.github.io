---
layout: post
title: "DevExtreme 그리드 필터 오류를 해결하며 정리한 데이터 계약 원칙"
date: 2026-07-21 21:30:00 +0900
categories: [개발작업, 트러블슈팅]
tags: [Vue, DevExtreme, Spring Boot, QueryDSL, 페이징, 공통코드]
---

오늘은 인사·근무 관리 화면의 목록, 검색, 저장 흐름을 점검했다. 겉으로는 “필터에 값이 하나만 보인다”, “날짜 검색이 안 된다”, “페이지를 바꾸면 결과가 달라진다” 같은 UI 문제였지만, 실제 원인은 화면·그리드·API·DB가 서로 다른 값을 기준으로 동작한 데 있었다.

이번 글에서는 특정 업무 데이터 대신, 같은 유형의 관리자 화면에 재사용할 수 있는 해결 원칙을 정리한다.

## 1. 표시명과 검색값은 분리해야 한다

공통코드 컬럼은 보통 DB에 코드값을 저장하고 화면에는 코드명을 보여 준다.

```text
DB: employmentTypeCd = 103
화면: 계약직
```

처음에는 `lookup` 하나로 표시와 필터를 동시에 해결하려 했다. 하지만 원격 페이징 그리드에서는 lookup이 현재 로드된 행을 바탕으로 필터 선택지를 축소할 수 있다. 첫 페이지에 없는 값은 필터에 나타나지 않고, 코드명이 아니라 코드값을 서버로 보내야 하는 컬럼에서는 검색 조건도 어긋날 수 있다.

그래서 역할을 분리했다.

| 역할 | 기준 값 |
| --- | --- |
| 셀 표시 | 공통코드명 |
| 필터 선택지 | 공통코드 전체 목록 |
| API 요청 | 실제 코드값 |
| DB 조건 | 실제 코드값 |

```js
const employmentTypeOptions = getLeafCodes(codeMap['employment-type']);

const column = {
  dataField: 'employmentTypeCd',
  customizeText: cellInfo =>
    employmentTypeOptions.find(code => code.codeId === cellInfo.value)?.codeNm ?? '',
  headerFilter: {
    dataSource: employmentTypeOptions.map(code => ({
      text: code.codeNm,
      value: code.codeId,
    })),
  },
};
```

핵심은 **표시용 이름을 필터의 원본 값으로 사용하지 않는 것**이다.

## 2. 공통 그리드의 필터 에디터는 화면별로 검증해야 한다

같은 DevExtreme 옵션이라도 데이터 소스 구성이 다르면 결과가 달라진다.

- 일반 배열 데이터 소스에서는 `dxSelectBox` 에디터가 잘 동작할 수 있다.
- 원격 `CustomStore`와 서버 필터링을 함께 쓰는 화면에서는 초기 필터값 또는 에디터 옵션 변경이 곧바로 재조회 조건에 영향을 줄 수 있다.
- lookup은 편하지만 현재 페이지 값만 노출하는 문제가 생길 수 있다.

즉, “모든 화면에서 lookup을 없애고 SelectBox로 바꾸자”는 안전한 해결책이 아니었다. 원격 목록에서는 아래 순서로 확인해야 한다.

1. 컬럼의 실제 데이터 필드가 코드값인지 확인한다.
2. 필터 선택지가 전체 코드 목록인지 확인한다.
3. 선택 직후 `loadOptions.filter`가 어떤 값으로 만들어지는지 확인한다.
4. API 요청 파라미터와 DB 조건이 같은 값 체계를 사용하는지 확인한다.
5. 변경 후 초기 목록 요청이 비어 있지 않은지 확인한다.

필터 UI를 수정한 뒤 목록 자체가 비어 보인 경우에는, 우선 정상 조회 경로를 복구하고 필터 적용 방식을 화면의 데이터 소스 구조에 맞춰 다시 설계하는 편이 낫다.

## 3. 날짜 필터의 달력과 서버 검색 형식은 별개다

변경일시 컬럼에 `dataType: 'datetime'`을 지정하면 DevExtreme은 날짜·시간 선택기를 만든다. 화면 요구가 “달력 없이 직접 입력”이라면 데이터 타입과 서버 처리도 함께 바꿔야 한다.

```js
{
  dataField: 'editDt',
  dataType: 'string',
  customizeText: cellInfo =>
    String(cellInfo.value ?? '').replace('T', ' ').replace(/\.\d+$/, ''),
}
```

서버에서는 입력값에서 숫자만 추출하고 날짜 범위로 변환했다.

```java
String digits = value.replaceAll("[^0-9]", "");
LocalDate date = LocalDate.parse(digits.substring(0, 8), DATE_FMT);

builder.and(changedAt.goe(date.atStartOfDay()));
builder.and(changedAt.lt(date.plusDays(1).atStartOfDay()));
```

이 방식이면 `2026-07-21`, `2026/07/21`, `20260721`처럼 입력 형식이 조금 달라도 해당 날짜 전체를 일관되게 검색할 수 있다.

## 4. 현재 페이지 필터와 전체 DB 필터를 의도적으로 구분한다

그리드가 서버 페이징을 사용하면 현재 화면에 100건이 보이더라도 실제 검색 결과는 수백 건일 수 있다. 이때 필터가 어디에서 동작하는지를 명확하게 정해야 한다.

- 코드, 사번, 성명, 날짜처럼 업무 검색에 사용하는 필터: 서버로 보내 전체 DB에서 검색
- 화면에서 임시 확인하는 계산 컬럼: 현재 페이지에서 로컬 검색

서버 필터라면 목록 쿼리와 count 쿼리가 반드시 같은 조건을 공유해야 한다.

```java
BooleanBuilder condition = buildSearchCondition(search);

List<Row> rows = queryFactory
    .select(projection)
    .from(entity)
    .where(condition)
    .offset(page.getOffset())
    .limit(page.getLimit())
    .fetch();

Long totalCount = queryFactory
    .select(entity.count())
    .from(entity)
    .where(condition)
    .fetchOne();
```

이 조건이 다르면 “검색 결과는 20건인데 전체 건수는 수백 건”처럼 페이지 UI가 틀어지는 문제가 생긴다.

## 5. 집계 화면은 먼저 페이지를 확정한 뒤 조인한다

개인별 근무 집계처럼 상담원 마스터와 연간 근태 이력을 조인하는 화면은 전체 대상자를 먼저 그룹화하면 느려질 수 있다.

기본 정렬에서는 다음 순서가 더 효율적이다.

1. 상담원 마스터 조건으로 현재 페이지의 상담원만 조회한다.
2. 현재 페이지의 상담원 ID에 대해서만 근태 이력을 조인한다.
3. 해당 페이지에서만 집계한다.

다만 근무일수처럼 집계값 자체를 기준으로 정렬할 때는 전체 집계 후 정렬해야 정확한 페이지 순서를 보장할 수 있다. 성능 최적화는 한 가지 쿼리로 통일하기보다, **정렬 기준에 따라 정확도와 비용을 나누는 문제**였다.

## 6. 저장 검증은 API 오류보다 먼저 보여 준다

필수 입력값이 비어 있는 상태에서 저장 요청을 보내면 사용자는 단순한 입력 실수를 서버 장애처럼 보게 된다. 저장 전에는 공통 검증으로 요청을 막고, 필드 오류와 안내 메시지를 함께 보여 주도록 정리했다.

```js
const validation = validationGroup.instance.validate();
if (!validation.isValid) {
  showAlert(validation.brokenRules[0].message);
  return;
}

await save();
```

사용자 입력 오류는 클라이언트에서, 실제 서버 예외는 공통 오류 처리에서 구분하는 것이 중요하다.

## 마무리

이번 작업에서 가장 중요한 점은 UI 컴포넌트 옵션 하나를 바꾸는 것보다 데이터 계약을 먼저 맞추는 일이었다.

> 화면에는 이름을 보여 주고, 필터와 API는 코드값을 사용하며, 목록·건수·다운로드는 같은 검색 조건을 공유해야 한다.

이 기준을 지키면 페이징, 공통코드, 날짜 입력, 엑셀 다운로드처럼 서로 다른 기능도 같은 방향으로 안정화할 수 있다.
