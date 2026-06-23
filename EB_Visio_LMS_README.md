# 🔷 EB Visio LMS

> **EB Visio License Management System** — Microsoft Visio 라이선스 신청 · 승인 · 배정 · 회수 · 현황 관리를 단일 HTML 파일로 구현한 웹 애플리케이션

| 항목 | 내용 |
|------|------|
| **버전** | v1.1 |
| **파일** | `EB_Visio_LMS_v1.html` (단일 파일) |
| **기술 스택** | Vanilla JavaScript · HTML5 · CSS3 · Chart.js · SheetJS · JSONBin.io |
| **데이터 저장** | JSONBin.io REST API (공유) + localStorage (캐시) |
| **배포 방식** | GitHub Pages / 로컬 파일 직접 실행 |
| **작성** | 조선 EDH 그룹, 홍광섭 프로 |

---

## 📋 목차

1. [시스템 개요](#1-시스템-개요)
2. [주요 기능](#2-주요-기능)
3. [데이터 구조](#3-데이터-구조)
4. [화면 구성](#4-화면-구성)
5. [동기화 아키텍처](#5-동기화-아키텍처--jsonbinio)
6. [함수 목록](#6-함수-목록)
7. [로그인 및 권한](#7-로그인-및-권한)
8. [업무 흐름](#8-업무-흐름)
9. [설치 및 실행](#9-설치-및-실행)
10. [환경 설정값](#10-환경-설정값)

---

## 1. 시스템 개요

### 배경

기존 수작업 Excel 관리 방식의 문제점을 해결하기 위해 개발된 웹 기반 라이선스 관리 시스템입니다.

| AS-IS (기존) | TO-BE (개선) |
|-------------|-------------|
| 수작업 Excel → 버전 불일치 | 단일 시스템 실시간 통합 |
| 중복 배정 / 미회수 발생 | 자동 배정 → 중복·누락 차단 |
| 이메일 신청·구두 승인 | 온라인 신청·승인 워크플로우 |
| 가용 수량 파악 불가 | Dashboard KPI 즉시 확인 |
| 감사 이력 없음 | Audit Log 완전 보존 |
| 데이터 PC마다 파편화 | JSONBin.io 멀티유저 실시간 공유 |

### 핵심 특징

- **Zero 설치** — 브라우저에서 HTML 파일 하나로 즉시 실행
- **로그인 없이 신청** — 게스트 신청 팝업으로 누구나 즉시 접근
- **실시간 멀티유저 동기화** — JSONBin.io API로 여러 PC 동시 접속 데이터 공유
- **완전한 감사 추적** — 모든 액션이 Audit Log에 자동 기록

---

## 2. 주요 기능

### 2.1 일반 사용자 기능

| 기능 | 설명 |
|------|------|
| **라이선스 신청** | 이름·사번·소속 그룹·Knox 계정·라이선스 유형 입력 후 신청 |
| **라이선스 해지** | 보유 중인 라이선스 해지 요청 |
| **현황 조회** | 전체 배정 현황 조회 / Excel 다운로드 |
| **비로그인 신청** | 로그인 화면에서 '📝 라이선스 등록 신청' 버튼으로 게스트 신청 |

### 2.2 관리자 전용 기능

| 기능 | 설명 |
|------|------|
| **승인 관리** | 신청·해지 요청 승인 / 반려 / 코멘트 처리 |
| **사용자 관리** | 사용자 추가·삭제·정보 수정 |
| **관리자 권한** | 관리자 권한 부여 / 회수 |
| **라이선스 등록/삭제** | Pool 시리얼번호 등록·소프트삭제·하드삭제 |
| **Pool 관리** | Pool 단건 추가·Excel 업로드·현황 조회 |
| **변경 이력** | 전체 Audit Log 조회 및 초기화 |
| **시스템 정보** | JSONBin 연결 상태·DB 통계·데이터 초기화 |
| **로그인 공지** | 로그인 화면 공지 문구 편집 |

### 2.3 라이선스 유형

| 유형 | 설명 |
|------|------|
| **Process** | 의장 Diagram 작성용 |
| **Detail** | 전장 Diagram 작성용 |

---

## 3. 데이터 구조

모든 데이터는 `DB` 객체에 저장되며 JSONBin.io와 localStorage에 동기화됩니다.

### 3.1 전체 스키마

```javascript
DB = {
  users:      User[],
  pools:      Pool[],
  licenses:   License[],
  requests:   Request[],
  audit_logs: AuditLog[],
  nextId:     { user, pool, license, request, log },
  login_notice: string
}
```

### 3.2 User (사용자)

```javascript
{
  id:           number,       // 자동 증가 PK
  employee_id:  string,       // 사번 (로그인 키)
  name:         string,       // 이름 (로그인 키)
  group:        string,       // 소속 그룹
  knox_account: string,       // Knox 계정 (이메일)
  is_admin:     boolean,      // 관리자 여부
  is_active:    boolean,      // 활성 여부
  created_at:   string        // 등록일 (YYYY-MM-DD)
}
```

> **시스템 관리자**: `employee_id: '0000'`, `name: 'Admin'` — 항상 `ensureSystemAdmin()`으로 보장

### 3.3 Pool (라이선스 Pool)

```javascript
{
  id:                  number,
  serial_number:       string,   // 시리얼번호 (e.g. 'VISIO-2024-0001')
  license_type:        string,   // 'Process' | 'Detail'
  status:              string,   // 'active' | 'inactive'
  is_assigned:         boolean,  // 배정 여부
  is_license_deleted:  boolean,  // 소프트삭제 여부
  assigned_user_name:  string,   // 배정된 사용자명
  assigned_employee_id:string,   // 배정된 사번
  assigned_group:      string,   // 배정된 그룹
  assigned_knox:       string,   // 배정된 Knox 계정
  remark:              string,   // 비고
  registered_date:     string,   // 등록일
  registered_by:       string    // 등록자
}
```

### 3.4 License (배정 레코드)

```javascript
{
  id:            number,
  user_id:       number,          // User.id 참조
  pool_id:       number,          // Pool.id 참조
  status:        string,          // 'active' | 'revoked'
  assigned_date: string,          // 배정일
  revoked_date:  string | null,   // 회수일
  remark:        string,
  assigned_by:   string           // 배정 처리자
}
```

### 3.5 Request (신청/해지 요청)

```javascript
{
  id:            number,
  requester_id:  number,    // User.id 참조
  request_type:  string,    // 'apply' | 'revoke'
  pool_id:       number | null,
  license_id:    number | null,
  status:        string,    // 'pending' | 'approved' | 'rejected'
  reason:        string,    // 신청 사유
  admin_comment: string,    // 관리자 코멘트
  approved_by:   string | null,
  approved_at:   string | null,
  created_at:    string     // 'YYYY-MM-DD HH:mm' 형식
}
```

### 3.6 AuditLog (감사 로그)

```javascript
{
  id:          number,
  action:      string,   // 'CREATE'|'UPDATE'|'DELETE'|'APPROVE'|'REJECT'|'REVOKE'|'UPLOAD'
  table_name:  string,   // 대상 테이블명
  record_id:   number | null,
  user_id:     number,
  username:    string,
  description: string,   // 상세 설명
  created_at:  string    // 한국 로컬타임 문자열
}
```

---

## 4. 화면 구성

### 4.1 페이지 목록

| Page ID | 메뉴명 | 접근 권한 |
|---------|--------|-----------|
| `page-dashboard` | 📊 Dashboard | 전체 |
| `page-apply` | 📝 라이선스 신청 | 전체 |
| `page-revoke` | ❌ 라이선스 해지 | 전체 |
| `page-status` | 🔍 현황 조회 | 전체 |
| `page-approval` | ✅ 승인 관리 | **관리자** |
| `page-admin` | ⚙️ 관리자 설정 | **관리자** |
| `page-search` | 🔎 검색 결과 | 전체 (내부) |

### 4.2 관리자 설정 탭

| Tab ID | 탭명 |
|--------|------|
| `admin-tab-users` | 👥 사용자 관리 |
| `admin-tab-perms` | 🔑 관리자 권한 |
| `admin-tab-lic` | 💿 라이선스 등록/삭제 |
| `admin-tab-pool` | 🗂️ Pool 관리 |
| `admin-tab-history` | 📜 변경 이력 |
| `admin-tab-sys` | 🔧 시스템 정보 |

### 4.3 Dashboard KPI

| KPI | 설명 |
|-----|------|
| 전체 Pool | 전체 시리얼번호 수 |
| 배정됨 | 현재 배정된 수 |
| 가용 수량 | 미배정 가용 수 |
| 대기 승인 | 처리 대기 중 요청 수 |
| 최근 7일 신규 | 최근 7일 신규 배정 수 |

---

## 5. 동기화 아키텍처 — JSONBin.io

### 5.1 구성

```
PC A (사용자)                JSONBin.io                PC B (관리자)
     │                    [Bin ID: 6a39b74d...]              │
     │── PUT /v3/b/{id} ──────────────────────────────────── │
     │                                                        │
     │─────────────────── GET /v3/b/{id}/latest ────────────▶│
     │                                                        │
     └──────── 15초 폴링 (양방향 자동 동기화) ───────────────┘
```

### 5.2 고정 설정값

```javascript
// 파일 내 하드코딩 — 별도 설정 불필요
const JSONBIN_API_KEY = '$2a$10$AnfRJklYwNs79NBLSvzdnea3IUCQEdvK.98YiahGPazxSMWmoQg06';
const JSONBIN_BIN_ID  = '6a39b74df5f4af5e291fcff3';
```

### 5.3 동기화 트리거

| 트리거 | 동작 |
|--------|------|
| 앱 시작 시 | JSONBin에서 최신 데이터 즉시 로드 |
| 데이터 저장 시 | 300ms throttle 후 자동 PUT 업로드 |
| 15초마다 | 폴링 → 변경 감지 시에만 UI 갱신 |
| 탭 포커스 복귀 | `visibilitychange` 이벤트 → 즉시 동기화 |
| F5 누를 때 | 수동 즉시 동기화 |
| 창 닫기 전 | `beforeunload` → 저장 보장 |

### 5.4 변경 감지 서명

```javascript
// 서명 = nextId.log + audit_logs.length + nextId.license
function _makeSig(data) {
  return `${data?.nextId?.log}_${data?.audit_logs?.length}_${data?.nextId?.license}`;
}
// 서명이 같으면 UI 갱신 건너뜀 → 불필요한 리렌더 방지
```

### 5.5 무료 한도 (JSONBin.io Free Plan)

| 항목 | 한도 |
|------|------|
| 월간 요청 수 | 10,000 req |
| Bin당 용량 | 100 KB |
| 권장 용도 | 소규모 팀 (50인 이하) |

---

## 6. 함수 목록

### 6.1 Storage / 동기화

| 함수 | 설명 |
|------|------|
| `_checkApiKey()` | API Key 유효성 검사, `_apiReady` 플래그 설정 |
| `_makeSig(data)` | 변경 감지용 서명 문자열 생성 |
| `sharedGet()` | JSONBin에서 DB 데이터 읽기 (GET) |
| `sharedSet(data)` | JSONBin에 DB 데이터 쓰기 (PUT) |
| `loadDB()` | localStorage에서 DB 로드 |
| `saveDB()` | localStorage + JSONBin 동시 저장 (throttle 300ms) |
| `syncFromShared(silent)` | 원격 → 로컬 동기화, `silent=true`면 토스트 생략 |
| `_syncUI(state)` | 동기화 인디케이터 UI 업데이트 (`syncing`/`ok`/`err`/`local`) |
| `startAutoSync()` | 15초 폴링 타이머 시작 |
| `initDBFromIDB()` | 앱 시작 시 초기 데이터 로드 |
| `refreshCurrentPage()` | 현재 활성 페이지 재렌더 |

### 6.2 인증

| 함수 | 설명 |
|------|------|
| `doLogin()` | 이름+사번으로 로그인, `CU` 전역 변수 설정 |
| `doLogout()` | 로그아웃, `CU = null`, 로그인 화면 복귀 |
| `ensureSystemAdmin()` | DB에 Admin(0000) 반드시 존재하도록 보장 |

### 6.3 네비게이션

| 함수 | 설명 |
|------|------|
| `nav(page)` | 페이지 전환, 사이드바 active 갱신 |
| `sw(ns, id, btn)` | 탭 전환 (approval / admin 네임스페이스) |
| `swPool(id, btn)` | Pool 관리 탭 전환 |
| `onGS(q)` | 전역 검색 (이름/사번) |

### 6.4 게스트 신청

| 함수 | 설명 |
|------|------|
| `openGuestApply()` | 비로그인 신청 팝업 열기 |
| `closeGuestApply()` | 팝업 닫기 |
| `onGaLT()` | 게스트 신청 라이선스 유형 변경 핸들러 |
| `submitGuestApply()` | 게스트 신청 제출, 신규 사용자 자동 등록 포함 |

### 6.5 렌더링 — Dashboard

| 함수 | 설명 |
|------|------|
| `renderDashboard()` | KPI·차트·최근 변경 이력 전체 렌더 |
| `openPoolModal()` | 전체 Pool 모달 열기 |
| `closePoolModal()` | 모달 닫기 |
| `renderPoolModal()` | Pool 모달 내 필터·테이블 렌더 |
| `poolTypeSummaryHTML(pools)` | 유형별 요약 테이블 HTML 반환 |

### 6.6 렌더링 — 라이선스 신청/해지

| 함수 | 설명 |
|------|------|
| `renderApply()` | 신청 폼 렌더 (이미 신청 중이면 폼 숨김) |
| `onLT(radio)` | 라이선스 유형 라디오 변경 핸들러 |
| `fillApplyPool()` | 유형별 가용 수량 표시 갱신 |
| `submitApply()` | 신청 제출, 가용 Pool 확인 후 Request 생성 |
| `renderRevoke()` | 해지 신청 폼 렌더 |
| `submitRevoke(licId)` | 해지 요청 제출 |

### 6.7 렌더링 — 현황 조회

| 함수 | 설명 |
|------|------|
| `renderStatus()` | 필터(이름/사번/그룹/상태) 적용 테이블 렌더 |
| `downloadStatusExcel()` | 현황 Excel(.xlsx) 다운로드 |

### 6.8 렌더링 — Pool

| 함수 | 설명 |
|------|------|
| `renderPool()` | Pool 목록 테이블 렌더 |
| `refreshNsSerial()` | Pool 추가 폼 시리얼 드롭다운 갱신 |
| `onNsSerialChange()` | 시리얼 선택 시 상세 정보 표시 |
| `addPool()` | Pool 단건 추가 (라이선스 유형·시리얼·사용자 정보 필수) |
| `handlePoolUpload(input)` | Excel/CSV 파일 업로드로 Pool 일괄 등록 |
| `downloadPoolExcel()` | Pool 현황 Excel 다운로드 |
| `initPoolTab()` | Pool 탭 초기화 |

### 6.9 렌더링 — 승인 관리

| 함수 | 설명 |
|------|------|
| `renderApproval()` | 승인 관리 페이지 렌더 |
| `renderPending()` | 대기 중 요청 목록 렌더 |
| `doApprove(reqId)` | 요청 승인 처리, Pool·License 상태 자동 업데이트 |
| `doReject(reqId)` | 요청 반려 처리 |
| `renderApprHist()` | 전체 요청 이력 테이블 렌더 |

### 6.10 렌더링 — 관리자 설정

| 함수 | 설명 |
|------|------|
| `renderAdmin()` | 관리자 설정 페이지 렌더 (권한 체크 포함) |
| `renderAdminUsers()` | 사용자 목록 테이블 렌더 |
| `addUser()` | 사용자 추가 (라이선스 동시 배정 가능) |
| `forceDeleteUser(uid)` | 사용자 강제 삭제 |
| `renderAdminPerms()` | 관리자 목록·권한 부여 폼 렌더 |
| `grantAdmin()` | 관리자 권한 부여 |
| `revokeAdmin(uid)` | 관리자 권한 회수 |
| `renderAdminLic()` | 라이선스 등록/삭제 탭 렌더 |
| `adminAddLic()` | 관리자 직접 라이선스 배정 |
| `licSoftDelete(poolId)` | Pool 소프트삭제 (사용자 데이터 보존) |
| `licHardDelete(poolId)` | Pool 완전 삭제 |
| `deleteLic(poolId)` | `licSoftDelete` 래퍼 |
| `renderHistory()` | 변경 이력 테이블 렌더 |
| `clearHistory()` | 이력 초기화 (관리자 전용) |
| `renderSys()` | 시스템 정보 탭 렌더 |
| `resetAllData()` | 전체 데이터 초기화 (JSONBin 포함) |

### 6.11 라이선스 등록 모달 (관리자 → 사용자)

| 함수 | 설명 |
|------|------|
| `openAssignLicModal(uid)` | 특정 사용자에게 라이선스 등록 모달 열기 |
| `closeAssignLicModal()` | 모달 닫기 |
| `refreshAlmSerial()` | 유형별 가용 시리얼 드롭다운 갱신 |
| `confirmAssignLic()` | 라이선스 등록 확정 |

### 6.12 공지 / 업로드 / 유틸리티

| 함수 | 설명 |
|------|------|
| `renderLoginNotice()` | 로그인 화면 공지 문구 렌더 |
| `editLoginNotice()` | 공지 편집 모드 전환 |
| `saveLoginNotice()` | 공지 저장 |
| `cancelLoginNotice()` | 편집 취소 |
| `loadSheetJS(cb)` | SheetJS CDN 동적 로드 |
| `parseExcelFile(file, cb)` | Excel/CSV 파싱 후 콜백 |
| `normalizeCol(key)` | 컬럼명 정규화 (한영 매핑) |
| `handleAdminLicUpload(input)` | 관리자 라이선스 Excel 업로드 |
| `handleUpload(input)` | `handlePoolUpload` 래퍼 |
| `getNextAvailPool(licType)` | 유형별 다음 가용 Pool 반환 |
| `addLog(action, table, rid, desc)` | Audit Log 추가 |
| `toast(msg, type)` | 토스트 알림 (`s`=성공/`e`=오류/`w`=경고) |
| `refreshNuSerial(lt)` | 사용자 추가 폼 시리얼 드롭다운 갱신 |
| `onNuLicTypeChange()` | 사용자 추가 폼 라이선스 유형 변경 핸들러 |
| `onNuSerialChange()` | 사용자 추가 폼 시리얼 변경 핸들러 |

---

## 7. 로그인 및 권한

### 7.1 로그인 방식

- **입력**: 이름 + 사번 (양쪽 모두 일치해야 로그인)
- **DB 조회**: `DB.users`에서 `name`과 `employee_id` 동시 매칭
- **테스트 계정**: `이름: test, 사번: test` (로그인 공지 참고)

### 7.2 관리자 계정

| 구분 | 이름 | 사번 |
|------|------|------|
| 시스템 관리자 | Admin | 0000 |
| 일반 관리자 | DB의 `is_admin: true` 사용자 | - |

### 7.3 권한 분리

```
일반 사용자 → Dashboard / 라이선스 신청 / 해지 / 현황 조회
관리자      → 위 모든 기능 + 승인 관리 + 관리자 설정 전체
```

---

## 8. 업무 흐름

### 8.1 라이선스 신청 플로우

```
[사용자 / 게스트]
    │
    ├─ (비로그인) 로그인 화면 → '📝 라이선스 등록 신청' 버튼 → 팝업
    │                              이름·사번·그룹·Knox 입력
    │                              신규 사용자 자동 등록 포함
    │
    └─ (로그인 후) 사이드바 → '📝 라이선스 신청'
                              이름·사번·그룹·Knox 입력
                              유형 선택 (Process / Detail)
                              ↓
                       [신청 제출] → DB.requests 에 status:'pending' 추가
                              ↓
                   [관리자] 승인 관리 → 대기 목록 확인
                              ↓
                         승인 → Pool.is_assigned = true
                                License 레코드 생성
                                Audit Log 기록
                         반려 → 코멘트 저장, status:'rejected'
```

### 8.2 라이선스 해지 플로우

```
[사용자] 라이선스 해지 메뉴 → 보유 라이선스 목록
    │
    └─ 해지 사유 입력 → [해지 신청] → Request 생성 (type:'revoke')
                              ↓
                   [관리자] 승인 → License.status = 'revoked'
                                   Pool.is_assigned = false (자동 반납)
                                   Audit Log 기록
```

### 8.3 자동 배정 로직

```javascript
// 승인 시 getNextAvailPool() 호출
// → license_type 일치 + is_assigned=false + is_license_deleted=false
// → id 오름차순 정렬 → 첫 번째 Pool 배정
```

---

## 9. 설치 및 실행

### 9.1 로컬 실행

```bash
# 다운로드 후 브라우저에서 직접 열기
open EB_Visio_LMS_v1.html
```

> ⚠️ 일부 브라우저(Chrome)는 `file://` 프로토콜에서 CORS 제한이 있을 수 있습니다.
> 이 경우 로컬 서버를 사용하거나 GitHub Pages 배포를 권장합니다.

### 9.2 GitHub Pages 배포

```bash
# 1. GitHub 저장소 생성
git init
git add EB_Visio_LMS_v1.html
git commit -m "init"
git push origin main

# 2. GitHub 설정
# Settings → Pages → Branch: main / root → Save

# 3. 접속 URL
# https://[org].github.io/[repository-name]/EB_Visio_LMS_v1.html
```

### 9.3 로컬 서버 실행

```bash
# Python
python -m http.server 8080

# Node.js
npx serve .

# 접속
http://localhost:8080/EB_Visio_LMS_v1.html
```

---

## 10. 환경 설정값

### 10.1 JSONBin.io 설정 (코드 내 고정)

```javascript
// EB_Visio_LMS_v1.html (line 833~837)
const JSONBIN_API_KEY = '$2a$10$AnfRJklYwNs79NBLSvzdnea3IUCQEdvK.98YiahGPazxSMWmoQg06';
const JSONBIN_BASE    = 'https://api.jsonbin.io/v3';
const JSONBIN_BIN_ID  = '6a39b74df5f4af5e291fcff3';
```

> API Key 또는 Bin ID 변경 시 위 상수값만 수정하면 됩니다.

### 10.2 동기화 주기

```javascript
// 15초 폴링 (line 962)
_syncTimer = setInterval(() => syncFromShared(true), 15000);

// 저장 throttle (line 904)
_saveQueue = setTimeout(() => { sharedSet(DB); }, 300);
```

### 10.3 외부 라이브러리 (CDN)

| 라이브러리 | 버전 | 용도 |
|-----------|------|------|
| Chart.js | 4.4.0 | Dashboard 도넛·막대 차트 |
| SheetJS (XLSX) | latest CDN | Excel 파일 입출력 |

---

## 변경 이력

| 버전 | 날짜 | 변경 내용 |
|------|------|-----------|
| v1.0 | 2026-06-09 | 초기 릴리즈 |
| v1.1 | 2026-06-23 | JSONBin.io 실시간 멀티유저 동기화 추가, Bin ID·API Key 고정 내장 |

---

*EB · Visio LMS · Internal Use Only · 조선 EDH 그룹, 홍광섭 프로*
