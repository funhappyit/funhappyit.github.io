---
layout: post
title: "ISR을 켜기 전에 걸린 두 가지 - 캐시에 섞인 로그인 정보와 42% 중복 응답"
date: 2026-11-23 21:50:00 +0900
categories: [사이드프로젝트, tobeol-ledger]
tags: [Nuxt, ISR, Vercel, 성능, 보안, SSR]
---

1만 건짜리 목록을 매 요청마다 DB에서 읽을 수는 없어서 ISR을 켜려 했다.
켜기 직전에 **캐시에 남의 로그인 정보가 섞이는 구조**를 발견했고,
켠 뒤에는 **응답의 42%가 중복**이라는 걸 알게 됐다.

## 1. ISR 캐시에 로그인 정보가 섞일 뻔했다

### 문제 상황

머리글의 로그인 표시 컴포넌트에 이런 주석이 달려 있었다. 내가 쓴 것이다.

```js
/**
 * 헤더의 로그인 상태 표시
 *
 * 로그인 여부는 서버 렌더 시점에도 쿠키로 알 수 있어서 깜빡임 없이 그려진다.
 */
const user = useSupabaseUser();
```

ISR이 없을 때는 맞는 말이다. 요청마다 서버가 그리니 각자 자기 계정이 보인다.

**ISR은 다르다.** 한 번 만든 HTML을 정해진 시간 동안 모든 방문자에게 그대로 돌려준다.

```
사용자 A 가 /dex 방문  →  HTML 생성 (A 의 계정이 박힌 채)  →  캐시
방문자 B 가 /dex 방문  →  캐시된 HTML 전달  →  B 에게 A 의 계정이 보임
```

오류가 나지 않는다. 테스트도 통과한다. 혼자 개발하는 동안에는 드러나지 않는다.

### 해결 방법

로그인 표시를 **브라우저에서만** 그리도록 돌렸다.

```html
<ClientOnly>
  <UserMenu />
  <template #fallback>
    <div class="ml-auto h-[34px] w-20" aria-hidden="true" />
  </template>
</ClientOnly>
```

`fallback`으로 자리를 미리 잡아 두는 게 중요하다. 없으면 메뉴가 뜰 때 머리글이 흔들린다.

그리고 캐시하는 화면을 **누구에게나 같은 것으로** 한정했다.

```ts
routeRules: {
  '/': { isr: 300 },
  '/dex': { isr: 300 },
  '/dex/**': { isr: 300 },
  // 개인 자료가 실리는 화면은 넣지 않는다
}
```

### 검증

주장으로 끝내지 않고 빌드 결과로 확인했다.

```bash
node .output/server/index.mjs &
curl -s http://localhost:3100/dex > page.html

for w in 로그아웃 로그인 계정명 @gmail; do
  echo "$w: $(grep -c "$w" page.html)건"
done
```

```
로그아웃: 0건
로그인:   0건
계정명:   0건
@gmail:   0건
전체 10,875개 ← 굿즈는 정상
```

로그인 UI가 서버 렌더 HTML에 한 글자도 없고 굿즈는 그대로 그려진다.
배포 후 실제 사이트에서도 같은 검사를 했다.

### 인증을 쓰는 곳 전수 조사

같은 문제가 다른 데도 있는지 확인했다.

```bash
grep -rn 'useSupabaseUser\|useSupabaseClient' app/
```

| 사용처 | SSR 영향 |
| --- | --- |
| 머리글 로그인 표시 | **HTML에 계정이 박힘** ← 문제 |
| 목록 화면 | 버튼 클릭 때만 사용 — 안전 |
| 기록 다이얼로그 | 열었을 때만 렌더 — 안전 |
| 토벌 기록 composable | `onMounted` 이후에만 — 안전 |
| 개인 페이지들 | ISR 대상이 아님 |

문제는 한 군데였다. 나머지가 안전했던 이유는 **클릭하거나 마운트된 뒤에만** 사용자 정보를 보기 때문이다.

## 2. 응답의 42%가 중복이었다

### 문제 상황

"도감으로 갈 때 너무 느리다"는 피드백을 받았다. 추측하지 않고 쟀다.

```bash
curl -s -o /dev/null --compressed \
  -w 'TTFB %{time_starttransfer}s  총 %{time_total}s  %{size_download} bytes\n' \
  https://example.app/dex
```

```
/dex 문서          TTFB 66ms   11.6KB   ← 빠르다
/dex/_payload.json             940KB    ← 이것
/api/dex?all       TTFB 381ms  740KB  총 2.06초
```

브라우저 네트워크도 직접 봤다. `_payload.json` 하나만 받고 API를 또 받지는 않았다.
다행히 중복 요청은 없었지만, **940KB를 다 받을 때까지 화면이 바뀌지 않는다.**
그동안 아무 표시가 없으니 "눌렀는데 반응이 없다"로 느껴진다.

### 원인 — 같은 글자를 두 번 보내고 있었다

필드별로 크기를 쟀다.

```
search        2.93MB   42%   ████████████████████
imageUrl      1.09MB   16%   ████████
nameJa        0.94MB   13%   ███████
nameKo        0.79MB   11%   ██████
categoryCodes 0.32MB    4%
…
전체 7.02MB
```

`search`는 검색 대조용으로 서버에서 미리 만들어 두는 문자열이었다.

```ts
search: [item.nameKo, item.nameJa, item.handle, ...item.tagsRaw]
  .join(' ')
  .toLowerCase(),
```

**그 안의 `nameKo`·`nameJa`·`handle`은 같은 응답에 이미 따로 실려 있다.**
1만 건이면 그 중복만 2MB였다.

### 해결 방법

서버는 태그만 보내고, 이름과 합치는 일은 받는 쪽이 한다.

```ts
// 서버
tags: item.tagsRaw.join(' ').toLowerCase(),

// 화면에서 쓰는 합성
export function buildSearchText(item: DexItem): string {
  return [item.nameKo, item.nameJa, item.handle, item.tags]
    .filter((v): v is string => typeof v === 'string' && v.length > 0)
    .join(' ')
    .toLowerCase();
}
```

그냥 이렇게 하면 검색이 느려진다. 글자를 칠 때마다 1만 건을 다시 훑기 때문이다.
**목록이 바뀔 때만** 인덱스를 만들어 두면 양쪽을 피할 수 있다.

```ts
const searchIndex = computed(() => {
  const index = new Map<string, string>();

  for (const item of props.items) {
    index.set(item.handle, buildSearchText(item));
  }

  return index;
});
```

### 압축 후에도 효과가 있는지 확인했다

이미지 주소의 공통 접두사(`https://cdn.example.com/s/files/1/…`)도 줄여 볼까 했다.
하지만 **반복되는 문자열은 압축이 이미 처리한다.** 재 보기로 했다.

```ts
import { gzipSync, brotliCompressSync } from 'node:zlib';

function sizes(label: string, value: unknown) {
  const raw = Buffer.from(JSON.stringify(value), 'utf8');
  console.log(`${label} 원본 ${raw.length} gzip ${gzipSync(raw).length} brotli ${brotliCompressSync(raw).length}`);
}
```

```
고치기 전   원본 7.02MB  gzip 780KB  brotli 487KB
고친 뒤     원본 5.34MB  gzip 627KB  brotli 380KB
            24% 감소     20% 감소    22% 감소
```

중복 제거는 압축 후에도 22% 남았다. 같은 글자가 **멀리 떨어져 있으면** 압축 사전에 잘 안 걸리기 때문이다.

반면 이미지 접두사는 포기했다. 응답 헤더를 확인하니 이미 brotli였다.

```bash
curl -sI -H 'Accept-Encoding: br' https://example.app/api/dex | grep -i content-encoding
# content-encoding: br
```

접두사처럼 **가까이 반복되는 문자열**은 압축이 거의 공짜로 처리한다. 복잡도만 늘 뻔했다.

### 기다리는 동안 표시를 띄운다

크기를 줄여도 1.5초는 걸린다. 그동안 아무것도 안 보이는 게 문제였다.

```html
<NuxtLoadingIndicator color="var(--accent)" :height="3" />
```

실제로 뜨는지 브라우저에서 추적했다.

```js
const el = document.querySelector('.nuxt-loading-indicator');
const samples = [];
const t0 = performance.now();
const timer = setInterval(() => {
  samples.push(`${Math.round(performance.now() - t0)}ms ${getComputedStyle(el).opacity}`);
}, 120);
document.querySelector('a[href="/dex"]').click();
await new Promise((r) => setTimeout(r, 2600));
clearInterval(timer);
```

```
160ms   opacity 0      (아직)
406ms   opacity 0.27   (나타나기 시작)
601ms   opacity 0.94
845ms~  opacity 1
전환 완료 후 → 0
```

0.4초쯤 뒤에 뜨는 건 Nuxt가 짧은 전환에는 일부러 안 띄우기 때문이다. 깜빡임을 막는 쪽이 낫다.

## 결과

```
                      전              후           감소
payload             940KB           780KB         17%
/api/dex?all  740KB · 2.06초  581KB · 1.58초   21% / 23%
```

응답에 무엇을 싣고 무엇을 빼는지가 곧 성능이라, 테스트로 못박아 두었다.

```ts
it('화면이 쓰는 값만 싣는다', () => {
  assert.deepEqual(Object.keys(item).sort(), [
    'categoryCodes', 'handle', 'imageUrl', 'listPriceJpy',
    'nameJa', 'nameKo', 'purchaseLimit', 'releasedOn', 'source', 'tags',
  ]);
});

it('검색용 태그에 이름과 handle 을 섞지 않는다', () => {
  assert.ok(!item.tags.includes('치이카와 클리닉'));
  assert.ok(!item.tags.includes('4570189213295'));
});
```

## 정리

- **ISR은 생성된 HTML을 모든 방문자에게 돌려준다.** 사용자별로 달라지는 부분은 `ClientOnly`로 분리해야 한다
- 이 문제는 오류를 내지 않는다. 빌드 결과 HTML을 직접 grep 해서 확인하는 편이 확실하다
- 캐시 대상은 **누구에게나 같은 화면**으로 한정한다
- 느리다는 말을 들으면 먼저 재야 한다. 문서가 아니라 payload가 범인일 수 있다
- 서버에서 미리 만들어 보내는 파생 값은 **원본과 중복되지 않는지** 확인한다
- 중복 제거가 압축 후에도 효과가 있는지는 `zlib`으로 직접 재 보면 된다.
  **가까이 반복되는 문자열은 압축이 처리하고, 멀리 떨어진 중복은 그렇지 않다**
- 줄여도 남는 대기 시간에는 진행 표시를 띄운다
