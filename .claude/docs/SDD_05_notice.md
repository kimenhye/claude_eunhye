# SDD-05: 공지 관리

| 항목 | 내용 |
|------|------|
| 문서 버전 | v1.0 |
| 작성일 | 2026-04-03 |
| 대상 모듈 | Notice Management |

---

## 1. 개요

### 1.1 목적
시스템 공지사항을 작성하여 Edge 디바이스에 발송하는 기능을 설계한다.

### 1.2 범위
- 공지 작성 (제목, 내용, 유형, 표시 옵션)
- 수신 대상 지정 (조직/그룹/전체)
- 즉시/예약 발송
- 공지 이력 관리

---

## 2. 기능 설계

### 2.1 기능 목록

| No | 기능명 | 설명 | 우선순위 |
|----|--------|------|----------|
| 1 | 공지 작성 | 제목, 내용, 발신자, 유형 입력 | 상 |
| 2 | 수신 대상 지정 | 조직/그룹 트리에서 대상 선택 | 상 |
| 3 | 공지 유형 | information, warning, error 선택 | 중 |
| 4 | 알림창 표시 | Gear 알림창 표시 여부 (SHOW/HIDE) | 중 |
| 5 | 공지 발송 | 즉시 발송 | 상 |
| 6 | 이력 조회 | 발송 이력 조회 (페이징) | 중 |
| 7 | 대상 앱 선택 | AppId별 공지 관리 | 중 |

### 2.2 기능 흐름도

```
[대상 앱 선택] -> [수신자 선택 (조직/그룹)]
    -> [공지 작성 (제목/내용/유형)] -> [발송]
    -> [Edge Agent 수신] -> [알림 표시]
```

---

## 3. API 설계

### 3.1 API 목록

| No | Method | Endpoint | 설명 |
|----|--------|----------|------|
| 1 | POST | /api/edge/save/notification/publish | 공지 발송 |
| 2 | POST | /api/edge/service/notification/getAll | 공지 이력 조회 |
| 3 | GET | /api/edge/preference/like/edgemanager.appId. | 앱 목록 조회 |
| 4 | POST | /organization/info.do | 조직/그룹 조회 |

### 3.2 공지 발송 API 상세

- **URL**: `POST /api/edge/save/notification/publish`

**Request**
```json
{
  "header": { "appId": "앱ID" },
  "body": {
    "requestData": [{
      "notiId": 1234567890,
      "receiver": "000000",
      "title": "제목",
      "content": "내용",
      "sender": "발신자",
      "receiverType": "DEPTS|GROUPS",
      "appId": "앱ID",
      "displayPosition": "SHOW|HIDE",
      "link": [],
      "displayTime": 5000,
      "iconType": "information|warning|error",
      "iconData": ""
    }]
  }
}
```

**Response**
```json
{
  "body": {
    "resultCode": "0",
    "resultMessage": "성공메시지"
  }
}
```

---

## 4. 데이터 설계

### 4.1 주요 엔티티

| 엔티티 | 테이블 | 설명 |
|--------|--------|------|
| Notice | em_notice | 공지 정보 |
| NoticePK | - | 복합키 |
| NoticeTarget | em_notice_target | 공지 수신 대상 |
| NoticeTargetPK | - | 복합키 |

### 4.2 em_notice 테이블

| 컬럼 | 타입 | 설명 |
|------|------|------|
| NOTI_ID | VARCHAR | 공지 ID (PK) |
| APPID | VARCHAR | 앱 ID |
| TITLE | VARCHAR | 제목 |
| CONTENT | TEXT | 내용 |
| DISPLAY_TIME | INT | 표시 시간 (ms) |
| NOTI_TYPE | VARCHAR | 발송 유형 (now/reserved) |
| ICON_TYPE | VARCHAR | 아이콘 유형 (information/warning/error) |

### 4.3 DataList 매핑

| DataList ID | 용도 | 주요 컬럼 |
|-------------|------|-----------|
| dlt_history | 공지 이력 | notiId, appId, receiverType, receiver, sender, displayPosition, title, content, createdAt |
| dlt_orgList | 조직 목록 | ORGCD, LEVEL, ORGNM, UPPERCD |
| dlt_groupList | 그룹 목록 | ORGCD, ID, LEVEL, NAME, UPPERCODE |

---

## 5. 상세 설계

### 5.1 주요 로직

#### 수신자 유형 결정
- 조직/그룹 모두 미사용 시: 전체 발송 (receiver="000000")
- 조직 선택 시: receiverType="DEPTS"
- 그룹 선택 시: receiverType="GROUPS"

#### iconType 값
- information (정보)
- warning (경고)
- error (오류)
- alarm, question, custom (코드 주석에 정의, UI 미구현)

#### 알림창 표시
- SHOW: Gear 알림창에 표시
- HIDE: 알림창 미표시

### 5.2 라이선스 제약
- 조직/그룹 기능은 라이선스에 따라 활성화/비활성화
- 모두 미사용 시 전체 발송만 가능

---

## 6. 화면 설계

| 화면 ID | 파일 경로 | 설명 |
|---------|-----------|------|
| notice | /notice/notice.xml | 공지 메인 화면 |
| notice_versionUp | /notice/notice_versionUp.xml | 버전업 공지 |
| popup_notice | /notice/popup_notice.xml | 공지 팝업 |
| popup_sender | /notice/popup_sender.xml | 발신자 팝업 |
| link_create | /notice/link_create.xml | 링크 생성 |

---

## 7. 현행 이슈 및 개선사항

| No | 구분 | 현행 | 개선 | 비고 |
|----|------|------|------|------|
| 1 | iconType | iconType 값이 한국어("경고")로 저장됨 | 영문 value(warning)로 저장되어야 함 | getValue()가 label 반환 가능성 |
| 2 | notiDate | NaN-NaN-NaN NaN:NaN 표시 | 날짜 파싱 로직 수정 필요 | notiType=now일 때 발생 |

---

## 변경 이력

| 버전 | 일자 | 변경 내용 |
|------|------|-----------|
| v1.0 | 2026-04-03 | 최초 작성 |
