# 💻 WarmUpdate 프로젝트 기술 스택 문서

**작성자**: [백승현](https://github.com/sirosho)

**문서 버전**: v1.0


## 1. 개발 환경

* **운영체제:** Windows 11 Pro
* **웹 브라우저:** Google Chrome
* **개발 도구:**
    * **백엔드 & 프론트엔드:** IntelliJ IDEA.



## 2. 프론트엔드 (Frontend)


* **프레임워크 및 라이브러리:**
    * **React:** 사용자 인터페이스를 구축하기 위한 JavaScript 라이브러리입니다.
    * **Zustand:** 상태 관리를 위한 라이브러리입니다.
    * **React Router DOM:** 애플리케이션 내 페이지 라우팅을 관리합니다.
* **패키지 매니저:** npm
* **빌드 도구:** Vite
    * **설정:** 개발 서버는 5005 포트를 사용하며, `/api`로 시작하는 요청은 백엔드 서버(`http://localhost:9005`)로 프록시 처리하여 CORS 문제를 해결합니다.
* **스타일링:** Sass / SCSS
* **테스트 및 린트:** ESlint


## 3. 백엔드 (Backend)

* **프레임워크:** Spring Boot 3.5.5
* **언어:** Java 17
* **주요 라이브러리:**
    * **Spring Boot Starter Data JPA:** 데이터베이스와의 상호작용을 위한 ORM(객체 관계 매핑) 기술입니다.
    * **Spring Boot Starter Security & OAuth2 Client:** 사용자 인증, 인가 및 소셜 로그인 기능을 구현합니다.
    * **Spring Boot Starter Web:** RESTful API를 구축합니다.
    * **Lombok:** 개발자의 코드 작성을 간편하게 도와주는 라이브러리입니다.
    * **QueryDSL:** JPQL을 타입-세이프하게 작성할 수 있도록 돕는 라이브러리입니다.
    * **JJWT:** JWT 기반의 인증 토큰을 생성하고 검증하는 데 사용됩니다.
    * **Spring Dotenv:** `.env` 파일을 로드하여 환경 변수를 관리합니다.
* **데이터베이스:**
    * **주요 DB:** MariaDB
    * **개발/테스트 DB:** H2 Database (인메모리 데이터베이스)
* **빌드 도구:** Gradle
* **API:** RESTful API



## 4. 기타

* **AI 서비스:** Google Gemini API
* **외부 서비스 연동:** 카카오, 구글, 네이버 로그인 API, 한국천문연구원 API
* **협업 도구:** Git, GitHub