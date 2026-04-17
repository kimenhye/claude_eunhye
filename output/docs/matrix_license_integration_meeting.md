# Matrix - EdgeSquare 라이선스 통합 회의 안건

> **작성일:** 2026-04-17  
> **회의 예정:** 다음주 (Matrix 담당자)  
> **작성자:** 김은혜 (엣지솔루션팀)

---

## 1. 배경

Matrix 제품과 EdgeSquare 라이선스를 통합할지 분리할지에 대한 방향성 결정이 필요하다.  
현재 두 제품은 같은 WAR에 포함되어 있으나, DB와 라이선스 제어 방식에 불일치가 존재한다.

---

## 2. 현황 분석

### 2.1 라이선스 구조

| 항목 | 현재 상태 |
|------|----------|
| 라이선스 파일 | `license_EdgeSquare` 하나를 공유 |
| 제품 코드 | 32(Manager), 33(Monitor), 34(Deploy) — Matrix 전용 코드 없음 |
| 모드 결정 | 부팅 시 `edgesquare.mode` 시스템 프로퍼티로 결정 (monitor / manager) |
| 데모 라이선스 | `license_EdgeSquareDemo` 폴백 지원 |

**라이선스 JSON 주요 필드:**
```
sLicenseTrgtCd : 제품 코드 (쉼표 구분)
sExpiredDt     : 만료일 (yyyyMMdd)
sMaxAdmin      : 최대 관리자 수
sMaxUser       : 최대 에이전트 수
sMaxServer     : 최대 서버 수
```

### 2.2 DB 구조

| 구분 | DB명 | 용도 |
|------|------|------|
| EdgeSquare | `edgesquare` | Manager/Monitor/Deploy |
| Matrix | `edgesquare_matrix` | Matrix 전용 (앱 배포, 푸시, 크래시 등) |

- 두 DB 모두 `em_menu`, `em_preference` 테이블을 각각 보유
- `em_preference`에 라이선스 관련 키가 **양쪽에 중복** 존재:
  - `license.feature.group`
  - `license.manager.isActive`
  - `license.feature.organization`

### 2.3 메뉴 제어 방식 (`em_menu.ESQ_MODE`)

**EdgeSquare 쪽:** 1depth 메뉴에 `ESQ_MODE` 값으로 모드별 노출 제어

| ESQ_MODE | 의미 | 예시 메뉴 |
|----------|------|----------|
| `'monitor'` | Monitor 전용 (monitor 모드에서만 노출) | 실시간현황, iWorks, 탐색 등 |
| `'manager'` | Manager 전용 (manager 모드에서만 노출) | 대시보드, 배포 관리 등 |
| `NULL` | 하위 메뉴 (부모 메뉴의 ESQ_MODE를 따름) | 2depth 이하 메뉴들 |

**필터링 로직** (`AdminUserController.buildMenuInfo()`):
- `monitor` 모드 → 전체 메뉴 노출
- `manager` 모드 → 1depth에서 `ESQ_MODE='manager'`인 메뉴 + 하위만 노출

**Matrix 쪽 문제:** `em_menu` INSERT에 `ESQ_MODE` 컬럼이 **빠져있어 전부 NULL**

```
Matrix 1depth 메뉴 (ESQ_MODE = NULL):
├── 1000 대시보드
├── 4960 앱 관리
├── 5000 앱 위변조 방지
├── 5100 푸시 관리
├── 6000 모니터링
├── 7000 사용자 관리
└── 9000 관리자 설정
```

### 2.4 코드 구조

- 같은 WAR (`edgesquare-2.2.war`)에 Matrix 패키지(`com.inswave.appplatform.matrixmanager`)가 포함
- `StartManagerSpringApplication`에서 `matrixmanager` 패키지를 EntityScan/ComponentScan
- `LicenseService`는 공통 모듈 (`com.inswave.appplatform.common`)에 위치
- Matrix 전용 datasource: `datasource.site.enabled: false`, `datasource.wedgemanager`만 사용

---

## 3. 논의 안건

### 안건 1: 라이선스 통합 방향

**제안:** Manager(32) 라이선스 발급 시 Matrix 기능도 포함

| 방식 | 장점 | 단점 |
|------|------|------|
| **A. Manager(32)에 포함** | 라이선스 발급 변경 불필요, 기존 고객 즉시 적용 | Matrix만 별도 판매 불가 |
| **B. 별도 코드 추가 (예: 35)** | Matrix 단독 판매 가능, 세밀한 제어 | 라이선스 발급/검증 로직 수정 필요 |
| **C. 완전 분리 (별도 파일)** | 독립 운영 가능 | 고객 관리 복잡, 중복 코드 증가 |

> **결정 필요:** Matrix를 독립 제품으로 판매할 계획이 있는가?

### 안건 2: Matrix 메뉴 ESQ_MODE 적용

**현상:** Matrix DDL의 `em_menu` INSERT에 `ESQ_MODE` 값이 없음 (전부 NULL)

**필요 작업:**
- Matrix 1depth 메뉴에 `ESQ_MODE = 'manager'` 추가
- 이를 통해 `edgesquare.mode=manager`일 때 Matrix 메뉴 자동 노출

**확인 필요:**
- Matrix 메뉴 중 Monitor 모드에서도 보여야 하는 메뉴가 있는가?
- 아니면 전부 `'manager'`로 통일해도 되는가?

### 안건 3: DB 분리 유지 여부

| 방식 | 장점 | 단점 |
|------|------|------|
| **현행 유지 (분리)** | 변경 없음, 스키마 독립 관리 | DB 2개 운영, em_preference 중복 |
| **통합 (`edgesquare` 하나)** | 운영 단순, preference 일원화 | 마이그레이션 필요, 테이블명 충돌 가능, 기존 고객 영향 |

> **결정 필요:** 단기적으로 현행 유지하되, 중장기 통합 로드맵이 필요한가?

### 안건 4: em_preference 중복 정리

양쪽 DB에 동일한 라이선스 키가 존재:

```
license.feature.group        → 양쪽 동일
license.manager.isActive     → 양쪽 동일
license.feature.organization → 양쪽 동일
```

**논의:** 
- DB 분리를 유지한다면, 어느 쪽을 기준으로 할 것인지
- 값이 불일치할 경우 어떤 동작을 기대하는지

### 안건 5: 공통 테이블 DDL 스키마 불일치

EdgeSquare 표준 DDL (`schema/ddl/maria,mysql/DDL.sql`)과 Matrix DDL (`config/Matrix/DDL/Matrix_Manager_DDL.sql`)을 비교한 결과, **41개 공통 테이블 중 5개에서 컬럼 차이**가 발견되었다. (나머지 36개는 동일)

#### 5-1. EdgeSquare에만 있는 컬럼 (Matrix에 누락)

| 테이블 | 컬럼 | 타입 | 용도 |
|--------|------|------|------|
| `em_admin_user` | `logout_date` | DATETIME NULL | 로그아웃 일시 |
| `em_file_dist_package` | `bandwidth` | INT NULL | 배포 대역폭 |
| `em_file_dist_package` | `description` | VARCHAR(512) NULL | 패키지 설명 |
| `em_file_dist_version` | `description` | VARCHAR(512) NULL | 버전 설명 |
| `em_preference` | `data_type` | VARCHAR(32) NULL | 설정 값 타입 |

> EdgeSquare 표준 DDL이 더 최신이며, Matrix DDL에 반영이 안 된 상태

#### 5-2. Matrix에만 있는 컬럼 (Matrix 커스텀)

| 테이블 | 컬럼 | 타입 | 용도 |
|--------|------|------|------|
| `em_mobile_crash_log` | `issue_status` | VARCHAR(20) NULL | 이슈 상태 |
| `em_mobile_crash_log` | `issue_status_update_date` | VARCHAR(17) NULL | 이슈 상태 변경일 |
| `em_mobile_crash_log` | `issue_resolver_id` | VARCHAR(50) NULL | 이슈 해결자 ID |

> 크래시 로그 이슈 관리용 컬럼으로 Matrix에서 독자적으로 추가한 것으로 보임

#### 5-3. 테이블 분포 (참고)

| 구분 | 수량 | 비고 |
|------|------|------|
| 공통 테이블 | 41개 | 그 중 5개에서 컬럼 차이 |
| EdgeSquare에만 있는 테이블 | 3개 | em_notice, em_notice_target, verifier_webhook |
| Matrix에만 있는 테이블 | 14개 | esm_dashboard, esm_policy_integrity_*, esm_log_alert_*, em_debug_symbol 등 |

**논의:**
- DB 통합 시 공통 테이블의 스키마 동기화가 선행되어야 함
- EdgeSquare 표준 DDL 기준으로 Matrix DDL을 맞출 것인지
- Matrix 커스텀 컬럼(crash log 이슈 관리)을 EdgeSquare 표준에도 반영할 것인지

---

## 4. 제안 방향 (초안)

```
[단기] Manager(32) 라이선스에 Matrix 포함
       + Matrix em_menu에 ESQ_MODE='manager' 적용
       + DB는 현행 분리 유지

[중장기] DB 통합 여부 별도 검토
         em_preference 중복 정리
         공통 테이블 DDL 스키마 동기화
```

---

## 5. 참고 파일

| 파일 | 위치 |
|------|------|
| 라이선스 서비스 | `com.inswave.appplatform.common.service.LicenseService` |
| 라이선스 컨트롤러 | `com.inswave.appplatform.common.controller.LicenseController` |
| 메뉴 필터링 로직 | `AdminUserController.buildMenuInfo()` (line 725~752) |
| Menu 엔티티 | `com.inswave.appplatform.legacy.persistance.domain.Menu` (esqMode 필드) |
| 부팅 모드 결정 | `StartManagerSpringApplication.resolveModeFromLicense()` |
| EdgeSquare 표준 DDL | `schema/ddl/{DB종류}/DDL.sql` |
| Matrix DDL | `config/Matrix/DDL/Matrix_Manager_DDL.sql` |
| Matrix 설정 | `config/Matrix/application-matrix.yml` |
| Matrix SDD 문서 | `claude-team/.claude/docs/SDD_07_matrix.md` |
