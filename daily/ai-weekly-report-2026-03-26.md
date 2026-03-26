# AI 업무 혁신 주간보고

**보고일:** 2026-03-26  
**사번:** 100407  
**이름:** 김은혜  
**부서/팀:** UAP솔루션/엣지솔루션

---

## 업무 1. 공지 마크다운 에디터 방식 검토 및 확정 (ESQ-448)

### 기존 방식
- HTML 에디터(WYSIWYG) 방식으로 개발 시도
- contentEditable, iframe designMode 등 브라우저 네이티브 편집 API 직접 구현 필요
- 브라우저별 생성 HTML 상이 (Chrome: `<b>`, IE: `<strong>`, 복붙 시 인라인 스타일 난립)
- 수동으로 구현 시 약 1~2일 소요 예상

### AI 적용
- Claude Code로 WYSIWYG 3가지 방식(contentEditable div, iframe designMode, execCommand) 순차 시도
- WebSquare 프레임워크와 충돌하는 부분을 실시간으로 확인·분석
- 최종적으로 마크다운 원문 저장 방식이 호환성 최적임을 판단, 편집/미리보기 탭 + 마크다운 문법 삽입 툴바로 확정
- 툴바 JS(10개 버튼), CSS 스타일, HTML 구조 코드 전량 Claude 작성

### 결과
- 방식 검토부터 최종 구현까지 약 1시간 소요 (수동 시 약 1~2일 추정)
- 변경 규모: popup_notice.xml, contents.css — 475줄 추가 / 91줄 삭제
- 브라우저 호환성 확보 (마크다운 원문은 일반 텍스트이므로 환경 무관)
- 증적: `popup_notice.xml`, `contents.css` 변경분 (미커밋, git diff로 확인 가능)

---

## 업무 2. License Controller MVC 리팩토링 + 라이선스 로드 오류 해결

### 기존 방식
- LicenseController에 비즈니스 로직(날짜 변환, null 처리, 추적 데이터 조합) 직접 작성 (145줄)
- JSONObject 직접 반환, 타입 안전성 없음
- 라이선스 파일 경로 고정 → 개발 환경에서 파일 없으면 null 반환, 원인 파악 어려움
- 수동으로 DTO 설계 + Service 분리 + 디버깅 시 약 2~3시간 소요 예상

### AI 적용
- Claude Code로 기존 Notice 모듈의 DTO 패턴(Request/Response, Header/Body 구조) 분석
- 동일 패턴으로 LicenseDataResponseDTO, LicenseTrackingResponseDTO 자동 생성 (Swagger 어노테이션 포함)
- LicenseService에 getLicenseDataDTO(), getTrackingDTO() 비즈니스 로직 이관
- 라이선스 null 원인 분석: 파일 경로 추적 → classpath fallback 로직 추가
- 기동 시 라이선스 로드 결과 로그 출력 추가

### 결과
- 전체 작업 약 30분 소요 (수동 시 약 2~3시간 추정, 시간 절감 약 75%)
- Controller 145줄 → 93줄 (약 36% 경량화)
- DTO 2개, Service 메서드 2개 신규 생성
- 기존 프론트엔드(infoall.xml) 호환성 유지 (/api/license/data 레거시 응답 유지)
- 증적: `LicenseController.java`, `LicenseService.java`, `LicenseDataResponseDTO.java`, `LicenseTrackingResponseDTO.java` (미커밋, git diff로 확인 가능)

---

## 기술지원

- 산업은행 오픈 지원 (3/23)
- 우리카드 형상관리를 위한 구성 패치용 sh 작성 (3/25)
