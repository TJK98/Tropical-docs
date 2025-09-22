# WarmUp Date 백엔드 프로젝트 클론 및 환경설정 가이드

본 문서는 WarmUp Date 백엔드 프로젝트를 로컬 환경에 클론하여 정상적으로 개발 및 실행하기 위한 환경 설정 방법과 필수 유의사항을 안내합니다.

**작성자:** [백승현](https://github.com/sirosho)

**문서 버전:** v1.0

**대상 독자:**
- **백엔드 개발자**: 개발 환경 세팅 및 실행 방법 숙지
- **QA/테스터**: 테스트 환경 구축 및 재현 환경 설정
- **신규 합류자**: 빠른 온보딩을 위해 로컬에서 프로젝트 실행이 필요한 인원
- **운영자**: 배포 전 로컬 환경 확인 및 기본 실행 점검

---

## **Quick Start (빠른 실행 가이드)**

프로젝트를 바로 실행해보고 싶으신 경우, 아래 절차를 따라주세요.

```bash
# 1. 저장소 클론
git clone https://github.com/Aicemelt/Tropical-backend.git

cd Tropical-backend

# 2. 데이터베이스 생성 (MariaDB 실행 필요)
CREATE DATABASE tropical_db
DEFAULT CHARACTER SET utf8mb4
DEFAULT COLLATE utf8mb4_unicode_ci;

# 3. 환경 변수 설정
# .env 파일을 생성하여 설정

# 4. 실행 (Gradle Wrapper 사용)
./gradlew bootRun   # Mac/Linux
gradlew.bat bootRun # Windows

# 5. 테스트 브라우저에서 접속하거나 프론트엔드 서버 구동 후 프론트엔드 서버 브라우저에 접속

API 테스트 url

http://localhost:9005/swagger-ui/index.html

프론트엔드 서버 url

http://localhost:5005

```

---

## **1. 사전 준비 사항**

프로젝트 설정을 시작하기 전에, 로컬 개발 환경에 아래 사항들이 먼저 준비되어야 합니다.

### **1-1. 필수 요구사항**

- JDK 17 이상
- MariaDB ([MariaDB Server 다운로드](https://mariadb.org/download/))
- Git

### **1-2. 데이터베이스 설정**

MariaDB에서 다음 명령을 실행하여 데이터베이스를 생성합니다

```sql
CREATE DATABASE tropical_db
DEFAULT CHARACTER SET utf8mb4
DEFAULT COLLATE utf8mb4_unicode_ci;
```

---

## **2. 환경 변수 설정**

프로젝트 실행을 위해 `.env` 파일을 생성하여 다음 환경 변수들을 설정해야 합니다.

`.env.example` 파일을 참고하여 환경변수를 설정합니다.

.env 파일 생성위치는 [프로젝트 구조](#5-프로젝트-구조) 를 참고 바랍니다.

> API 키 등 민감 정보의 보안을 위하여 `.env` 파일은 gitignore 목록에 추가되어 있습니다.

---

### **2-1. 외부 API 키 발급**

[Google Gemini API](https://aistudio.google.com/apikey)

[Chat gpt API](https://platform.openai.com/settings/organization/api-keys)

[한국천문연구원 특일 정보 API](https://www.data.go.kr/data/15012690/openapi.do)

[구글 로그인 API](https://developers.google.com/identity/protocols/oauth2)

[네이버 로그인 API](https://developers.naver.com/products/login/api/)

[카카오 로그인 API](https://developers.kakao.com/docs/latest/kakaologin)

---

### **2-2. JWT 시크릿 토큰 생성**

[JWT 키 생성가이드]()


## **3. 빌드 및 실행**

### **3-1. 프로젝트 빌드**

```bash
./gradlew build
```

### **3-2. 서버 실행**

```bash
./gradlew bootRun
```

기본 실행 포트는 `9005`입니다.

---

## **4. API 테스트**

서버가 정상적으로 실행되면 다음 URL에서 API를 테스트할 수 있습니다.

[http://localhost:9005/swagger-ui/index.html](http://localhost:9005/swagger-ui/index.html) 






---

## **5. 프로젝트 구조**

주요 디렉토리 구조는 다음과 같습니다:

```
Tropical-backend/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── tropical/
│   │   │           └── backend/
│   │   │               ├── auth/           # 로그인/회원가입 및 인증 관련 기능
│   │   │               ├── bucketList/     # 버킷리스트 기능
│   │   │               ├── calendar/       # 캘린더 관련 기능
│   │   │               ├── common/         # 공통 유틸리티
│   │   │               ├── config/         # 설정 파일
│   │   │               ├── diary/          # 일기 기능
│   │   │               ├── schedule/       # 일정 관리 기능
│   │   │               ├── smalltalk/      # 스몰토크 주제 추천 기능
│   │   │               ├── todo/           # 할 일 관리 기능
│   │   │               └── TropicalBackendApplication.java
│   │   └── resources/
│   │       ├── application.yml            # 스프링 부트 설정
│   │       └── static/
│   │           └── images/                # 정적 이미지 파일
│   └── test/                             # 테스트 코드
│       └── java/
│           └── com/
│               └── tropical/
│                   └── backend/           # 테스트 클래스들
├── gradle/                               # Gradle Wrapper 설정
│   └── wrapper/
├── .env                                 # 환경 변수 설정 파일
├── .env.example                         # 환경 변수 설정 예시
├── build.gradle                         # Gradle 빌드 설정
├── gradlew                              # Gradle Wrapper (Unix)
├── gradlew.bat                          # Gradle Wrapper (Windows)
└── settings.gradle                      # Gradle 프로젝트 설정

주요 패키지 설명:
- auth/: 사용자 인증 및 권한 관리
- bucketList/: 버킷리스트 CRUD 및 관리
- calendar/: 캘린더 뷰 및 일정 관리
- common/: 공통 유틸리티
- config/: 스프링 설정 및 보안 설정
- diary/: 일기 작성 및 관리
- schedule/: 일정 관리
- smalltalk/: 스몰토크 주제 추천 기능
- todo/: 할 일 목록 관리
```

---

## **6. 문제 해결**

일반적인 문제 해결 방법:

1. 데이터베이스 연결 오류
   - MariaDB 서비스가 실행 중인지 확인
   - 데이터베이스 접속 정보가 올바른지 확인 

2. 빌드 실패
   - Gradle 캐시 삭제 후 재시도: `./gradlew clean`
   - JDK 버전 확인

---

## **7. 참고 사항**

- 개발 환경에서는 JPA의 `ddl-auto`가 `create-drop`으로 설정되어 있어 서버 재시작 시 데이터가 초기화됩니다.
- 실제 운영 환경에서는 이 설정을 `update` 또는 `none`으로 변경해야 합니다.

---




