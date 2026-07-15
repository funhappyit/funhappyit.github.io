---
layout: post
title: "DevExtreme 그리드 엑셀 다운로드에 숨겨진 상세 컬럼 추가하기"
date: 2026-07-15 19:30:00 +0900
categories: [개발작업, 일일로그]
tags: [Vue, DevExtreme, QueryDSL, Spring Boot]
---

## 오늘 한 일

### 상담원 목록 화면 - 엑셀 다운로드에 상세정보 카드 항목 추가

목록 그리드에는 이미 우측 상단에 엑셀 다운로드 버튼이 있었는데, 개별 상담원의 "상세정보 카드" 팝업에만 있던 항목(생년월일, 양력/음력, 전화번호, 휴대폰번호, 이메일, 비상연락처, 비상연락처와의 관계, 우편번호, 주소, 상세주소)도 이 엑셀 다운로드에 포함되게 해달라는 요청을 받았다.

별도의 다운로드 버튼이나 팝업을 새로 만드는 대신, 기존 그리드의 엑셀 export 기능에 컬럼만 추가하는 방식으로 진행했다.

**백엔드**

목록 조회 API가 사용하는 QueryDSL 쿼리를 확인해보니, 이미 상세정보 테이블을 `LEFT JOIN`하고 있었다. 다만 그 프로젝션(DTO 생성자)에는 일부 필드(비상연락처 등)만 담겨 있고 생년월일·주소 계열 필드는 빠져 있는 상태였다. 새 JOIN이나 매퍼 작업 없이, DTO 필드와 QueryDSL 프로젝션에 누락된 컬럼만 추가하면 되는 상황이었다.

```java
private JPAQuery<MemberDto.SearchResult> getMemberQuery() {
    return queryFactory
        .select(Projections.constructor(MemberDto.SearchResult.class,
            member.id,
            member.name,
            // ...기존 필드 생략...
            memberDetail.emergencyPhone,
            memberDetail.emergencyContactRelation,
            // 여기부터 이번에 추가한 필드
            memberDetail.birthDt,
            memberDetail.lunarCd,
            memberDetail.telNo,
            memberDetail.phoneNo,
            memberDetail.email,
            memberDetail.postNo,
            memberDetail.address,
            memberDetail.addressDetail
        ))
        .from(member)
        .leftJoin(memberDetail)
        .on(member.id.eq(memberDetail.masterId));
}
```

같은 DTO를 사용하는 생성자가 두 개(기본형 / 시험정보 포함형)라서, 양쪽 시그니처와 호출부를 모두 맞춰줘야 하는 게 조금 번거로웠다.

**프론트엔드**

그리드 컬럼 정의에 새 컬럼 10개를 `visible: false`로 추가했다. 사용 중인 그리드 컴포넌트는 엑셀 export 시 "`allowExporting`이 명시적으로 `false`가 아닌 컬럼은 화면 표시 여부와 무관하게 전부 포함시키는" 방식으로 이미 동작하고 있었다. 덕분에 별도의 export 커스터마이징 없이 `visible: false` 컬럼만 추가하면 자동으로 엑셀에 포함됐다. 실제로 같은 패턴(화면엔 숨기고 엑셀에만 노출)을 쓰는 기존 컬럼이 있어서 그대로 참고했다.

```js
{
  caption: '생년월일',
  dataField: 'birthDt',
  visible: false, // 화면에는 숨기고 엑셀 다운로드에만 포함
  calculateCellValue: rowData => {
    const v = rowData?.birthDt;
    return v && v.length === 8 ? `${v.slice(0, 4)}-${v.slice(4, 6)}-${v.slice(6, 8)}` : (v ?? '');
  },
},
```

## 이슈 / 특이사항

### 버튼 컬럼이 엑셀에 원본 값 그대로 찍히는 문제

컬럼을 추가하고 나서 실제 다운로드 결과를 확인해보니, 그리드에서 "상세정보 카드" 팝업을 여는 아이콘 버튼 컬럼이 엉뚱하게 숫자로 찍혀 있었다.

원인은 이 컬럼이 화면에서는 `cellTemplate`으로 아이콘 버튼만 렌더링하고 있었는데, 엑셀 export는 `cellTemplate`을 거치지 않고 해당 컬럼의 `dataField` 원본 값을 그대로 내보내기 때문이었다. 그 결과 다운로드한 엑셀에는 버튼 대신 내부 식별자(숫자 id) 값이 그대로 노출되고 있었다.

`cellTemplate`을 쓰는 버튼/아이콘 컬럼은 화면 렌더링과 export 결과가 서로 다른 경로를 탄다는 걸 다시 확인한 셈이다. 해당 컬럼에 `allowExporting: false`를 추가해서 화면 버튼 동작은 그대로 두고 엑셀 다운로드 대상에서만 제외하는 것으로 해결했다.

```js
{
  caption: '상세정보 카드',
  dataField: 'detailId',
  allowExporting: false, // 버튼 컬럼 - 엑셀에는 원본 id 값만 찍히므로 제외
  cellTemplate: (container, options) => {
    // 아이콘 버튼 렌더링, 클릭 시 상세 팝업 오픈
  },
},
```

정리하면, DevExtreme 그리드에서 엑셀 다운로드에 화면에 없는 컬럼을 끼워 넣고 싶을 땐 `visible: false`만 추가하면 되고, 반대로 화면에는 있지만 엑셀에서는 빼고 싶은 컬럼(특히 버튼/아이콘 `cellTemplate` 컬럼)은 `allowExporting: false`를 명시해야 한다는 걸 한 번에 확인한 작업이었다.
