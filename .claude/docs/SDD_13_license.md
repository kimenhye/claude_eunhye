# SDD-13: 라이선스 관리

| 항목 | 내용 |
|------|------|
| 문서 버전 | v3.1 |
| 작성일 | 2026-04-15 |
| 대상 모듈 | License Management |

---

## 1. 개요

### 1.1 목적
시스템 라이선스를 검증하고, 라이선스에 따라 기능 활성화/비활성화, 사용량 제한, 제품 모드를 제어하는 기능을 설계한다.

### 1.2 범위
- 라이선스 파일 복호화 및 검증
- 제품 모드 결정 (Manager / Monitor)
- 기능별 Feature Flag 제어
- 사용량 제한 (관리자, Agent, 서버)
- 사용량 트래킹 (5분 주기)
- 라이선스 정보 조회 API

### 1.3 용어 정의

| 용어 | 설명 |
|------|------|
| Feature Flag | em_preference에 저장된 기능 활성화 boolean 플래그 |
| Product Code | 32=Manager, 33=Monitor, 34=Deploy |
| edgesquare.mode | 제품 동작 모드 (manager / monitor) |

---

## 2. 시스템 아키텍처

### 2.1 라이선스 처리 흐름

```
[license_EdgeSquare 파일 (정식)]
    |
    v
[Phase 1: StartManagerSpringApplication.resolveModeFromLicense()] (Spring Boot 기동 전)
    | - license_EdgeSquare → 없으면 license_EdgeSquareDemo (데모 폴백)
    | - DES 복호화
    | - sLicenseTrgtCd에서 모드 결정
    | - edgesquare.mode 프로퍼티 설정
    v
[Phase 2: LicenseService @PostConstruct] (Spring 컨텍스트 초기화)
    | - license_EdgeSquare 로드 시도
    | - 실패 시 → loadDemoLicenseFallback() → license_EdgeSquareDemo 로드
    | - 데모: 제품코드 없으면 Monitor(33) 기본 적용, demo=true 설정
    | - licenseData (JSONObject) 메모리 보관
    v
[Runtime]
    | - LicenseController: REST API 제공 (demo 플래그 포함)
    | - AdminUserController: 로그인 시 경고 팝업 (만료/만료임박)
    | - EdgeRegisterController: Agent 접속 제한
    | - Frontend: cm.getSystemProperty()로 Feature Flag 조회
    v
[Tracking: 5분 주기 스케줄]
    | - 현재 사용량 수집 (admin, agent, server)
    | - Nitrite DB에 이력 저장
```

### 2.2 라이선스 파일

#### 정식 라이선스
- **경로**: `${application.path.home}/license/license_EdgeSquare`
- **폴백**: `classpath:license/license_EdgeSquare`
- **형식**: Base64 인코딩 + DES 암호화된 JSON
- **복호화**: 파일 5~35번째 문자에서 DES 키 추출, 나머지 본문 복호화

#### 데모 라이선스
- **경로**: `${application.path.home}/license/license_EdgeSquareDemo`
- **폴백**: `classpath:license/license_EdgeSquareDemo`
- **적용 조건**: 정식 라이선스(`license_EdgeSquare`) 파일이 없을 때 자동 폴백
- **제약**: 만료일만 체크, 수량/제품코드 검증 안 함
- **기본 동작**: 제품코드 없으면 Monitor(33) 자동 적용, `demo=true` 설정

### 2.3 라이선스 데이터 구조

```json
{
  "sMaxAdmin": "10",
  "sMaxUser": "100",
  "sMaxServer": "3",
  "sLicenseTrgtCd": "32,33",
  "sExpiredDt": "20261231"
}
```

| 필드 | 설명 | 특수값 |
|------|------|--------|
| sMaxAdmin | 최대 관리자 수 | -1 = 무제한 |
| sMaxUser | 최대 Agent 수 | -1 = 무제한 |
| sMaxServer | 최대 클러스터 서버 수 | -1 = 무제한 |
| sLicenseTrgtCd | 제품 코드 (콤마 구분) | 32=Manager, 33=Monitor, 34=Deploy |
| sExpiredDt | 만료일 (yyyyMMdd) | 정보성 표시만, 강제 차단 없음 |

---

## 3. 기능 설계

### 3.1 기능 목록

| No | 기능명 | 설명 | 우선순위 |
|----|--------|------|----------|
| 1 | 라이선스 복호화 | DES 복호화로 라이선스 파일 로드 | 상 |
| 2 | 제품 모드 결정 | sLicenseTrgtCd 기반 manager/monitor 모드 설정 | 상 |
| 3 | Agent 접속 제한 | sMaxUser 초과 시 신규 접속 차단 | 상 |
| 4 | Feature Flag | 기능별 활성화/비활성화 제어 | 상 |
| 5 | 라이선스 정보 조회 | REST API로 라이선스 정보 제공 | 중 |
| 6 | 사용량 트래킹 | 5분 주기 사용량 수집/저장 | 중 |
| 7 | 관리자 수 제한 | sMaxAdmin 초과 경고 | 중 |

### 3.2 제품 모드에 따른 컴포넌트 로딩

| 모드 | 조건 | 로딩되는 컴포넌트 | 메뉴 |
|------|------|-------------------|------|
| monitor | sLicenseTrgtCd에 "33" 포함 | edgemonitor 패키지 전체 (로그, APM, 대시보드) | 전체 메뉴 |
| manager | "33" 미포함 | edgemonitor 미로딩 | esqMode='manager'인 메뉴만 |

> **참고**: 데모/정식 라이선스 구분 없이 `edgesquare.mode` 기반으로 동일하게 메뉴가 필터링된다.

```
@ConditionalOnProperty(name="edgesquare.mode", havingValue="monitor", matchIfMissing=true)
→ MonitorAutoConfiguration이 edgemonitor 패키지 활성화
```

---

## 4. API 설계

### 4.1 API 목록

| No | Method | Endpoint | 설명 |
|----|--------|----------|------|
| 1 | POST | /api/license/data | 라이선스 정보 조회 (legacy) |
| 2 | POST | /api/license/info | 라이선스 정보 조회 (DTO) |
| 3 | POST | /api/license/tracking | 사용량 트래킹 조회 |

### 4.2 API 상세

#### POST /api/license/data (legacy)

**Response**
```json
{
  "sMaxAdmin": "10",
  "sMaxUser": "100",
  "sLicenseTrgtCd": "32,33",
  "expiredDate": "2026-12-31",
  "showProduct": "Manager Monitor",
  "features": { "manager": true, "monitor": true, "deploy": false },
  "registered": true,
  "expired": false,
  "demo": false
}
```

#### POST /api/license/info

**Response**
```json
{
  "resultCode": 0,
  "resultMessage": "license data loaded",
  "licenseDetail": {
    "maxAdmin": "10",
    "maxUser": "100",
    "maxServer": "3",
    "licenseTrgtCd": "32,33",
    "expiredDate": "2026-12-31"
  }
}
```

#### POST /api/license/tracking

**Response**
```json
{
  "resultCode": 0,
  "resultMessage": "tracking success",
  "timeStamp": "2026-04-03T10:00:00+09:00",
  "trackingAdminUser": { "currentCount": "5", "licenseLimit": "10" },
  "trackingAgent": { "currentCount": "42", "licenseLimit": "100" },
  "trackingServer": { "currentCount": "2", "licenseLimit": "3" }
}
```

---

## 5. 데이터 설계

### 5.1 em_preference 테이블 (라이선스 관련)

| DATA_KEY | DATA_VALUE | 설명 |
|----------|------------|------|
| edgesquare.mode | monitor / manager | 제품 동작 모드 |
| license.feature.group | true / false | 그룹 기능 |
| license.feature.organization | true / false | 조직 기능 |
| license.feature.distribute.apply | true / false | 배포 승인 워크플로우 |
| license.feature.mobileAppDist | true / false | 모바일 앱 배포 |

### 5.2 트래킹 저장소

| 저장소 | 경로 | 설명 |
|--------|------|------|
| Nitrite DB | {license}/licenseTracking | 5분 주기 사용량 이력 |
| 임계치 | 10MB | 큐 사이즈 초과 시 디스크 영속화 |
| 실행 조건 | Hazelcast 클러스터 마스터 노드에서만 실행 |

---

## 6. 상세 설계

### 6.1 복호화 알고리즘

```
1. 라이선스 파일 전체 읽기
2. 5~35번째 문자 추출 → Base64 디코딩 → DES 키
3. 나머지 문자열(0~4 + 35~) 결합 → Base64 디코딩 → DES 복호화
4. 복호화 결과 → JSON 파싱
```

- **알고리즘**: DES (Data Encryption Standard)
- **구현 위치**: `LicenseService.decipherLicense()`, `LicenseEnvironmentPostProcessor.decipherLicense()`
- **커스텀 Base64**: Java util 의존 없는 자체 디코더 사용 (Bootstrap 단계 호환)

### 6.2 Agent 접속 제한 로직

```
[Agent 접속 요청] → EdgeRegisterController.verify()
    |
    ├─ sMaxUser == -1 → 무제한 허용
    ├─ 현재 접속수 < sMaxUser → 토큰 발급 (접속 허용)
    └─ 현재 접속수 >= sMaxUser → 빈 토큰 반환 (접속 차단)
```

**응답에 포함되는 라이선스 정보**:
```json
{
  "license": {
    "registered": true,
    "maxUserCount": 100,
    "activeUserCount": 42
  }
}
```

### 6.3 Feature Flag 프론트엔드 사용 패턴

```javascript
// 단일 키 조회
var isGroupEnabled = cm.getSystemProperty("license.feature.group");

// 접두사 기반 조회 (모든 feature 플래그)
var allFeatures = cm.getSystemProperty("license.feature.", true);

// 조건부 UI 제어
if (isGroupEnabled === "true") {
    select_org.show();  // 그룹 선택 UI 표시
}
```

**사용하는 주요 화면**:

| 화면 | Feature Flag | 동작 |
|------|-------------|------|
| notice.xml | license.feature.organization, group | 조직/그룹 선택 UI 표시/숨김 |
| distribute/agent_app.xml | license.feature.mobileAppDist | 모바일 앱 배포 활성화 |
| distribute/group_management.xml | license.feature.group, organization | 그룹/조직 선택 |
| deployer/popup_deployTransferExecute.xml | license.feature.distribute.apply | 승인 워크플로우 표시 |

### 6.4 관리자 수 제한

- `/setting/account.xml`에서 관리자 목록 조회 시 라이선스 정보 함께 반환
- `sMaxAdmin` vs 현재 관리자 수 비교하여 경고 표시
- 강제 차단은 하지 않음 (경고만)

### 6.5 로그인 시 라이선스 경고 팝업

`LicenseService.getLicenseWarningMessage()`가 로그인 성공 시 호출되어 경고 메시지를 결정한다. 어떤 경우에도 **로그인을 차단하지 않는다** (팝업 알림만).

| 상태 | 팝업 메시지 | 조건 |
|------|------------|------|
| 미등록 | "��이선스가 등록되지 않았습니다." | 정식/데모 라이선스 둘 다 없음 |
| 만료 (정식) | "라이선스가 만료되었습니다." | `isExpired() && !demo` |
| 만료 (데모) | "데모 라이선스가 만료되었습니다." | `isExpired() && demo` |
| 만료 임박 (정식) | "라이선스 만료까지 N일 남았습니다." | 만료일 30일 이내, `!demo` |
| 만료 임박 (데모) | "데모 라이선스 만료까지 N일 남았습니다." | 만료일 30일 이내, `demo` |
| 데모 정상 (31일+) | 팝업 없음 (null) | `demo && daysLeft > 30` |
| 정식 정상 (31일+) | 팝업 없음 (null) | `!demo && daysLeft > 30` |

```
AdminUserController.login()
    ├─ 로그인 성공 (resultCode == 1)
    │   └─ licenseService.getLicenseWarningMessage()
    │       ├─ null → 팝업 없음
    │       └─ 메시지 있음 → result.put("licenseWarning", message)
    └─ Frontend: login.xml에서 licenseWarning 있으면 팝업 표시
```

### 6.6 데모 라이선스 UI 처리 (infoall.xml)

`/api/license/data` 응답의 `demo` 플래그에 따라 라이선스 정보 화면을 분기한다.

| 구분 | 정식 라이선스 | 데모 라이선스 |
|------|-------------|-------------|
| 제품 표시 | " Manager Monitor Deploy" | " DEMO " |
| 최대 관리자 수 | 표시 | **행 숨김** (hide) |
| 최대 에이전트 수 | 표시 | **행 숨김** (hide) |
| 만료일 | 표시 | 표시 |

```javascript
// infoall.xml — scwin.licenseInfo() 콜백
if (licenseData.demo) {
    showProduct = " DEMO ";
    trMaxAdmin.hide();   // 최대 관리자 수 행
    trMaxUser.hide();    // 최대 에이전트 수 행
} else {
    // 기존 제품코드 기반 표시 로직
}
```

### 6.8 메뉴 필터링 (buildMenuInfo)

`AdminUserController.buildMenuInfo()`는 로그인 성공 시 사용자에게 반환할 메뉴 목록을 결정한다. 데모/정식 라이선스 구분 없이 `edgesquare.mode`에 따라 동일한 필터링 규칙을 적용한다.

```
[buildMenuInfo()]
    |
    ├─ mode == "monitor" → 전체 메뉴 반환
    └─ mode == "manager"
        ├─ 1depth: parentMenuId == null && esqMode == "manager" → allowedIds 수집
        ├─ 하위 메뉴: parentMenuId ∈ allowedIds → allowedIds에 추가 (반복 탐색)
        └─ allowedIds에 포함된 메뉴만 반환
```

| 라이선스 유형 | 모드 | 노출 메뉴 |
|-------------|------|----------|
| 정식 | manager | esqMode='manager'인 1depth + 하위 메뉴 |
| 정식 | monitor | 전체 메뉴 |
| 데모 | manager | esqMode='manager'인 1depth + 하위 메뉴 (정식과 동일) |
| 데모 | monitor | 전체 메뉴 (정식과 동일) |

> **변경 이력 (v3.1)**: 기존에는 `licenseService.isDemo() == true`이면 mode 무관하게 전체 메뉴를 반환했으나, 정식 라이선스와 동일한 mode 기반 필터링으로 변경.

---

### 6.7 트래킹 스케줄러

```java
@Scheduled(cron = "0 */5 * * * *")  // 5분마다
public void tracking() {
    // Hazelcast 마스터 노드에서만 실행
    // 현재 admin 수, agent 접속 수, 클러스터 서버 수 수집
    // Nitrite DB에 저장
}
```

---

## 7. 주요 클래스 참조

### 7.1 Backend

| 클래스 | 경로 | 설명 |
|--------|------|------|
| StartManagerSpringApplication | (root) | Spring 기동 전 라이선스 → edgesquare.mode 결정 (데모 폴백 포함) |
| LicenseService | common/service/ | 라이선스 핵심 서비스 |
| LicenseController | common/controller/ | REST API |
| LicenseDataResponseDTO | edgemanager/controller/api/dto/license/ | 라이선스 조회 응답 |
| LicenseTrackingResponseDTO | edgemanager/controller/api/dto/license/ | 트래킹 응답 |
| MonitorAutoConfiguration | edgemanager/config/ | 모드별 컴포넌트 조건 로딩 |
| WemDataSourceConfig | edgemanager/config/ | 모드별 엔티티 패키지 스캔 |
| EdgeRegisterController | edgemanager/controller/ | Agent 접속 시 라이선스 검증 |

### 7.2 Frontend

| 파일 | 경로 | 설명 |
|------|------|------|
| licenseInfo.xml | /xml/ | 라이선스 정보 화면 |
| infoall.xml | /xml/ | 대시보드 라이선스 정보 |
| common.js | /cm/js/ | cm.getSystemProperty() 구현 |
| account.xml | /setting/ | 관리자 수 제한 경고 |

---

## 8. 에러 처리 및 폴백

| 상황 | 동작 |
|------|------|
| 정식 라이선스 파일 없음 | license_EdgeSquareDemo로 데모 폴백, demo=true |
| 정식+데모 둘 다 없음 | mode=monitor 기본값, licenseData=null |
| 복호화 실패 | 에러 로그, licenseData=null |
| DB 동기화 실패 | 경고 로그, 파일 기반 데이터 사용 |
| Agent 제한 초과 | 빈 토큰 반환 (접속 차단) |
| Feature Flag 없음 | false 기본값 (기능 비활성화) |
| 만료일 초과 | 정보성 표시만 (강제 차단 없음) |

---

## 9. 제약사항 및 고려사항

### 9.1 제약사항
- DES 암호화 사용 (보안 취약, 레거시)
- 만료일은 정보 표시만, 강제 적용 없음
- Feature Flag는 em_preference 수동 설정 (라이선스 파일과 별도 관리)

### 9.2 보안 고려사항
- 라이선스 키가 파일 내에 포함됨 (자체 포함 구조)
- DES는 현대 보안 기준에 부적합 → AES 전환 검토 필요

### 9.3 성능 고려사항
- 트래킹 5분 주기: Nitrite 큐 10MB 임계치
- 클러스터 마스터에서만 트래킹 실행 (중복 방지)

---

## 10. 현행 이슈 및 개선사항

| No | 구분 | 현행 | 개선 | 비고 |
|----|------|------|------|------|
| 1 | 암호화 | DES 사용 | AES-256 전환 검토 | 보안 취약점 |
| 2 | 만료일 | 정보 표시만 | 만료 시 기능 제한 정책 필요 | 비즈니스 판단 |
| 3 | Feature Flag | DB 수동 설정 | 라이선스 파일에서 자동 매핑 | 관리 편의성 |
| 4 | 관리자 제한 | 경고만 표시 | 초과 시 생성 차단 | 정책 결정 필요 |

---

## 변경 이력

| 버전 | 일자 | 변경 내용 |
|------|------|-----------|
| v1.0 | 2026-04-03 | 최초 작성 |
| v2.0 | 2026-04-03 | 코드 분석 기반 상세 설계 추가 |
| v3.0 | 2026-04-15 | 데모 라이선스 폴백(license_EdgeSquareDemo), 로그인 경고 팝업(만료 30일 전), 데모 UI 처리(infoall.xml DEMO 표시/행 숨김), API demo 필드 추가 |
| v3.1 | 2026-04-15 | buildMenuInfo() 데모 메뉴 필터링 수정 — 데모/정식 무관하게 edgesquare.mode 기반 동일 필터링 적용 |
