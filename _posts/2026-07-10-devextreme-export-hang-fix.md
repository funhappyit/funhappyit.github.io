---
layout: post
title: "DevExtreme 마스터-디테일 그리드에서 엑셀 전체 다운로드가 조용히 멈추던 문제"
date: 2026-07-10 18:00:00 +0900
categories: [오류해결, DevExtreme]
tags: [Vue, DevExtreme, ExcelJS]
---

## 문제 상황

서버 페이징(`remoteOperations`)과 마스터-디테일(행을 펼치면 하위 상세 그리드가 나오는 구조)이 함께 켜진 DevExtreme `DataGrid`에서, "전체 다운로드" 버튼을 눌러도 아무 반응이 없었다. 에러도, 다운로드도 없이 그냥 멈췄다.

엑셀 다운로드는 요약 시트 + 상세 시트 두 장을 만드는 커스텀 `onExporting` 핸들러로 구현되어 있었다.

## 원인

### 1. 하위 행이 펼쳐진 상태에서 `dataSource`를 통째로 교체하며 발생한 크래시

"전체 다운로드" 시 서버에서 페이징 없이 전체 데이터를 다시 조회해, 그 결과로 그리드의 `dataSource`를 교체했다. 그런데 이 재조회 결과에는 마스터-디테일 렌더링에 필요한 메타 정보가 빠져 있었다.

하위 상세 행이 **펼쳐진 상태**에서 이 교체가 일어나면, 그리드 라이브러리가 해당 메타 정보를 읽으려다 `TypeError`가 발생했다. `try/catch`가 없어서 콘솔에도 안 뜨고 그냥 조용히 실패했다(unhandled rejection).

```js
// 문제가 됐던 흐름 (예시)
// 정상 조회 결과에는 masterDetail 메타가 있지만,
// export 전용 재조회 결과에는 이게 빠져 있었다.
component.option('dataSource', summaryRowsWithoutMasterDetailMeta);
// → 펼쳐진 행이 있으면 렌더링 시점에 여기서 TypeError
```

### 2. `exportDataGrid()` + `dataSource` 스왑 조합이 응답 없이 멈추는 문제

1번을 고친 뒤에도 "재조회 API는 성공하는데 그 이후 아무 반응이 없다"는 증상이 남아있었다. 네트워크 탭으로 확인해보니 재조회 API 호출은 정상 응답을 받았지만, 그 다음 단계는 아예 실행되지 않고 있었다.

원인은 `exportDataGrid()`(DevExtreme 유틸)에 `dataSource`를 바꿔치기한 그리드 인스턴스를 그대로 넘기는 방식 자체였다. 서버 사이드 페이징/필터/정렬 + 마스터-디테일이 동시에 켜진 조합에서는 이 유틸이 내부적으로 데이터 반영을 기다리다 멈춰버렸다.

```js
// 문제가 됐던 코드 (예시)
const oldDataSource = component.option('dataSource');
component.option('dataSource', newRows);
await exportDataGrid({ component, worksheet }); // 여기서 멈춤
component.option('dataSource', oldDataSource);
```

## 해결 방법

**1. 다운로드 시작 전, 펼쳐진 하위 행을 모두 접는다**

```js
const collapseAllExpandedRows = instance => {
  instance.getVisibleRows().forEach(row => {
    if (instance.isRowExpanded(row.key)) {
      instance.collapseRow(row.key);
    }
  });
};
```

**2. `exportDataGrid()` 대신 ExcelJS로 데이터를 직접 시트에 기입한다**

그리드 인스턴스의 `dataSource`를 아예 건드리지 않고, 조회한 데이터 배열을 그대로 엑셀 셀에 써넣는 방식으로 바꿨다.

```js
const buildWorksheet = (workbook, rows, columns) => {
  const sheet = workbook.addWorksheet('Sheet1');
  sheet.columns = columns.map(c => ({ header: c.header, key: c.key, width: 14 }));
  rows.forEach(row => sheet.addRow(row));
  sheet.getRow(1).font = { bold: true };
  return sheet;
};
```

**3. `try/catch` 안전망 추가**

이후 예상 못한 오류가 나더라도 조용히 사라지지 않고, 사용자에게 실패 사실이 보이도록 처리했다.

```js
try {
  // 시트 구성, 파일 저장
} catch (error) {
  console.error('엑셀 다운로드 실패:', error);
  showErrorMessage('엑셀 다운로드 중 오류가 발생했습니다.');
}
```

## 결과

- 하위 행이 펼쳐진 상태여도, 아니어도 "전체 다운로드"가 정상 동작
- export 로직이 그리드 인스턴스의 현재 상태(펼침 여부, dataSource)에 의존하지 않게 되어 구조적 리스크 자체가 사라짐
- 실패해도 최소한 에러 메시지는 사용자에게 노출됨

### 배운 점

`exportDataGrid()`처럼 "라이브 그리드 인스턴스를 그대로 넘기는" export 유틸은, 그리드 설정이 단순(마스터-디테일 없음, 클라이언트 사이드 페이징)할 때는 문제없이 동작한다. 하지만 서버 사이드 오퍼레이션 + 마스터-디테일처럼 복잡한 조합이 섞이면 예측 못한 방식으로 멈출 수 있다. 이럴 땐 프레임워크 유틸에 기대기보다 "조회된 데이터 배열을 export 라이브러리(ExcelJS 등)로 직접 기입"하는 패턴이 더 안전하다.
