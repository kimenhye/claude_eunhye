# 주간보고 (2026.03.13 ~ 2026.03.19)

## 1. [ESQ-229] 공지 메시지 MVC 모델 적용 (완료)

**기존 문제**: NoticeController에 비즈니스 로직이 혼재되어 유지보수 어려움

**작업 내용**:
- **Controller → Service 분리**: NoticeController의 비즈니스 로직을 NoticeService로 이관 (Controller 166줄 → 52줄으로 경량화)
- **DTO 신규 생성** (6개):
  - `NoticeDeleteRequestDTO` / `NoticeDeleteResponseDTO`
  - `NoticeGetRequestDTO` / `NoticeGetResponseDTO`
  - `NoticeStorageRequestDTO` / `NoticeStorageResponseDTO`
  - `NoticeUpdateRequestDTO` / `NoticeUpdateResponseDTO`
- **Repository 정비**: NoticeRepository에 복합조건 쿼리 메서드 추가 (`findByAppIdAndSenderInAndIconTypeInAndNotiDateBetween` 등)
- **Service 계층 기능 구현**:
  - 공지 저장/수정/삭제/조회/임시저장 서비스 로직 구현
  - 예약 공지 배치 처리 (`@Scheduled`, 10초 간격)
  - SSE 실시간 알림 발송 로직 분리
  - 페이징 처리 적용
- **UI 수정**: notice_versionUp.xml, popup_notice.xml 발송/예약/임시저장 탭 기능 연동
- **변경 규모**: 15개 파일, +506줄 / -248줄

## 2. 공지 메시지 마크다운 에디터 적용 (진행 중)

**작업 내용**:
- `popup_notice.xml` 내용 작성 영역에 **편집/미리보기 탭 전환** 방식 마크다운 에디터 구현
- **marked.js** 라이브러리 추가 (마크다운 → HTML 파싱)
- **GitHub Markdown Light CSS** 적용 (`contents.css`에 통합)
- 마크다운 문법(제목, 굵게, 목록, 코드블록, 테이블 등) 실시간 미리보기 지원

## 3. 개발환경 설정 변경 (진행 중)

- `application.yml` DB 접속 정보 변경 (MariaDB → Oracle)
- Hazelcast 설정 수정
- IntelliJ Tomcat 실행 설정 파일 업데이트
