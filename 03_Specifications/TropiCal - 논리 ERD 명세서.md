# TropiCal 논리 ERD 명세서

본 문서는 **TropiCal 플랫폼**의 데이터베이스 구조를 정의하는 **최종 논리적 ERD 명세서**입니다.

**작성자:** [백승현](https://github.com/Sirosho)

**문서 버전:** v1.0

**대상 독자:**

- **프로젝트 멤버** : 엔티티 구조와 관계 정의를 코드(Entity/Repository)로 구현


## 1. 개요

### 1.1. 문서 목적

본 명세서는 프로젝트에 참여하는 **개발자, 데이터베이스 관리자(DBA), 기획자** 등 모든 구성원이 데이터 구조에 대해 일관된 이해를 갖는 것을 목표로 합니다.

### 1.2. 표기법 안내
-   **PK (Primary Key)**: 테이블의 각 레코드를 고유하게 식별하는 기본 키입니다.
-   **FK (Foreign Key)**: 다른 테이블의 PK를 참조하여 테이블 간의 관계를 설정하는 외래 키입니다.
-   **UNIQUE**: 해당 컬럼(또는 컬럼 조합)의 모든 값이 고유해야 함을 나타내는 제약 조건입니다.
-   **NOT NULL**: 해당 컬럼에 반드시 값이 존재해야 함을 의미합니다.

---

## 2. 테이블 명세

### 2.0. 다이어그램

<img src="../assets/DB다이어그램.png" width="40%" alt="DB다이어그램">



### 2.1. user
-   **설명**: 플랫폼의 모든 사용자에 대한 기본 정보를 저장하는 핵심 테이블입니다. 로컬 및 소셜 로그인 사용자를 모두 포함합니다.
-   **비고**: 사용자의 알림, 캘린더 설정 등 개인화 관련 정보가 포함됩니다. `email` 및 `nickname`은 고유성을 보장합니다.

| 컬럼명 | 데이터 타입 | 키 | 제약 조건 | 설명 |
|---|---|---|---|---|
| `id` | BIGINT | PK | NOT NULL, AUTO_INCREMENT | 사용자 고유 ID |
| `email` | VARCHAR(255) | | NOT NULL | 이메일 주소 |
| `password_hash` | VARCHAR(255) | | NULL | 암호화된 비밀번호 (로컬 계정) |
| `email_verified` | TINYINT(1) | | NOT NULL, DEFAULT 0 | 이메일 인증 완료 여부 |
| `account_type` | ENUM('LOCAL', 'SOCIAL') | | NOT NULL | 계정 타입 |
| `nickname` | VARCHAR(50) | UNIQUE | NOT NULL | 사용자 닉네임 |
| `birth_date` | DATE | | NULL | 생년월일 |
| `onboarding_completed` | TINYINT(1) | | NOT NULL, DEFAULT 0 | 온보딩 완료 여부 |
| `created_at` | DATETIME | | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 계정 생성 시간 |
| `last_login_at` | DATETIME | | NULL | 마지막 로그인 시간 |
| `status` | ENUM('ACTIVE', 'BLOCKED', 'DELETED') | | NOT NULL, DEFAULT 'ACTIVE' | 계정 상태 |
| `week_start` | ENUM('SUN', 'MON') | | NOT NULL, DEFAULT 'MON' | 캘린더 주 시작 요일 |
| `timezone` | VARCHAR(50) | | NOT NULL, DEFAULT 'Asia/Seoul' | 시간대 설정 |
| `show_holidays` | TINYINT(1) | | NOT NULL, DEFAULT 1 | 공휴일 표시 여부 |
| `smalltalk_notification_enabled`| TINYINT(1) | | NOT NULL, DEFAULT 1 | 스몰 토크 알림 활성화 |
| `smalltalk_notification_time`| TIME | | NOT NULL, DEFAULT '08:00:00'| 스몰 토크 알림 시간 |
| `smalltalk_notification_days`| VARCHAR(20) | | NOT NULL, DEFAULT 'daily' | 알림 요일 |
| `schedule_notification_enabled`| TINYINT(1) | | NOT NULL, DEFAULT 1 | 일정 알림 활성화 |
| `schedule_notification_minutes`| INT | | NOT NULL, DEFAULT 30 | 일정 시작 전 알림 (분) |
| `todo_notification_enabled` | TINYINT(1) | | NOT NULL, DEFAULT 1 | 투두 마감 알림 활성화 |
| `todo_notification_time` | TIME | | NOT NULL, DEFAULT '08:00:00'| 투두 마감 알림 시간 |
| `todo_notification_days_before`| INT | | NOT NULL, DEFAULT 1 | 마감 며칠 전 알림 |

### 2.2. social_account
-   **설명**: 소셜 로그인을 통해 가입한 사용자의 연동 정보를 관리하는 테이블입니다.
-   **비고**: `provider`와 `provider_user_id`의 조합은 고유합니다. `ON DELETE CASCADE` 제약조건으로 사용자 삭제 시 관련 소셜 계정 정보도 삭제됩니다.

| 컬럼명 | 데이터 타입 | 키 | 제약 조건 | 설명 |
|---|---|---|---|---|
| `id` | BIGINT | PK | NOT NULL, AUTO_INCREMENT | 소셜 연동 정보 고유 ID |
| `user_id` | BIGINT | FK | NOT NULL | 연결된 사용자 ID |
| `provider` | ENUM('KAKAO', 'GOOGLE', 'NAVER')| UNIQUE | NOT NULL | 소셜 로그인 제공자 |
| `provider_user_id` | VARCHAR(255)| UNIQUE | NOT NULL | 소셜 제공자의 사용자 고유 ID |
| `linked_at` | DATETIME | | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 소셜 계정 연결 시간 |

### 2.3. user_consent
-   **설명**: 사용자가 서비스 약관, 개인정보 처리방침 등에 동의한 기록을 저장하는 테이블입니다.
-   **비고**
    - `user_id`와 `consent_type`의 조합은 중복될 수 없습니다.
    - `consent_type` 은 ENUM으로 관리되며 ENUM의 종류는 다음과 같습니다.

        - `TERMS_OF_SERVICE`
        - `CALENDAR_PERSONALIZATION`
        - `DIARY_PERSONALIZATION`
        - `TODO_PERSONALIZATION`
        - `BUCKET_PERSONALIZATION`


<br>

| 컬럼명 | 데이터 타입 | 키 | 제약 조건 | 설명 |
|---|---|---|---|---|
| `id` | BIGINT | PK | NOT NULL, AUTO_INCREMENT | 동의 정보 고유 ID |
| `user_id` | BIGINT | FK | NOT NULL | 대상 사용자 ID |
| `consent_type` | ENUM(...) | UNIQUE | NOT NULL | 동의 항목 타입 |
| `agreed` | TINYINT(1) | | NOT NULL | 동의 여부 |
| `agreed_at` | DATETIME | | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 동의 처리 시간 |




### 2.4. holiday
-   **설명**: 국가별 공휴일 정보를 관리하는 마스터 테이블입니다.
-   **비고**: `country_code`, `start_date`, `end_date`, `name_ko`의 조합은 고유합니다.

| 컬럼명 | 데이터 타입 | 키 | 제약 조건 | 설명 |
|---|---|---|---|---|
| `id` | BIGINT | PK | NOT NULL, AUTO_INCREMENT | 공휴일 정보 고유 ID |
| `country_code` | CHAR(2) | UNIQUE | NOT NULL, DEFAULT 'KR' | 국가 코드 |
| `start_date` | DATE | UNIQUE | NOT NULL | 공휴일 시작 날짜 |
| `end_date` | DATE | UNIQUE | NOT NULL | 공휴일 종료 날짜 |
| `name_ko` | VARCHAR(100) | UNIQUE | NOT NULL | 공휴일 이름 (한국어) |
| `is_substitute` | TINYINT(1) | | NOT NULL, DEFAULT 0 | 대체공휴일 여부 |
| `year` | SMALLINT | | NOT NULL | 연도 |

### 2.5. schedule
-   **설명**: 사용자가 작성한 개인 일정 정보를 저장하는 테이블입니다.
-   **비고**: `user_id`에 대한 외래키 제약조건이 설정되어 있습니다.

| 컬럼명 | 데이터 타입 | 키 | 제약 조건 | 설명 |
|---|---|---|---|---|
| `schedule_id` | BIGINT | PK | NOT NULL, AUTO_INCREMENT | 일정 고유 ID |
| `user_id` | BIGINT | FK | NOT NULL | 사용자 ID |
| `title` | VARCHAR(255) | | NOT NULL | 일정 제목 |
| `memo` | TEXT | | NULL | 일정 관련 메모 |
| `schedule_date` | DATE | | NOT NULL | 일정 날짜 |
| `start_time` | TIME | | NULL | 일정 시작 시간 |
| `end_time` | TIME | | NULL | 일정 종료 시간 |
| `location` | VARCHAR(100) | | NULL | 일정 장소 |
| `attendees` | VARCHAR(255) | | NULL | 참여자 |
| `is_completed` | TINYINT(1) | | NOT NULL, DEFAULT 0 | 일정 완료 여부 |
| `created_at` | DATETIME | | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성 시간 |
| `updated_at` | DATETIME | | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 수정 시간 |

### 2.6. diary
-   **설명**: 사용자가 작성한 일기 및 관련 감정, 날씨 정보를 관리하는 테이블입니다.
-   **비고**: `user_id`에 대한 외래키 제약조건이 설정되어 있습니다.

| 컬럼명 | 데이터 타입 | 키 | 제약 조건 | 설명 |
|---|---|---|---|---|
| `diary_id` | BIGINT | PK | NOT NULL, AUTO_INCREMENT | 일기 고유 ID |
| `user_id` | BIGINT | FK | NOT NULL | 사용자 ID |
| `title` | VARCHAR(255) | | NOT NULL | 일기 제목 |
| `content` | TEXT | | NOT NULL | 일기 본문 내용 |
| `emotion` | VARCHAR(20) | | NOT NULL | 감정 |
| `weather` | VARCHAR(20) | | NOT NULL | 날씨 |
| `diary_date` | DATE | | NOT NULL | 일기 날짜 |
| `created_at` | DATETIME | | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성 시간 |
| `updated_at` | DATETIME | | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 수정 시간 |

### 2.7. todo
-   **설명**: 사용자의 할 일(To-Do) 목록을 관리하는 테이블입니다.
-   **비고**: `user_id`에 대한 외래키 제약조건이 설정되어 있습니다.

| 컬럼명 | 데이터 타입 | 키 | 제약 조건 | 설명 |
|---|---|---|---|---|
| `todo_id` | BIGINT | PK | NOT NULL, AUTO_INCREMENT | Todo 고유 ID |
| `user_id` | BIGINT | FK | NOT NULL | 사용자 ID |
| `content` | TEXT | | NOT NULL | 할 일 내용 |
| `due_date` | DATE | | NULL | 마감 기한 |
| `is_completed` | TINYINT(1) | | NOT NULL, DEFAULT 0 | 완료 여부 |
| `created_at` | DATETIME | | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성 시간 |
| `updated_at` | DATETIME | | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 수정 시간 |

### 2.8. bucket_list
-   **설명**: 사용자가 설정한 버킷리스트 항목을 관리하는 테이블입니다.
-   **비고**: `user_id`에 대한 외래키 제약조건이 설정되어 있습니다.

| 컬럼명 | 데이터 타입 | 키 | 제약 조건 | 설명 |
|---|---|---|---|---|
| `bucket_id` | BIGINT | PK | NOT NULL, AUTO_INCREMENT | 버킷리스트 고유 ID |
| `user_id` | BIGINT | FK | NOT NULL | 사용자 ID |
| `content` | TEXT | | NOT NULL | 버킷리스트 내용 |
| `is_completed` | TINYINT(1) | | NOT NULL, DEFAULT 0 | 완료 여부 |
| `created_at` | DATETIME | | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성 시간 |
| `updated_at` | DATETIME | | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 수정 시간 |

### 2.9. smalltalk_topic
-   **설명**: AI가 사용자의 데이터를 기반으로 생성한 스몰토크 주제를 관리하는 테이블입니다.
-   **비고**: `user_id`에 대한 외래키 제약조건이 설정되어 있습니다.

| 컬럼명 | 데이터 타입 | 키 | 제약 조건 | 설명 |
|---|---|---|---|---|
| `id` | BIGINT | PK | NOT NULL, AUTO_INCREMENT | 주제 고유 ID |
| `user_id` | BIGINT | FK | NOT NULL | 사용자 ID |
| `topic_type` | VARCHAR(50) | | NOT NULL | 주제 유형 |
| `topic_content` | VARCHAR(200) | | NOT NULL | AI가 생성한 주제 문장 |
| `example_question` | VARCHAR(255) | | NOT NULL | AI가 생성한 예시 질문 |
| `created_at` | DATETIME | | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성 시간 |

### 2.10. smalltalk_sources
-   **설명**: AI 스몰토크 주제가 생성된 원본 데이터(일정, 일기, 투두 등)와의 매핑 정보를 관리하는 테이블입니다.
-   **비고**: `smalltalk_id`에 대한 외래키 제약조건이 설정되어 있습니다.

| 컬럼명 | 데이터 타입 | 키 | 제약 조건 | 설명 |
|---|---|---|---|---|
| `id` | BIGINT | PK | NOT NULL, AUTO_INCREMENT | 매핑 고유 ID |
| `smalltalk_id` | BIGINT | FK | NOT NULL | 스몰토크 주제 ID |
| `source_type` | ENUM('SCHEDULE', 'DIARY', 'TODO', 'BUCKET')| | NOT NULL | 출처 테이블 구분 |
| `source_id` | BIGINT | | NOT NULL | 출처 테이블의 레코드 ID |
| `created_at` | DATETIME | | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성 시간 |

### 2.11. terms_and_policies
-   **설명**: 서비스 약관, 개인정보 처리방침 등 각종 정책의 내용을 관리하며, 버전별로 기록하는 마스터 테이블입니다.
-   **비고**: `type`과 `version`의 조합은 고유합니다.

| 컬럼명 | 데이터 타입 | 키 | 제약 조건 | 설명 |
|---|---|---|---|---|
| `id` | BIGINT | PK | NOT NULL, AUTO_INCREMENT | 약관 고유 ID |
| `type` | ENUM(...) | UNIQUE | NOT NULL | 약관 타입 |
| `version` | VARCHAR(20) | UNIQUE | NOT NULL | 약관 버전 |
| `title` | VARCHAR(200) | | NOT NULL | 약관 제목 |
| `content` | LONGTEXT | | NOT NULL | 약관 내용 |
| `effective_date` | DATETIME | | NOT NULL | 시행일 |
| `created_at` | DATETIME | | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성 시간 |