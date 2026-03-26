# 주간보고 (2026.03.20 ~ 2026.03.26)

---

## 금주 실적

### 내부 업무

#### 1. [ESQ-448] 공지 마크다운 에디터 개선 (진행 중)

- **WYSIWYG 방식 시도 → 마크다운 에디터로 확정**: contentEditable, iframe designMode 등 WYSIWYG 방식 검토 후 브라우저 호환성 문제로 마크다운 원문 저장 방식으로 최종 결정
- **마크다운 툴바 추가**: 굵게(B), 기울임(I), 제목(H), 목록(•/1.), 인용(❝), 코드(</>), 링크(🔗), 표(▦), 구분선(―) 버튼 구현
- **편집/미리보기 탭 복원**: textarea 편집 + marked.js 미리보기 탭 전환 방식
- **변경 파일**: popup_notice.xml, contents.css (475줄 추가 / 91줄 삭제)

#### 2. License MVC 구조 리팩토링 (진행 중)

- **Controller → Service 비즈니스 로직 분리**: LicenseController의 날짜 변환, null 처리 등 비즈니스 로직을 LicenseService로 이관 (Controller 145줄 → 93줄)
- **DTO 신규 생성** (2개):
  - `LicenseDataResponseDTO` — 라이선스 정보 조회 응답 (maxAdmin, maxUser, maxServer, expiredDate)
  - `LicenseTrackingResponseDTO` — 추적 데이터 응답 (currentCount, licenseLimit)
- **API 엔드포인트 정리**:
  - `/api/license/data` — 기존 레거시 응답 유지 (프론트엔드 호환)
  - `/api/license/info` — 신규 DTO 구조화 응답
  - `/api/license/tracking` — 신규 추적 데이터 API
- **라이선스 파일 경로 fallback 추가**: application home에 라이선스 파일 없을 시 classpath(resources/license)에서 자동 탐색

#### 3. 기타 커밋

- banner.txt 버전 수정 (Suite 2.1 → 2.2)
- Notice DTO 디스크립션 수정

### 기술지원

- 없음

---

## 차주 계획

### 내부 업무
- [ESQ-448] 공지 마크다운 에디터 마무리 및 테스트
- License MVC 리팩토링 커밋 및 테스트
- 라이선스 트래킹 동작 검증

---

## Issue / 미결사항
- 라이선스 트래킹 API 인증(401) 이슈 — 로그인 세션 기반 호출 필요, 서버 로그로 동작 확인 예정

---

## Claude Code 활용 내역

### 공지 마크다운 에디터 방식 결정 (ESQ-448)
- **과정**: WYSIWYG(contentEditable → iframe designMode) 3가지 방식 시도 후 WebSquare 프레임워크 호환성 문제 확인
- **결론**: 마크다운 원문 저장 + marked.js 렌더링 방식이 브라우저 호환성에 최적임을 확인, 편집/미리보기 탭 + 마크다운 문법 삽입 툴바로 확정
- **기여도**: 툴바 JS 함수, CSS 스타일, HTML 구조 변경 코드 전량 Claude 작성

### License MVC 리팩토링
- **DTO 클래스 2개 생성**: LicenseDataResponseDTO, LicenseTrackingResponseDTO (Swagger 어노테이션 포함)
- **LicenseService 비즈니스 로직 이관**: getLicenseDataDTO(), getTrackingDTO() 메서드 작성
- **LicenseController 경량화**: 비즈니스 로직 제거, Service 위임 구조로 변경
- **기여도**: DTO 설계, Service 메서드, Controller 리팩토링 코드 전량 Claude 작성

### 라이선스 파일 로드 실패 원인 분석 및 해결
- **문제**: `/api/license/data` 응답이 null
- **분석**: LicenseService.postConstruct() → 파일 경로 추적 → `${user.home}/edgesquare/develop/home/license/` 디렉토리에 `license_EdgeSquare` 파일 부재 확인
- **해결**: classpath fallback 로직 추가 + 기동 시 라이선스 로드 결과 로그 출력
