---
layout: post
title: "목록 API가 막힌 스토어에서 상품 데이터 가져오기 - sitemap과 JSON-LD"
date: 2026-11-23 21:10:00 +0900
categories: [사이드프로젝트, tobeol-ledger]
tags: [데이터수집, robots.txt, JSON-LD, schema.org, TypeScript, Nuxt]
---

굿즈 도감에 스토어를 하나 더 붙였다.
그런데 이 스토어는 상품 목록을 주는 경로가 `robots.txt`로 막혀 있었다.
막힌 길을 우회하는 대신, **스토어가 열어 둔 길만으로** 전량을 받아 오는 과정을 정리한다.

## 문제 상황

처음 확인한 것은 `robots.txt`였다.

```
User-Agent: *
Disallow: /api/

Sitemap: https://example-store.com/sitemap.xml
```

막힌 건 `/api/` 하나뿐이다. 그래서 상품 목록 페이지를 받아 봤다.

```bash
curl -s https://example-store.com/products/ | grep -c '/products/[0-9a-f]'
# 0
```

**링크가 한 개도 없다.** 목록 페이지가 금지된 `/api/`를 호출해서 그려지기 때문이다.
HTML에는 껍데기만 있고 상품은 브라우저가 나중에 채운다.

## 해결 방법

### 1. 목록은 sitemap에서 얻는다

`robots.txt`가 직접 알려 준 sitemap에 상품 URL이 전량 실려 있었다.

```xml
<urlset>
  <url><loc>https://example-store.com/</loc></url>
  <url><loc>https://example-store.com/products/</loc></url>
  <url><loc>https://example-store.com/products/b75cc1e2…/</loc></url>
  …
</urlset>
```

전체 284개 중 상품이 259개였다. sitemap은 "여기를 긁어가라"고 스토어가 내놓은 목록이므로,
이쪽을 쓰는 것은 우회가 아니다.

```ts
const RE_HANDLE = /^\/products\/([0-9a-f]{32})\/?$/;

export function extractProductUrls(xml: string, origin: string): string[] {
  const host = new URL(origin).host;
  const urls: string[] = [];
  let match: RegExpExecArray | null;

  RE_LOC.lastIndex = 0;
  while ((match = RE_LOC.exec(xml)) !== null) {
    const url = new URL(match[1].trim());
    if (url.host === host && RE_HANDLE.test(url.pathname)) {
      urls.push(`${origin}${url.pathname}`);
    }
  }

  return urls;
}
```

### 2. 상세는 JSON-LD에서 읽는다

상품 페이지에는 검색엔진용 구조화 데이터(schema.org)가 박혀 있었다.

```json
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "…",
  "image": ["https://cdn.shopify.com/…/a.webp?v=1768044441"],
  "url": "https://example-store.com/products/b75cc1e2…",
  "offers": { "@type": "Offer", "price": "440.0", "priceCurrency": "JPY" }
}
```

HTML 구조를 긁는 것보다 이쪽이 낫다. **화면이 개편돼도 깨지지 않는다.**
검색엔진에 보여주려고 만든 데이터라 형식이 안정적이다.

## 두 번 실패하고 알게 된 것

### 실패 1 — 변형 상품은 `Product`가 아니다 (91건)

1차 수집에서 259건 중 168건만 들어왔다. 실패한 페이지를 열어 보니 JSON-LD는 있었다.
`@type`이 달랐다.

```json
{
  "@type": "ProductGroup",
  "productGroupID": "1b4f934c…",
  "variesBy": ["サイズ"],
  "hasVariant": [
    { "@type": "Product", "name": "… M", "offers": { "price": "4400.0" } },
    { "@type": "Product", "name": "… L", "offers": { "price": "4620.0" } }
  ]
}
```

사이즈·색상이 있는 상품은 `ProductGroup`으로 내려오고 **가격이 상품이 아니라 변형마다 붙는다.**

변형을 따로 세지 않고 한 건으로 묶었다. 상품 주소가 하나뿐이라 식별자도 하나이고,
사용자 기록이 그 식별자에 묶이기 때문이다. 정가는 변형 중 최저가를 쓴다.

```ts
function pickListPriceJpy(ld: ProductLd): number | null {
  const direct = toPrice(ld.offers?.price);
  if (direct !== null) {
    return direct;
  }

  const prices = (ld.hasVariant ?? [])
    .map((v) => toPrice(v.offers?.price))
    .filter((p): p is number => p !== null);

  return prices.length > 0 ? Math.min(...prices) : null;
}
```

최상위만 본다는 점이 중요하다. `hasVariant` 안의 `Product`는 변형이지 별개 상품이 아니다.

### 실패 2 — 구조화 데이터가 아예 없는 시리즈 (75건)

2차에서 184건까지 올라갔다. 남은 75건은 특정 시리즈였고, `<script type="application/ld+json">`이
하나도 없었다. 가격을 찾아봤다.

```bash
grep -oE '"price"|円' page.html | wc -l
# 0
```

**가격이 HTML에 한 글자도 없다.** 그 값은 금지된 `/api/`로만 내려온다.

여기서 선택이 갈린다. 금지된 경로를 호출하거나, 가격을 포기하거나.
후자를 택했다. og 메타태그로 이름과 이미지만 건지고 정가는 `null`로 둔다.

```ts
export function extractOpenGraph(html: string): ProductLd | null {
  const og = new Map<string, string>();
  let m: RegExpExecArray | null;

  RE_OG.lastIndex = 0;
  while ((m = RE_OG.exec(html)) !== null) {
    if (!og.has(m[1])) {
      og.set(m[1], m[2]);
    }
  }

  // og:title 은 `상품명 | 스토어명` 형태다
  const title = (og.get('title') ?? '').replace(RE_TITLE_SUFFIX, '').trim();
  if (title.length === 0) {
    return null;
  }

  return { '@type': 'Product', name: title, image: [og.get('image')!], url: og.get('url') };
}
```

구조화 데이터를 먼저 보고, 없으면 og로 넘어간다.

```ts
const ld = extractProductLd(body) ?? extractOpenGraph(body);
```

화면에는 "가격 미정"으로 나온다. 정가를 모르는 굿즈는 지출 분석에서 비교 대상이 빠지지만,
**모르는 값을 지어내는 것보다 낫다.**

3차 수집에서 259/259건, 실패 0건이 됐다.

## 태그가 없으면 이름에서 뽑는다

이 스토어는 분류 태그도, 한국어 상품명도 주지 않는다.
대신 이름이 규칙적이었다.

```
ちいかわぽけっと  ピクニック な マスコット （ちいかわ）
└─ 시리즈 ────┘  └ 테마 ┘      └ 품목 ┘  └ 캐릭터 ┘
```

259건을 분해하니 조각이 **쉰 개 남짓으로 수렴**했다.
사전에 넣고 긴 것부터 바꿔치기하면 캐릭터·종류·한국어 이름을 전부 이름에서 얻을 수 있다.

```ts
/** 긴 조각부터. `ハンドタオル` 가 `タオル` 보다 먼저 걸려야 한다 */
const KEYS_BY_LENGTH = Object.keys(DICTIONARY).sort((a, b) => b.length - a.length);

export function translate(name: string): string {
  let out = name;

  for (const ja of KEYS_BY_LENGTH) {
    if (out.includes(ja)) {
      out = out.split(ja).join(` ${DICTIONARY[ja]} `);
    }
  }

  return out.replace(/\s+/g, ' ').trim();
}
```

### 길이 정렬만으로는 부족했다

`ハンドタオル`(핸드타월)를 처리한 뒤에도 `タオル`(타월)이 또 걸려 두 종류로 잡혔다.
긴 것부터 보는 것만으로는 **이미 맞춘 구간이 남아 있기 때문**이다.
맞춘 자리를 지워 가며 진행해야 한다.

```ts
function matchVocabulary(haystack: string, keywords: string[]): string[] {
  let rest = haystack;
  const codes: string[] = [];

  for (const keyword of keywords) {
    if (!rest.includes(keyword)) {
      continue;
    }
    codes.push(toCode(keyword));
    rest = rest.split(keyword).join(' '); // 맞춘 구간을 지운다
  }

  return codes;
}
```

### 접두사가 캐릭터 이름을 품고 있었다

가장 조용한 함정이었다. 시리즈 접두사 안에 캐릭터명이 들어 있다.

```
ちいかわぽけっと       ← 이 안에 캐릭터명 `ちいかわ` 가 있다
ちいかわクリスマスマーケット  ← 여기에도 있다
```

접두사를 떼지 않고 이름 전체에서 캐릭터를 찾으면 **259건 전부가 같은 캐릭터 굿즈로 분류된다.**
오류가 나지 않고 숫자만 조용히 틀리는 종류라, 테스트로 못박아 두었다.

```ts
it('접두사의 캐릭터명을 캐릭터로 세지 않는다', () => {
  const parts = parseName('ちいかわぽけっと ピクニックなマスコット（ハチワレ）');

  assert.ok(!parts.characterCodes.includes('chiikawa'));
});
```

## 결과

| | |
| --- | --- |
| 수집 | 259/259건, 실패 0건 |
| 종류 분류 | 259건 전부 |
| 한국어 이름 | 259건 전부 (일본어 잔여 0건) |
| 정가 | 184건 (75건은 페이지에 가격 없음) |

어휘를 보강한 뒤에는 재수집 없이 해석만 다시 할 수 있게 해 두었다.

```bash
npx tsx scripts/collect-pocket.ts --reparse   # 요청 0회
```

원본 이름을 레코드에 남겨 두었기 때문이다. 실제로 1차 수집 뒤 종류를 못 찾은 37건을
사전 보강 + `--reparse`로 0건까지 줄였다. 스토어에 추가 요청은 한 번도 보내지 않았다.

## 정리

- `robots.txt`가 막은 경로는 쓰지 않는다. 대신 `sitemap`과 구조화 데이터라는 **열린 길**이 있다
- 구조화 데이터는 HTML 구조보다 안정적이다. 화면 개편에 안 깨진다
- 변형 상품은 `Product`가 아니라 `ProductGroup`으로 올 수 있다. 가격 위치가 다르다
- 구조화 데이터가 없는 페이지는 og 메타태그가 받침이 된다. **모르는 값은 `null`로 두는 편이 낫다**
- 사전 치환은 긴 것부터 하되, **맞춘 구간을 지워야** 짧은 말이 또 걸리지 않는다
- 원본을 레코드에 남겨 두면 분류 규칙이 바뀌어도 재수집이 필요 없다
