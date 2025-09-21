# WarmUpdate 프로젝트 기술 스택 문서

**작성자**: [백승현](https://github.com/sirosho)

**문서 버전**: v1.1

**최종 수정일:** 2025.09.20

## 1. 개발 환경

* **운영체제:** Windows 11 Pro
* **웹 브라우저:** Google Chrome
* **개발 도구:**
    * **백엔드 & 프론트엔드:** IntelliJ IDEA

---

## 2. 프론트엔드 (Frontend)

* **프레임워크 및 라이브러리:**
    * **React:** 사용자 인터페이스 구축을 위한 JavaScript 라이브러리
    * **Zustand:** 상태 관리 라이브러리
    * **React Router DOM:** 페이지 라우팅 관리
    * **Axios:** 백엔드와 통신을 위한 HTTP 클라이언트
    * **FullCalendar:** 달력 기능 구현
* **패키지 매니저:** npm
* **빌드 도구:** Vite
    * **설정:** 개발 서버는 5005 포트를 사용하며, `/api`로 시작하는 요청은 백엔드 서버(`http://localhost:9005`)로 프록시 처리하여 CORS 문제를 해결
* **스타일링:** Sass / SCSS
* **테스트 및 린트:** ESlint

---

## 3. 백엔드 (Backend)

* **프레임워크:** Spring Boot 3.5.5
* **언어:** Java 17
* **주요 라이브러리:**
    * **Spring Data JPA & QueryDSL:** JPA(Hibernate) 기반의 ORM 및 타입-세이프 쿼리 작성
    * **Spring Security & OAuth2 Client:** 사용자 인증, 인가 및 소셜 로그인 기능
    * **Spring Boot Starter Web & Webflux:** RESTful API 및 비동기 HTTP 클라이언트(WebClient) 구축
    * **Lombok:** 코드 간소화 유틸리티
    * **JJWT:** JWT 기반 인증 토큰 생성/검증
    * **Spring Dotenv:** 환경 변수 관리
    * **Spring Validation, Mail, WebSocket:** Bean 유효성 검사, 이메일 발송, 실시간 통신 기능 제공
    * **Springdoc OpenAPI:** API 문서 자동화
* **데이터베이스:**
    * **주요 DB:** MariaDB
    * **개발/테스트 DB:** H2 Database (인메모리)
* **빌드 도구:** Gradle
* **API:** RESTful API
* **API 테스트 도구:** Swagger

---

## 4. 기타

* **AI 서비스:** Google Gemini API
* **외부 서비스 연동:** 카카오, 구글, 네이버 로그인 API, 한국천문연구원 API
* **협업 도구:** Git, GitHub
* **API 문서화:** Swagger