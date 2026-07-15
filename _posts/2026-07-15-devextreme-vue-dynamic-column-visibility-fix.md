---
layout: post
title: "DevExtreme 동적 컬럼이 columnOption()으로는 화면에 반영되지 않던 문제"
date: 2026-07-15 21:00:00 +0900
categories: [오류해결, DevExtreme]
tags: [Vue, DevExtreme, devextreme-vue]
---

## 문제 상황

상담사별 평가 결과를 피벗(pivot)해서 보여주는 DevExtreme `DataGrid` 화면이 있었다. 조회 조건(연도, 평가 유형/계획)에 따라 평가계획 단위로 점수 컬럼이 동적으로 생겨야 하는 구조라, 최대 개수(예: 60개)만큼 컬럼을 미리 `visible: false`로 선언해두고, 조회 결과에 맞춰 필요한 만큼만 `caption`/`visible`을 켜주는 방식으로 구현돼 있었다.

그런데 조회를 하면 Network 탭에는 점수 데이터가 분명히 내려오는데도, 그리드에는 이름/소속 같은 기본 정보 컬럼까지만 보이고 점수 컬럼은 전혀 나타나지 않는 현상이 있었다.

## 원인

컬럼을 켜는 로직이 그리드 위젯 인스턴스를 직접 얻어와서 다음과 같이 DevExtreme 위젯 옵션을 명령형으로 바꾸는 방식이었다.

```js
const instance = gridRef.value?.getInstance;
instance?.columnOption('score_0', 'visible', true);
```

문제는 이 방식이 devextreme-vue가 화면을 실제로 그릴 때 참조하는 "선언형 컬럼 소스"(부모 컴포넌트가 내려준 `columns` prop, 그리드 wrapper 내부에서는 별도의 반응형 상태로 보관)를 전혀 건드리지 않는다는 점이다. devextreme-vue는 `<dx-column>` 하위 컴포넌트가 **자기 자신의 prop 값이 실제로 바뀔 때만** 위젯에 값을 재적용하는 구조라서, "위젯에는 값을 직접 넣었지만 Vue가 추적하는 원본 컬럼 데이터는 여전히 `visible: false`"인 상태로 어긋나 있었던 것이다.

단서는 이미 코드 안에 있었다. 그리드 wrapper 컴포넌트의 엑셀 내보내기 로직에는 이런 주석이 남아있었다.

> 동적 컬럼(평가계획별 컬럼 등)은 반응형 `columns` 상태가 아니라, 위젯의 실시간 옵션(`component.option('columns')`)에서 읽어야 한다 — `columnOption()`으로 런타임에 바꾼 값은 반응형 상태에 반영되지 않기 때문.

즉 "`columnOption()`만으로는 반응형 소스가 갱신되지 않는다"는 사실은 이미 알려져 있었고 엑셀 내보내기 쪽은 그 문제를 우회해뒀지만, 정작 **화면을 그리는 경로**는 그 문제를 우회하지 못한 채로 남아있었던 게 이번 버그의 정체였다.

## 해결 방법

`instance.columnOption()` 호출을 걷어내고, 부모가 그리드에 넘기는 반응형 `columns` 배열의 해당 컬럼 객체를 직접 mutate하도록 바꿨다.

```js
// Before - 위젯 인스턴스에 직접 명령형으로 값을 밀어넣음 (Vue 반응형 소스는 안 바뀜)
const applyDynamicScoreColumns = scheduleItems => {
  const instance = gridRef.value?.getInstance;
  for (let idx = 0; idx < MAX_SCORE_COLUMNS; idx++) {
    const field = `score_${idx}`;
    if (idx < scheduleItems.length) {
      instance?.columnOption(field, 'caption', scheduleItems[idx].scheduleName);
      instance?.columnOption(field, 'visible', true);
    } else {
      instance?.columnOption(field, 'visible', false);
    }
  }
};

// After - 실제로 화면이 읽는 반응형 columns 배열 자체를 갱신
const applyDynamicScoreColumns = scheduleItems => {
  for (let idx = 0; idx < MAX_SCORE_COLUMNS; idx++) {
    const field = `score_${idx}`;
    const column = gridConfig.columns.find(c => c.dataField === field);
    if (!column) continue;

    if (idx < scheduleItems.length) {
      column.caption = scheduleItems[idx].scheduleName;
      column.visible = true;
    } else {
      column.visible = false;
    }
  }
};
```

`gridConfig`가 `reactive()` 객체이고, 그리드 wrapper 컴포넌트가 마운트 시점에 부모가 내려준 `columns` 배열을 그대로(얕은 복사) 내부 상태에 연결해서 쓰는 구조라면, 이렇게 **원본 배열 요소를 직접 바꾸는 것**이 devextreme-vue의 선언형 바인딩 경로를 그대로 타게 되어 위젯에도 정확히 반영된다. 위젯 인스턴스를 거치지 않고 Vue 반응형 시스템만으로 값이 전달되는 셈이라, 인스턴스가 아직 준비되지 않았을 수 있는 타이밍 문제에서도 자유로워진다.

## 결과

- 조회 즉시 평가계획명이 caption으로 붙은 점수 컬럼이 정상적으로 나타남.
- 위젯 인스턴스 참조(`ref`)와 `nextTick()` 대기 로직을 함께 걷어낼 수 있어 코드도 더 단순해짐.
- 정리하면: devextreme-vue로 동적 컬럼(caption/visible 등)을 다뤄야 한다면 위젯 인스턴스의 `columnOption()`보다 **컴포넌트에 내려주는 반응형 `columns` 소스를 직접 바꾸는 쪽**이 화면 표시와 이후 로직(엑셀 내보내기 등) 모두에서 일관되게 동작한다.
