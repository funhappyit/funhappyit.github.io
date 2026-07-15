---
layout: post
title: "DevExtreme 클라이언트 사이드 그리드에 계산 컬럼 + 헤더필터 추가하기"
date: 2026-07-15 18:00:00 +0900
categories: [개발작업, 일일로그]
tags: [Vue, DevExtreme, esp-ui]
---

## 오늘 한 일

- **상담원 목록 그리드에 "구분"(그룹A/그룹B) 필터 컬럼 추가**
  - 기존 화면에는 부서(센터/팀/섹션) 컬럼과 "계약형태" 컬럼(공통코드 기반 정규직/계약직/파견/외주 등)이 이미 있었는데, 여기에 "센터" 컬럼 바로 옆에 "구분" 컬럼을 새로 추가.
  - 요구사항: 구분값은 **전체 / 그룹A / 그룹B** 3가지. 그룹B는 "외주" 계약형태 하나만, 그룹A는 그 외 나머지 계약형태(정규직/계약직/파견 등)를 전부 포함.
  - 이 그리드는 `remoteOperations`가 전부 `false`(필터링·정렬·페이징이 모두 클라이언트 사이드)인 구조라, 서버 API나 파라미터 변경 없이 **프론트 계산 컬럼**만으로 해결.
  - 이미 로드되어 있던 공통코드 목록에서 "외주" codeId를 한 번만 찾아두고, 각 행의 계약형태 코드가 그 값과 일치하면 그룹B, 아니면 그룹A로 판정하는 헬퍼 함수를 만들어 컬럼의 `calculateCellValue`/`calculateFilterExpression`에 연결.
  - "전체" 옵션은 별도 구현 없이 DevExtreme 헤더필터가 기본 제공하는 "(모두 선택)"으로 자연스럽게 처리됨.

## 기술 메모: 클라이언트 사이드 그리드에서 "실제 컬럼이 없는" 필터 만들기

DevExtreme의 `dxDataGrid`는 원격 필터링(`remoteOperations.filtering: true`)을 쓰지 않는 경우, 화면에 로드된 데이터 전체를 grid가 직접 필터링한다. 이 방식에서는 데이터에 존재하지 않는 "가상 컬럼"도 다음 두 옵션만 채워주면 필터행/헤더필터가 그대로 동작한다.

```js
// 기존 공통코드 목록(예: 계약형태 코드 목록)에서 특정 코드의 id를 한 번만 찾아둔다
const outsourcedCodeId = employmentTypeList.find(c => c.codeNm === '외주')?.codeId;

// 행 데이터 -> 표시 그룹명("그룹A" or "그룹B") 판정 함수
const getEmploymentGroupLabel = rowData =>
  Number(rowData?.employmentTypeCd) === Number(outsourcedCodeId) ? '그룹B' : '그룹A';

const column = {
  caption: '구분',
  dataField: 'employmentGroupLabel', // 실제 데이터에는 없는 가상 필드명
  allowFiltering: true,
  allowHeaderFiltering: true,

  // 화면에 표시할 값
  calculateCellValue: rowData => getEmploymentGroupLabel(rowData),

  // 필터행 입력값 / 헤더필터 선택값이 들어왔을 때 어떻게 매칭할지
  calculateFilterExpression: filterValue => {
    if (!filterValue) return null;
    const lower = String(filterValue).toLowerCase();
    return [rowData => getEmploymentGroupLabel(rowData).toLowerCase().includes(lower), '=', true];
  },
};
```

포인트는 `headerFilter.dataSource`를 직접 지정하지 않아도 된다는 것. 클라이언트 사이드 데이터에서는 DevExtreme이 `calculateCellValue`로 계산된 값들(여기서는 "그룹A"/"그룹B") 중 실제로 존재하는 값만 자동으로 옵션 목록을 만들어 주고, 그 위에 "(모두 선택)"이 기본으로 붙는다. 즉 "전체/그룹A/그룹B" 3단계 요구사항 중 "전체"는 신경 쓸 필요가 없다.

또 하나, 컬럼을 원하는 위치(부서 컬럼들 중간)에 끼워 넣어야 했는데, 여러 컬럼을 한 번에 만들어주는 유틸 함수(`createDeptColumns()` 같은)를 배열 스프레드로 통째로 넣던 기존 코드를 배열 인덱스로 풀어서, 원하는 순서로 재배치했다.

```js
const deptColumns = createDeptColumns(); // [센터, 팀, 섹션]

columns: [
  deptColumns[0], // 센터
  column,         // 구분 (센터 바로 옆)
  deptColumns[1], // 팀
  deptColumns[2], // 섹션
  // ...
]
```

## 이슈 / 특이사항

- 원격 필터링을 쓰는 다른 화면(서버 페이징/필터링)이었다면 이렇게 간단히 끝나지 않고, 백엔드 쿼리 조건과 API 파라미터까지 함께 손봐야 했을 것. 화면별로 `remoteOperations` 설정을 먼저 확인하는 게 작업 범위를 잘못 잡지 않는 지름길이었다.
- 요구사항을 처음 전달받았을 때는 "필터 위치가 정확히 어디인지"(검색영역에 별도 토글로 넣을지, 그리드 컬럼으로 넣을지)가 불명확해서 먼저 확인하고 진행함 - 같은 코드베이스 안에 이미 비슷한 "전체/그룹A/그룹B" 형태의 토글 필터가 다른 화면(월별 조회 화면)에 구현되어 있어 혼동하기 쉬운 상황이었다.
