---
layout: post
title: "수집은 GitHub Actions, 서빙은 Vercel - 함수 시간 한도가 구조를 가른다"
date: 2026-11-23 21:30:00 +0900
categories: [사이드프로젝트, tobeol-ledger]
tags: [GitHub Actions, Vercel, Supabase, PostgreSQL, PL/pgSQL, 배포]
---

굿즈 데이터를 주기적으로 받아 와야 했다.
처음엔 Vercel Cron으로 끝날 줄 알았는데, **함수 시간 한도** 하나 때문에 구조가 통째로 갈렸다.

## 문제 상황

수집에 걸리는 시간을 재 봤다.

| 대상 | 요청 수 | 소요 |
| --- | --- | --- |
| 스토어 A (상품 JSON 일괄 조회) | 86회 | 2~3분 |
| 스토어 B (상품 페이지 하나씩) | 260회 | 5~6분 |

Vercel 서버리스 함수의 실행 시간 한도는 **Hobby 60초 / Pro 300초**다.
A도 넘고, B는 Pro로도 넘는다.

문제가 하나 더 있었다. Vercel의 파일시스템은 읽기 전용이라
지금처럼 수집 결과를 파일로 떨어뜨릴 수가 없다.

## 해결 방법 — 역할을 셋으로 나눈다

```
GitHub Actions (cron)  →  수집  →  Supabase 적재
                                        ↓
                        Vercel (Nuxt)  읽기 + ISR 캐시
```

Actions는 실행 시간이 6시간까지라 여유롭고, 공개 저장소면 무료다.
수집은 거기서 하고, Vercel은 **읽기만** 한다.

이렇게 나누면 부수 효과가 하나 생긴다.
**스토어에 가는 요청 수가 방문자 수와 무관해진다.** 크론이 하루에 도는 횟수로 고정된다.

### 주기를 대상별로 나눈 이유

```yaml
on:
  schedule:
    - cron: '0 18 * * *'  # 매일 03:00 KST — 스토어 A
    - cron: '0 19 * * 6'  # 일요일 04:00 KST — 스토어 B
```

A는 86회로 가볍고 신상이 자주 뜨니 매일, B는 260회로 무겁고 상품이 드물게 느니 주 1회다.
어느 크론이 깨웠는지는 `github.event.schedule`로 가른다.

```yaml
- name: 대상 결정
  id: plan
  run: |
    if [ "${{ github.event_name }}" = "workflow_dispatch" ]; then
      targets="${{ inputs.targets }}"
    elif [ "${{ github.event.schedule }}" = "0 19 * * 6" ]; then
      targets="b"
    else
      targets="a"
    fi
    echo "targets=$targets" >> "$GITHUB_OUTPUT"
```

적재 단계는 조건을 걸지 않았다. **러너가 매번 새로 뜨기 때문에**
이번에 받지 않은 쪽은 파일 자체가 없고, 적재 스크립트가 알아서 건너뛴다.

```
[b] items-b.jsonl — 수집본이 없습니다. 건너뜁니다.
```

`workflow_dispatch`를 열어 둔 것도 중요했다. 주 1회짜리를 검증하려고
일주일을 기다릴 수는 없다. 실제로 수동 실행으로 양쪽을 먼저 확인했다.

## 적재 — 표별로 나눠 보내지 않는다

굿즈 한 건은 여러 표로 흩어진다.

```
items                 굿즈 본체
├ item_characters     굿즈 ↔ 캐릭터 (다대다)
├ item_categories     굿즈 ↔ 종류   (다대다)
└ item_price_history  정가 이력
```

클라이언트에서 표별로 나눠 보내면 두 가지가 곤란하다.

1. 1만 건에 **수만 번 요청**이 된다
2. 중간에 끊기면 굿즈는 들어갔는데 분류는 안 들어간 **반쪽짜리**가 남는다

그래서 배치를 통째로 Postgres 함수에 넘겼다.

```sql
create or replace function load_items(payload jsonb)
returns jsonb
language plpgsql
as $$
declare
  touched uuid[];
begin
  create temp table _load_in on commit drop as
  select * from jsonb_to_recordset(payload) as p(
    handle text, name_ko text, list_price_jpy integer,
    character_codes jsonb, category_codes jsonb, …
  );

  with up as (
    insert into items (…) select … from _load_in
    on conflict (handle) do update set …
    returning id
  )
  select coalesce(array_agg(id), '{}') into touched from up;
  …
end;
$$;
```

500건씩 23번이면 1만 건이 다 들어간다. **한 배치가 한 트랜잭션**이라 반쪽짜리가 남지 않는다.

### 배열 컬럼은 jsonb로 받아야 한다

`jsonb_to_recordset`은 JSON 배열을 `text[]` 컬럼에 바로 넣지 못한다.
`jsonb`로 받고 함수 안에서 변환한다.

```sql
coalesce(array(select jsonb_array_elements_text(i.tags_raw)), '{}')
```

### CTE로 묶으면 행이 사라진다

분류를 교체할 때 처음엔 CTE 하나로 쓰려 했다.

```sql
-- 이렇게 하면 안 된다
with del as (
  delete from item_characters where item_id = any(touched)
),
ins as (
  insert into item_characters (item_id, character_code)
  select … on conflict do nothing
)
select 1;
```

같은 문 안의 data-modifying CTE들은 **같은 스냅샷을 본다.**
`delete`가 지운 행을 `insert`가 보지 못하고, 아직 있는 것으로 판단해 `on conflict do nothing`으로
건너뛴다. 결과적으로 **delete만 반영되고 행이 사라진다.**

PL/pgSQL 문으로 나누면 앞 문의 결과를 뒤 문이 본다.

```sql
delete from item_characters where item_id = any(touched);

insert into item_characters (item_id, character_code)
select it.id, c.code
from _load_in i
join items it on it.handle = i.handle
cross join lateral jsonb_array_elements_text(i.character_codes) as c(code)
where exists (select 1 from characters ch where ch.code = c.code)
on conflict do nothing;
```

### 어휘가 앞서 나가도 멈추지 않게

수집 쪽 분류 어휘가 DB의 분류 표보다 먼저 늘어나는 일이 생긴다.
그대로 두면 외래키 위반으로 적재가 통째로 멈춘다.

없는 코드는 **넣지 않고 이름만 돌려주도록** 했다.

```sql
where exists (select 1 from categories ca where ca.code = c.code)
```

```ts
if (unknownCats.size > 0) {
  console.log(`DB 에 없는 종류 코드: ${[...unknownCats].join(', ')}`);
  console.log('  categories 표에 추가해야 해당 분류가 붙습니다.');
}
```

적재는 끝까지 돌고, 무엇을 더해야 하는지는 로그로 남는다.

### 이력은 바뀐 날만 남긴다

정가 이력을 매번 기록하면 하루 1만 행, **1년에 400만 행**이 된다.
직전 기록과 값이 다를 때만 넣는다.

```sql
with latest as (
  select distinct on (item_id) item_id, list_price_jpy
  from item_price_history
  where item_id = any(touched)
  order by item_id, observed_on desc
)
insert into item_price_history (item_id, list_price_jpy)
select it.id, it.list_price_jpy
from items it
left join latest l on l.item_id = it.id
where it.id = any(touched)
  and it.list_price_jpy is not null
  and (l.item_id is null or l.list_price_jpy <> it.list_price_jpy)
on conflict (item_id, observed_on) do nothing;
```

실제로 첫 자동 수집에서 10,616건을 받았는데 이력은 **31행만** 늘었다.
그 사이 새로 올라온 굿즈의 수와 정확히 같다.

## 키를 어디에 두는가

적재는 RLS를 우회해야 해서 `service_role` 키가 필요하다.
이 키를 **Vercel에는 넣지 않았다.**

| | 로컬 | Vercel | GitHub Secrets |
| --- | :-: | :-: | :-: |
| URL | ✓ | ✓ | ✓ |
| publishable 키 | ✓ | ✓ | |
| **service_role 키** | ✓ | **✗** | ✓ |

앱은 읽기만 하므로 publishable 키로 충분하다.
Vercel에 두면 서버 코드가 실수로 집어 쓸 길이 생긴다. **적재하는 쪽에만** 둔다.

Vercel의 Supabase 연동(Optional Integrations)을 쓰지 않은 것도 이 때문이다.
그 연동은 `service_role` 키까지 자동으로 주입한다.

## 결과

수동 실행으로 양쪽을 먼저 검증했다.

```
스토어 A   4분 11초   10,616건   정가 이력 31행
스토어 B   9분        259건      실패 0건
```

우려했던 건 러너의 curl이 레이트리밋에 걸리는 것이었다.
로컬에서 Windows 기본 curl은 429를 맞고 Git 번들 curl은 통과한 전력이 있어서,
Ubuntu 러너가 어느 쪽일지 몰랐다. 통과하는 쪽이었다.
요청 간격을 로컬(1초)보다 넉넉한 2초로 둔 것도 거들었을 것이다.

## 정리

- 서버리스 함수의 시간 한도는 **아키텍처를 가르는 제약**이다. 수 분짜리 작업은 다른 곳에서 돌려야 한다
- 수집과 서빙을 나누면 외부 요청 수가 **방문자 수와 무관**해진다
- 러너가 매번 새로 뜨는 성질을 이용하면 조건 분기를 덜 쓸 수 있다
- 여러 표로 흩어지는 적재는 **배치를 통째로 DB 함수에 넘기는 편**이 요청 수와 원자성 양쪽에 낫다
- 같은 문의 data-modifying CTE는 같은 스냅샷을 본다. `delete` 후 `insert`는 **문을 나눠야** 한다
- RLS를 우회하는 키는 그것이 꼭 필요한 곳에만 둔다
