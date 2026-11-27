---
layout: post
title: "프로젝트 전체 eslint --fix 정리 - indent 규칙 충돌과 blame 노이즈"
date: 2026-11-27 14:00:00 +0900
categories: [개발작업]
tags: [ESLint, Nuxt, Vue, eslint-stylistic, git, 코드스타일]
---

Nuxt 프로젝트에서 ESLint 규칙과 실제 코드 스타일이 크게 어긋나 있어서 전체 파일에 `eslint --fix`를 한 번에 돌렸다.
100여 개 파일, 수천 줄이 바뀌는 커밋이었는데, 그 과정에서 생긴 규칙 충돌과 일괄 정리 커밋을 다룰 때 챙길 점을 정리한다.

## 오늘 한 일

### 1. 규칙과 코드 불일치 정리 (완료)

- 설정에는 작은따옴표, 세미콜론 필수, 2칸 들여쓰기, 여러 줄 끝 쉼표가 정해져 있었다.
- 실제 코드는 큰따옴표가 섞여 있었다. 심지어 설정 파일 자신도 큰따옴표였다.
- 그래서 lint 오류가 수천 개 쌓여서, 새로 생긴 오류를 알아볼 수 없었다.

### 2. `.vue`의 `<script>` 들여쓰기 규칙 충돌 해소 (완료)

`.vue` 파일에서 `--fix`가 끝나지 않고, 고친 곳이 다시 오류가 되는 현상이 있었다.

**원인:** `.vue` 파일의 `<script>`에 두 규칙이 동시에 적용되고 있었다.

- `@stylistic/indent`: JS/TS 일반 들여쓰기 규칙
- `vue/script-indent`: SFC `<script>` 전용 들여쓰기 규칙

두 규칙은 일부 구문(여러 줄 삼항, 체이닝, 객체 인자 등)에서 들여쓰기 계산이 달라서, 한쪽에 맞추면 다른 쪽이 오류를 낸다.

**해결:** `.vue` 파일에서만 `@stylistic/indent`를 끈다.

```js
// eslint.config.mjs
export default withNuxt({
  rules: {
    '@stylistic/indent': ['error', 2],
    'vue/script-indent': ['error', 2, { baseIndent: 0 }],
    'vue/html-indent': ['error', 2],
    // ...
  },
}, {
  // .vue의 <script>는 vue/script-indent 하나로만 검사
  files: ['**/*.vue'],
  rules: {
    '@stylistic/indent': 'off',
  },
});
```

flat config에서는 뒤에 오는 설정 객체가 앞의 규칙을 덮어쓰므로, `files`로 범위를 좁힌 객체를 하나 더 붙이면 된다.

### 3. "기능 변경 없음" 검증 (완료)

형식만 바꾸는 커밋이라도 자동 수정이 의미를 바꾸는 경우가 있어서 diff에서 몇 가지를 확인했다.

| 확인 항목 | 결과 |
| --- | --- |
| `vue/attribute-hyphenation`: `v-model:expandedKeys` → `v-model:expanded-keys` | Vue 3 컴파일러가 인자를 camelize해서 `onUpdate:expandedKeys`로 연결하므로 동작 동일 |
| SVG 속성(`viewBox` 등)이 `view-box`로 바뀌었는지 | 네이티브 요소는 변환 대상이 아니라 변경 없음 |
| `==` / `!=` 연산자 | 그대로 (따옴표만 변경) |
| 타입 오류 개수 | 수정 전후 동일 |
| 프로덕션 빌드 | 성공 |

`git show -w --stat`으로 공백 변경을 빼고 봐도 대부분 따옴표·세미콜론·쉼표 변경이었다.

## 이슈 / 특이사항

### 일괄 정리 커밋은 꼭 따로 분리한다

기능 변경과 섞으면 리뷰할 때 "따옴표만 바뀐 줄" 사이에서 진짜 변경을 찾아야 한다.
형식 정리만 단독 커밋으로 두면 이후 기능 커밋의 diff가 깨끗해진다.

### `git blame` 노이즈는 ignore-revs로 처리한다

거의 모든 줄의 마지막 수정자가 이 커밋이 되므로 blame이 쓸모없어진다.
저장소 루트에 커밋 해시를 적은 파일을 두면 된다.

```text
# .git-blame-ignore-revs
# eslint --fix 일괄 정리
<정리 커밋 해시>
```

```bash
git config blame.ignoreRevsFile .git-blame-ignore-revs
```

GitHub 웹의 blame 화면은 이 파일을 자동으로 인식한다.

### 오래된 브랜치와의 충돌

정리 커밋 이전에 갈라진 브랜치를 머지하면 대량 충돌이 난다.
머지 전에 그 브랜치에서도 같은 `eslint --fix`를 먼저 돌리면 충돌이 크게 줄어든다.
