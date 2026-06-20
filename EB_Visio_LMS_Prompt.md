# EB Visio LMS — GPT Prompt Engineering 가이드

> 이 파일은 `EB_Visio_LMS_v3.html` 파일을 GPT(ChatGPT / Claude 등)에 업로드하거나  
> 코드를 붙여넣고 기능 수정·추가를 요청할 때 사용하는 **프롬프트 템플릿 모음**입니다.

---

## 🔰 시스템 기본 컨텍스트 (모든 요청 전 반드시 포함)

```
아래는 EB Visio LMS(License Management System) ver 1.0 시스템입니다.

[시스템 특성]
- 단일 HTML 파일 (HTML + CSS + JS 완전 내장)
- 프레임워크 없음 (Vanilla JS)
- 데이터 저장: IndexedDB (영구) + localStorage (백업)
- 외부 라이브러리: SheetJS (Excel 처리, CDN 동적 로드)
- DB 테이블: users / pools / licenses / requests / audit_logs
- 라이선스 유형: Process / Detail 두 가지

[주요 전역 변수 및 함수]
- DB: 전체 데이터 객체
- CU: 현재 로그인한 사용자 객체
- gU(id): user 조회, gP(id): pool 조회, gL(id): license 조회
- saveDB(): IndexedDB + localStorage 저장
- addLog(action, table, rid, desc): 감사 로그 기록
- toast(msg, type): 알림 메시지 표시 ('s'=성공, 'e'=에러, 'w'=경고)
- nav(page): 페이지 이동
- sw(ns, id, btn): 탭 전환
- esc(s): HTML 이스케이프

[페이지 ID]
dashboard / apply / revoke / status / approval / admin

[관리자 설정 탭 ID]
users / perms / lic / pool / history / sys

[Pool 관리 서브탭 ID]
pl(목록) / pa(단건 추가) / pu(Excel 업로드)

이 파일의 HTML/CSS/JS를 수정할 때는:
1. 기존 변수명·함수명·ID를 그대로 유지할 것
2. 전체 코드를 다시 출력하지 말고, 수정할 부분만 "변경 전/변경 후" 형식으로 제시할 것
3. 수정 후 영향받는 함수(render 계열)도 함께 확인할 것
```

---

## 📌 요청 유형별 프롬프트 템플릿

---

### 1. 기능 추가 요청

```
[파일] EB_Visio_LMS_v3.html (첨부)

[요청] {기능 설명을 구체적으로 작성}

[위치] {화면 경로 예: 관리자설정 > Pool 관리 > Pool 추가}

[조건]
- 기존 코드 스타일(Vanilla JS, CSS 변수 사용) 유지
- 새 함수명은 camelCase로 작성
- 변경된 데이터는 saveDB() 호출 후 관련 render 함수 모두 재호출
- 에러 처리: err-box 클래스 div로 표시

[출력 형식]
변경할 HTML 부분과 JS 함수를 구분하여 제시해줘.
```

**예시:**
```
[요청] Pool 목록 화면에 "그룹별 필터" 드롭다운을 추가해줘.
가용 그룹 목록은 DB.users에서 동적으로 추출하고,
선택한 그룹에 배정된 Pool만 표시되도록 renderPool() 함수를 수정해줘.
```

---

### 2. 버그 수정 요청

```
[파일] EB_Visio_LMS_v3.html (첨부)

[문제 상황]
- 발생 화면: {화면 경로}
- 수행한 동작: {어떤 버튼 클릭 또는 입력}
- 기대 결과: {어떻게 되어야 하는지}
- 실제 결과: {실제로 어떻게 되는지}

[관련 함수 추정] {알고 있다면 함수명 기재, 모르면 생략}

수정 시 다른 기능에 영향이 없도록 최소한의 코드만 변경해줘.
```

---

### 3. UI/디자인 수정 요청

```
[파일] EB_Visio_LMS_v3.html (첨부)

[수정 위치] {화면 경로 및 요소명}

[현재 상태] {현재 어떻게 보이는지}

[원하는 상태] {어떻게 바꾸고 싶은지}

[제약]
- CSS 변수(--blue, --gray1 등) 를 활용할 것
- 기존 클래스(btn, fc, fg, badge 등)를 최대한 재사용할 것
- 인라인 스타일과 별도 클래스 추가를 구분하여 제시할 것
```

---

### 4. Excel 입출력 관련 수정

```
[파일] EB_Visio_LMS_v3.html (첨부)

[요청] {Excel 업로드 또는 다운로드 관련 수정 내용}

[관련 함수]
- 업로드: handlePoolUpload() / handleAdminLicUpload() / parseExcelFile()
- 다운로드: downloadStatusExcel() / downloadPoolExcel()
- 컬럼 정규화: normalizeCol()

[Excel 컬럼 구조]
업로드 필수: 시리얼번호, 라이선스유형, 등록일자
업로드 선택: 사용자, 사번, 그룹, Knox계정, Remark

수정 후 SheetJS 동적 로드(loadSheetJS) 방식은 유지해줘.
```

---

### 5. 데이터 구조 변경 요청

```
[파일] EB_Visio_LMS_v3.html (첨부)

[요청] DB 테이블 또는 필드 추가/변경

[변경 내용]
- 테이블명: {users / pools / licenses / requests / audit_logs}
- 추가/변경 필드: {필드명, 타입, 초기값}

[주의사항]
1. INIT 객체에 새 필드 추가
2. loadDB() 후 기존 DB에 필드 누락 시 기본값 보장 코드 추가
   예: if(DB.xxx===undefined) DB.xxx = 기본값;
3. 관련 render 함수에서 새 필드 표시 반영
4. Excel 업로드/다운로드 컬럼 매핑 업데이트 여부 확인
```

---

### 6. 승인 워크플로우 수정

```
[파일] EB_Visio_LMS_v3.html (첨부)

[요청] {승인/반려 프로세스 관련 수정}

[관련 함수]
- submitApply(): 라이선스 신청
- submitRevoke(): 라이선스 해지 신청
- doApprove(reqId): 승인 처리
- doReject(reqId): 반려 처리
- renderPending(): 대기 목록 렌더링
- renderApprHist(): 전체 이력 렌더링

[request 상태값]
pending / approved / rejected / cancelled
```

---

### 7. 반복 작업 지시 (단계별)

복잡한 작업을 단계별로 나눠서 요청할 때 사용:

```
[파일] EB_Visio_LMS_v3.html (첨부)

지금부터 아래 작업을 단계별로 진행해줘.
각 단계가 끝나면 "다음 단계로 진행할까요?" 라고 물어봐줘.

[1단계] {첫 번째 작업}
[2단계] {두 번째 작업}
[3단계] {세 번째 작업}

각 단계에서 변경되는 코드만 "변경 전 / 변경 후" 형식으로 출력해줘.
전체 파일을 다시 출력하지 말 것.
```

---

## 🎯 자주 쓰는 단발성 프롬프트

### 현재 파일 구조 파악 요청
```
첨부한 EB_Visio_LMS_v3.html 파일의 전체 기능 목록과
각 기능이 어떤 함수로 구현되어 있는지 표 형태로 정리해줘.
```

### 특정 화면 동작 확인
```
첨부 파일에서 "관리자설정 > {탭명}" 화면의
HTML 구조와 관련 JS 함수 목록을 설명해줘.
```

### 코드 리뷰 요청
```
첨부 파일의 {함수명} 함수를 검토하고
버그 가능성이나 개선점이 있으면 알려줘.
수정이 필요하면 변경 전/후 코드를 제시해줘.
```

### 유사 기능 참고 요청
```
첨부 파일에서 {기능 A}가 구현된 방식을 참고해서
{기능 B}도 동일한 패턴으로 추가해줘.
```

---

## ⚠️ GPT에게 주의사항으로 전달할 내용

```
[코딩 규칙 — 반드시 지켜줘]

1. 전체 파일 재출력 금지 — 변경 부분만 "변경 전 / 변경 후" 형식으로
2. 기존 함수명·변수명·HTML ID 변경 금지
3. CSS 변수 사용: --blue, --red, --green, --gray1~6, --blue-l, --red-l, --green-l
4. 공통 CSS 클래스 재사용: btn, btn-p, btn-d, btn-g, btn-sm, fc, fg, fl, req, badge, bav, bass, err-box, info-box, tw, card, g2, g3
5. 데이터 변경 후 반드시 saveDB() 호출
6. 감사 로그: addLog(action, table, rid, desc) 항상 기록
7. 사용자 피드백: toast() 함수로 처리
8. 시스템 관리자(employee_id='0000') 예외 처리 유지
9. IndexedDB 비동기 처리 패턴 유지 (openIDB, idbGet, idbSet)
10. SheetJS는 loadSheetJS(cb) 동적 로드 방식 유지
```

---

## 📂 파일 업로드 팁

| 상황 | 방법 |
|---|---|
| ChatGPT-4o | 파일 첨부 (📎 버튼) 후 프롬프트 입력 |
| Claude | 파일 첨부 후 프롬프트 입력 |
| 코드가 너무 길 경우 | 관련 함수 부분만 복사 + "전체 파일에서 이 함수를 수정해줘" |
| 맥락 유지 필요 시 | 대화 첫 메시지에 "기본 컨텍스트" 섹션을 붙여넣기 |

---

> 작성일: 2026-06-20  
> 대상 파일: `EB_Visio_LMS_v3.html`  
> 작성 목적: GPT 기반 코드 수정 효율화 및 일관성 유지
