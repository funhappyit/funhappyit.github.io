---
layout: post
title: "집계 숫자 클릭 시 대상자 목록을 보여주는 서브그리드 추가"
date: 2026-07-15 18:00:00 +0900
categories: [개발작업, 일일로그]
tags: [Vue, DevExtreme, SpringBoot, QueryDSL]
---

## 오늘 한 일

### 집계 화면에 "숫자 클릭 → 대상자 목록" 드릴다운 기능 추가

- 월별 인원수 집계(예: 이번 달 퇴사자 수, 근속기간 구간별 퇴사자 수 등)만 보여주던 관리자 화면에서, 숫자를 클릭하면 그 숫자를 구성하는 실제 대상자 목록을 바로 확인할 수 있도록 개선했다.
- 처음부터 새로 설계하지 않고, 이미 다른 화면(개인별 휴가/결근 집계)에서 검증된 "숫자 링크 클릭 → 마스터-디테일 서브그리드 펼침" 패턴을 그대로 재사용했다. 화면마다 UI 동작이 제각각이면 사용자 입장에서 학습 비용이 늘어나기 때문에, 이미 있는 패턴을 재사용하는 쪽을 우선했다.

**백엔드**

- 기존 집계 로직은 대상자 목록을 메모리에 올려두고 스트림 필터링으로 "개수"만 세는 구조였다. 이 필터 조건들을 `Predicate` 반환 팩토리 메서드로 추출해서, 집계(개수)와 상세조회(목록)가 **완전히 동일한 조건**을 공유하도록 리팩터링했다.

  ```java
  // Before: 조건이 count 메서드 안에 그대로 박혀 있음
  private long countByCondition(List<Target> targets, Period period) {
      return targets.stream()
          .filter(t -> t.getEventDt() != null && YearMonth.from(t.getEventDt()).equals(period))
          .count();
  }

  // After: 조건을 Predicate로 분리해서 count와 목록 조회가 동일 조건을 공유
  private Predicate<Target> isInPeriod(Period period) {
      return t -> t.getEventDt() != null && YearMonth.from(t.getEventDt()).equals(period);
  }

  private long countByCondition(List<Target> targets, Period period) {
      return targets.stream().filter(isInPeriod(period)).count();
  }

  private List<Target> listByCondition(List<Target> targets, Period period) {
      return targets.stream().filter(isInPeriod(period)).collect(Collectors.toList());
  }
  ```

  - 이렇게 해두면 나중에 집계 조건이 바뀌어도 상세 목록이 자동으로 같이 바뀐다. 조건을 두 군데(count용, 목록용)에 따로 유지하다가 한쪽만 고쳐서 "숫자는 5인데 목록은 4건" 같은 불일치가 생기는 걸 구조적으로 막을 수 있다.
  - 구간별 카테고리(예: "6개월 미만", "1년 이상" 등)에 따라 다른 조건을 적용해 상세 목록을 반환하는 조회 API를 신규로 추가했다.

**프론트엔드**

- 클릭 대상이 되는 인원수 컬럼(9개)에 링크 스타일 셀 템플릿을 적용했다.
  - 같은 컬럼을 다시 클릭하면 접히고(토글), 펼쳐진 상태에서 다른 컬럼을 클릭하면 행을 다시 펼치지 않고 하위 그리드에 전달하는 "카테고리" prop만 갱신되도록 구성했다. reactive props를 그대로 하위 컴포넌트에 물려주는 구조라, prop 변경만으로 하위 그리드가 재조회하고 재마운트는 일어나지 않는다.
- 신규 서브그리드 컴포넌트는 기존에 만들어둔 "표시된 행 수에 맞춰 그리드 높이를 동적으로 계산하는" 로직을 그대로 재사용했다. 마스터 그리드가 서브그리드의 높이 변화를 자동으로 감지하지 못하는 라이브러리 특성 때문에, 서브그리드 쪽에서 높이를 다시 잰 뒤 부모 그리드 인스턴스에 재계산을 직접 요청해주는 방식이다.

**라우팅 설정**

- 이 프로젝트는 프론트엔드 API 호출 시 액션명을 넘기면, 액션명과 실제 URL/HTTP 메서드를 매핑해둔 별도의 라우팅 테이블을 조회해서 요청을 보내는 구조로 되어 있다. 백엔드에 새 API를 추가한 것과 별개로, 이 라우팅 테이블에 신규 액션을 등록하는 인서트 스크립트도 함께 작성했다.

## 이슈 / 특이사항

- 액션-라우팅 매핑이 프론트엔드 코드에 하드코딩되어 있지 않고 별도 테이블에서 관리되는 구조라는 걸 다시 확인했다. 신규 API를 추가할 때 컨트롤러 매핑만 넣고 "다 됐다"고 착각하기 쉬운데, 이 프로젝트에서는 라우팅 테이블 등록까지 해줘야 실제로 호출이 붙는다.
- 집계값과 상세 리스트 조회 API를 처음부터 별도 로직으로 각각 구현했다면, 두 로직이 시간이 지나면서 조금씩 어긋나기 쉬웠을 것 같다. "카운트도, 목록도 같은 필터 함수 하나를 통과시킨다"는 원칙을 초반에 세워두는 게 이후 유지보수 비용을 줄이는 데 도움이 됐다.
