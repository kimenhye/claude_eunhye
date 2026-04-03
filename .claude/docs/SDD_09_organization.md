# SDD-09: 조직/부서 관리

| 항목 | 내용 |
|------|------|
| 문서 버전 | v1.0 |
| 작성일 | 2026-04-03 |
| 대상 모듈 | Organization / Department |

---

## 1. 개요

### 1.1 목적
계층형 조직/부서 구조를 관리하고, 디바이스와 사용자를 조직에 매핑하는 기능을 설계한다.

### 1.2 범위
- 조직/부서 계층 관리
- 사용자 소속 관리
- 그룹 관리
- 조직 기반 대상 선택 (공지, 배포 등에서 활용)

---

## 2. 기능 설계

### 2.1 기능 목록

| No | 기능명 | 설명 | 우선순위 |
|----|--------|------|----------|
| 1 | 부서 관리 | 부서 생성/수정/삭제, 계층 구성 | 상 |
| 2 | 조직 조회 | 트리 형태 조직도 조회 | 상 |
| 3 | 사용자 관리 | 사용자-부서 매핑 | 중 |
| 4 | 그룹 관리 | 사용자 정의 그룹 관리 | 중 |

---

## 3. API 설계

| No | Controller | Endpoint | 설명 |
|----|-----------|----------|------|
| 1 | OrganizationController | /organization/info.do | 조직 정보 조회 (id=101: 조직, id=401: 그룹) |
| 2 | DepartmentController | /api/edge/department | 부서 관리 |

---

## 4. 데이터 설계

### 4.1 주요 엔티티

| 엔티티 | 테이블 | 설명 |
|--------|--------|------|
| Department | em_department | 부서 (DEPARTCODE, DEPARTNAME, DEPARTLEVEL, DEPARTPATH) |
| - | em_users | 사용자 (ID, NAME, DEPARTCODE) |
| - | em_organizations | 조직 JSON 저장소 |

### 4.2 em_department 테이블

| 컬럼 | 타입 | 설명 |
|------|------|------|
| DEPARTCODE | VARCHAR | 부서 코드 (PK) |
| DEPARTNAME | VARCHAR | 부서명 |
| DEPARTLEVEL | INT | 계층 레벨 |
| DEPARTPATH | VARCHAR | 상위 부서 경로 |

### 4.3 DataList 매핑 (공지 등에서 활용)

| DataList ID | 용도 | 주요 컬럼 |
|-------------|------|-----------|
| dlt_orgList | 조직 트리 | ORGCD, LEVEL, ORGNM, UPPERCD, ORGDISP |
| dlt_groupList | 그룹 트리 | ORGCD, ID, LEVEL, NAME, UPPERCODE, ORGDISP |

---

## 5. 상세 설계

### 5.1 조직 트리 표시 규칙
- LEVEL > 2: "(부서코드) 부서명" 형태로 표시
- LEVEL <= 2: 부서명만 표시

### 5.2 그룹 트리 표시 규칙
- LEVEL > 1: "그룹명(코드)" 형태로 표시
- LEVEL <= 1: 그룹명만 표시

### 5.3 라이선스 연동
- license.feature.organization: 조직 기능 사용 여부
- license.feature.group: 그룹 기능 사용 여부

---

## 6. 현행 이슈 및 개선사항

| No | 구분 | 현행 | 개선 | 비고 |
|----|------|------|------|------|
| | | | | |

---

## 변경 이력

| 버전 | 일자 | 변경 내용 |
|------|------|-----------|
| v1.0 | 2026-04-03 | 최초 작성 |
