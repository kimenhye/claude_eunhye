# SDD-01: Edge 디바이스 관리

| 항목 | 내용 |
|------|------|
| 문서 버전 | v1.0 |
| 작성일 | 2026-04-03 |
| 대상 모듈 | Edge Device Management |

---

## 1. 개요

### 1.1 목적
Edge 디바이스의 등록, 상태 추적, 그룹 관리, 명령 실행 기능을 설계한다.

### 1.2 범위
- 디바이스 등록/해제
- 디바이스 상태 모니터링 (세션 상태)
- 디바이스 그룹 관리
- 원격 명령 실행
- 메시지 송수신

---

## 2. 기능 설계

### 2.1 기능 목록

| No | 기능명 | 설명 | 우선순위 |
|----|--------|------|----------|
| 1 | 디바이스 등록 | Edge Agent 등록/인증 | 상 |
| 2 | 디바이스 상태 조회 | 세션 상태, 접속 정보 조회 | 상 |
| 3 | 디바이스 그룹 관리 | 그룹 생성/수정/삭제, 멤버 관리 | 중 |
| 4 | 원격 명령 실행 | Edge에 명령 전송 및 결과 수신 | 상 |
| 5 | 메시지 관리 | 메시지 송수신, 예약 메시지 | 중 |
| 6 | 접근 제어 | Edge 디바이스 접근 권한 관리 | 상 |
| 7 | 상태 이력 | 디바이스 상태 변경 이력 관리 | 중 |

### 2.2 기능 흐름도

```
[Agent 설치] -> [디바이스 등록 요청] -> [Manager 인증/등록]
    -> [세션 연결] -> [상태 모니터링] -> [명령/메시지 수신]
```

---

## 3. API 설계

### 3.1 API 목록

| No | Controller | 주요 Endpoint | 설명 |
|----|-----------|---------------|------|
| 1 | EdgeRegisterController | /api/edge/register | 디바이스 등록 |
| 2 | EdgeCommandController | /api/edge/command | 명령 실행 |
| 3 | EdgeAccessController | /api/edge/access | 접근 제어 |
| 4 | EdgeGroupController | /api/edge/group | 그룹 관리 |
| 5 | EdgeHttpController | /api/edge/http | HTTP 통신 |
| 6 | EdgeMessageController | /api/edge/message | 메시지 관리 |

---

## 4. 데이터 설계

### 4.1 주요 엔티티

| 엔티티 | 테이블 | 설명 |
|--------|--------|------|
| Device | em_devices | 디바이스 정보 (EDGEID, APPID, OSTYPE, IP, HOSTNAME, SESSIONSTATE) |
| EdgeGroup | em_edge_group | 그룹 정보 |
| EdgeGroupMember | em_edge_group_member | 그룹 멤버 |
| EdgeMessage | em_edge_message | 메시지 |
| EdgeMessageReserved | em_edge_message_reserved | 예약 메시지 |
| EdgeStateHistory | em_edge_state_history | 상태 이력 |
| EdgePreference | em_edge_preference | 환경설정 |

### 4.2 em_devices 테이블

| 컬럼 | 타입 | 필수 | 설명 |
|------|------|------|------|
| EDGEID | VARCHAR | Y | Edge 고유 ID (PK) |
| APPID | VARCHAR | Y | 앱 ID |
| OSTYPE | VARCHAR | | OS 유형 |
| IP | VARCHAR | | IP 주소 |
| HOSTNAME | VARCHAR | | 호스트명 |
| SESSIONSTATE | VARCHAR | | 세션 상태 |

---

## 5. 상세 설계

### 5.1 주요 서비스

| 서비스 | 설명 |
|--------|------|
| EdgeService | Edge 핵심 비즈니스 로직 |
| EdgeCoreService | Edge 코어 기능 |
| EdgeStatusService | 상태 관리 |
| EdgeStateHistoryService | 상태 이력 관리 |
| EdgeGroupService | 그룹 관리 |
| EdgeMessageService | 메시지 관리 |

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
