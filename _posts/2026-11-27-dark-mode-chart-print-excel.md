---
layout: post
title: "다크 모드 차트를 인쇄·엑셀로 내보낼 때 - 화면 밖 밝은 테마 렌더링과 Listbox 선택 해제 방지"
date: 2026-11-27 13:00:00 +0900
categories: [오류해결, ExcelJS]
tags: [Vue, AG Charts, ExcelJS, PrimeVue, 다크모드, 엑셀다운로드]
---

다크 모드를 지원하는 대시보드에서 센서 상세 팝업을 손보면서 나온 문제 세 가지와 해결을 정리한다.

1. 다크 모드에서 차트 범례 글자가 안 보인다.
2. 다크 모드에서 인쇄하면 흰 종이에 흰 글자 차트가 찍힌다.
3. PrimeVue Listbox에서 선택된 항목을 다시 누르면 선택 표시가 사라진다.

엑셀 다운로드도 이때 추가했다.

## 문제 1. 범례 글자가 배경에 묻힌다

### 원인

축 라벨 색은 테마에 맞춰 직접 지정했지만, 범례는 AG Charts 기본값(검정 글자)이었다.

### 해결

차트 테마도 다크로 바꾸고, 범례 라벨 색을 축 라벨과 같은 값으로 지정했다.

```ts
{
  theme: isDark ? 'ag-default-dark' : 'ag-default',
  legend: {
    item: { label: { color: labelColor } },
  },
}
```

## 문제 2. 다크 모드에서 인쇄하면 차트가 안 보인다

### 원인

인쇄용 이미지를 **화면 캔버스를 그대로 캡처**해서 만들고 있었다.
다크 모드 화면의 차트는 투명 배경에 밝은 글자이므로, 흰 종이에 출력하면 글자가 사라진다.

### 해결: 옵션 생성 함수 분리 + 화면 밖 렌더링

먼저 차트 옵션을 `computed` 하나에서 **색상을 인자로 받는 함수**로 바꿨다.
화면용과 출력용 차트가 같은 정의를 공유하게 하기 위해서다.

```ts
type ChartColors = { label: string; grid: string; background: string; theme: string };

const buildChartOptions = (c: ChartColors) => {
  if (data.length === 0) return null;
  return {
    data,
    theme: c.theme,
    background: { fill: c.background },
    axes: { /* label.color = c.label, gridLine.stroke = c.grid */ },
    legend: { item: { label: { color: c.label } } },
  };
};

// 화면용: 현재 테마
const viewOptions = computed(() => buildChartOptions(currentThemeColors()));
```

출력용은 화면 밖에 임시 컨테이너를 만들고, 밝은 테마·전체 구간·고정 크기로 새로 그린 뒤 PNG로 뽑는다.

```ts
import { AgCharts } from 'ag-charts-enterprise';

const W = 1200, H = 480;

const renderLightChartImage = async (): Promise<string | null> => {
  const options = buildChartOptions({
    label: '#374151', grid: '#e5e7eb', background: '#ffffff', theme: 'ag-default',
  });
  if (!options) return null;

  const box = document.createElement('div');
  box.style.cssText = `position:fixed;left:-10000px;top:0;width:${W}px;height:${H}px;`;
  document.body.appendChild(box);

  let chart: ReturnType<typeof AgCharts.create> | null = null;
  try {
    chart = AgCharts.create({ ...options, container: box, width: W, height: H, zoom: { enabled: false } });
    await chart.waitForUpdate();
    return await chart.getImageDataURL({ fileFormat: 'image/png' });
  }
  catch {
    return null;
  }
  finally {
    chart?.destroy();
    box.remove();
  }
};
```

- 화면에서 줌을 해 둔 상태와도 무관하게 **전체 구간**이 출력된다.
- 인쇄 쪽은 이 함수를 먼저 쓰고, 실패하면 예전처럼 화면 캔버스 캡처로 대체한다.

## 엑셀 다운로드 추가

인쇄물과 같은 구성으로 ExcelJS 시트를 만든다: 제목 → 기본 정보 → 차트 이미지 → 측정 기록 표.

```ts
const { default: ExcelJS } = await import('exceljs'); // 필요할 때만 로드
const wb = new ExcelJS.Workbook();
const ws = wb.addWorksheet('상세');

// 차트 이미지
const img = await renderLightChartImage();
if (img) {
  const id = wb.addImage({ base64: img, extension: 'png' });
  const w = 780, h = Math.round(w * H / W);
  ws.addImage(id, { tl: { col: 1, row: rowNo - 1 }, ext: { width: w, height: h } });
  rowNo += Math.ceil(h / 20) + 1; // 기본 행 높이 20px 기준으로 이미지가 차지하는 행 건너뛰기
}

// 측정치는 숫자로 저장하고 표시 형식만 지정
cell.value = Number(row.value);
cell.numFmt = '0.00';

// 인쇄 설정: A4 가로, 너비 1페이지, 머리글 행 반복
ws.pageSetup = {
  paperSize: 9, orientation: 'landscape',
  fitToPage: true, fitToWidth: 1, fitToHeight: 0,
  printTitlesRow: `${headerRowNo}:${headerRowNo}`,
};
ws.views = [{ showGridLines: false }];
```

- ExcelJS는 동적 import로 불러와 초기 번들에 포함되지 않게 했다.
- 값은 문자열이 아니라 **숫자로 넣고 `numFmt`로 소수 자리만 맞춘다.** 그래야 엑셀에서 계산·정렬이 된다.
- 파일명에 쓸 수 없는 문자(`\ / : * ? " < > |`)는 `_`로 바꾼다.

## 문제 3. Listbox에서 선택이 풀린다

### 원인

PrimeVue Listbox는 선택된 항목을 다시 클릭하면 `update:modelValue`로 `null`을 보내고, **내부 선택 상태도 비운다.**
기존 코드는 `null`을 무시만 했다.

```vue
<Listbox :model-value="selected" @update:model-value="v => { if (v) selected = v; }" />
```

부모의 `selected`는 그대로지만 Listbox 내부 값은 이미 `null`이라, **데이터는 선택된 상태인데 화면에서는 선택 표시가 사라진다.** 항목이 하나뿐일 때 특히 눈에 띈다.

### 해결

`null`이 오면 다음 틱에 현재 값을 Listbox에 다시 써 넣는다.

```ts
const listbox = ref<{ writeValue?: (v: unknown) => void } | null>(null);

const onChange = (v: unknown) => {
  if (v) {
    selected.value = v;
    return;
  }
  nextTick(() => listbox.value?.writeValue?.(selected.value));
};
```

`writeValue`는 PrimeVue 입력 컴포넌트 공통 베이스에 있는 메서드라 공개 API라고 보긴 어렵다. 버전 업 시 확인이 필요하다.

## 덤: 표시 안 함으로 설정한 항목이 하단 바에 계속 나온다

화면 하단의 데이터 티커가 "표시 안 함"으로 설정한 채널까지 보여 주고 있었다.
서버 조회 결과에 표시 여부 필드가 **아예 빠져 있어서** 클라이언트가 걸러낼 수 없었다. 서버 응답에 필드를 추가하고 티커에서 필터링했다.

```ts
for (const item of channels) {
  if (item.visible === 'N') continue;
  // ...
}
```

## 결과

- 다크 모드에서도 범례가 보인다.
- 인쇄·엑셀 차트는 화면 테마와 상관없이 항상 밝은 테마, 전체 구간으로 나온다.
- Listbox는 항상 하나가 선택된 상태로 유지된다.

### 정리

- **화면 캡처로 출력물을 만들면 화면 상태(테마, 줌)가 그대로 따라온다.** 출력용은 따로 그리는 편이 안전하다.
- 같은 차트를 두 번 그릴 때는 옵션 생성을 함수로 빼서 정의를 하나로 유지한다.
- 컨트롤드 컴포넌트처럼 쓰더라도, 내부 상태를 따로 갖는 UI 라이브러리 컴포넌트는 "값을 무시"하는 것만으로 되돌아가지 않는다.
