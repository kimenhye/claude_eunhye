# 주간보고 (2026.03.13 ~ 2026.03.19)

---

## 금주 실적

### 내부 업무

#### 1. [ESQ-229] 공지 기능 수정 — 예약 로직 변경 (완료)

- **Controller → Service 분리**: NoticeController의 비즈니스 로직을 NoticeService로 이관 (Controller 166줄 → 52줄으로 경량화)
- **DTO 신규 생성** (6개):
  - `NoticeDeleteRequestDTO` / `NoticeDeleteResponseDTO`
  - `NoticeGetRequestDTO` / `NoticeGetResponseDTO`
  - `NoticeStorageRequestDTO` / `NoticeStorageResponseDTO`
  - `NoticeUpdateRequestDTO` / `NoticeUpdateResponseDTO`
- **Repository 정비**: NoticeRepository에 복합조건 쿼리 메서드 추가
- **Service 계층 기능 구현**:
  - 공지 저장/수정/삭제/조회/임시저장 서비스 로직 구현
  - 예약 공지 배치 처리 (`@Scheduled`, 10초 간격)
  - SSE 실시간 알림 발송 로직 분리
  - 페이징 처리 적용
- **UI 수정**: notice_versionUp.xml, popup_notice.xml 발송/예약/임시저장 탭 기능 연동
- **변경 규모**: 15개 파일, +506줄 / -248줄

#### 2. [ESQ-448] 공지 웹 에디터 (진행 중)

- `popup_notice.xml` 내용 작성 영역에 **편집/미리보기 탭 전환** 방식 마크다운 에디터 구현
- **marked.js** 라이브러리 추가 (마크다운 → HTML 파싱)
- **GitHub Markdown Light CSS** 적용 (`contents.css`에 통합)
- 마크다운 문법(제목, 굵게, 목록, 코드블록, 테이블 등) 실시간 미리보기 지원

#### 3. 기술지원 문서 양식 작성 (manager 버전)

### 기술지원

#### 미래등기
- 로그 분석 결과 전달 (3/17)

---

## 차주 계획

### 내부 업무
- [ESQ-448] 공지 웹 에디터 계속 진행
- 산업은행 오픈 지원 (3/23)

---

## Issue / 미결사항
- 미래등기 이슈
