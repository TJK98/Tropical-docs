-- =============================================

-- TropiCal 플랫폼 MariaDB 스키마 (MVP - 문서 기반)

-- =============================================

```mariadb

-- 1. 사용자 기본 정보 테이블

CREATE TABLE user (

  -- 식별자 및 인증 관련

  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '사용자 고유 ID',

  email VARCHAR(255) NOT NULL COMMENT '이메일 주소 (모든 계정 타입에서 필수)',

  password_hash VARCHAR(255) NULL COMMENT '암호화된 비밀번호 (로컬 계정만 사용)',

  email_verified TINYINT(1) NOT NULL DEFAULT 0 COMMENT '이메일 인증 완료 여부',

  account_type ENUM('LOCAL','SOCIAL') NOT NULL COMMENT '계정 타입 (로컬/소셜)',



  -- 사용자 프로필 정보

  nickname VARCHAR(50) NOT NULL COMMENT '사용자 닉네임 (필수)',

  birth_date DATE NULL COMMENT '생년월일 (선택)',



  -- 회원가입 진행 상태

  onboarding_completed TINYINT(1) NOT NULL DEFAULT 0 COMMENT '온보딩 완료 여부',



  -- 시스템 관리용 정보

  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '계정 생성 시간',

  last_login_at DATETIME NULL COMMENT '마지막 로그인 시간',

  status ENUM('ACTIVE','BLOCKED','DELETED') NOT NULL DEFAULT 'ACTIVE' COMMENT '계정 상태',



  -- 캘린더 표시 개인 설정

  week_start ENUM('SUN','MON') NOT NULL DEFAULT 'MON' COMMENT '주 시작 요일',

  timezone VARCHAR(50) NOT NULL DEFAULT 'Asia/Seoul' COMMENT '시간대 설정',

  show_holidays TINYINT(1) NOT NULL DEFAULT 1 COMMENT '공휴일 표시 여부',



  -- 알림 설정

  smalltalk_notification_enabled TINYINT(1) NOT NULL DEFAULT 1 COMMENT '스몰 토크 알림 활성화',

  smalltalk_notification_time TIME NOT NULL DEFAULT '08:00:00' COMMENT '스몰 토크 알림 시간',

  smalltalk_notification_days VARCHAR(20) NOT NULL DEFAULT 'daily' COMMENT '알림 요일',



  schedule_notification_enabled TINYINT(1) NOT NULL DEFAULT 1 COMMENT '일정 알림 활성화',

  schedule_notification_minutes INT NOT NULL DEFAULT 30 COMMENT '일정 시작 전 알림 (분)',



  todo_notification_enabled TINYINT(1) NOT NULL DEFAULT 1 COMMENT '투두 마감 알림 활성화',

  todo_notification_time TIME NOT NULL DEFAULT '08:00:00' COMMENT '투두 마감 알림 시간',

  todo_notification_days_before INT NOT NULL DEFAULT 1 COMMENT '마감 며칠 전 알림',



  -- 인덱스 및 제약조건

  UNIQUE KEY uq_user_nickname (nickname)

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='사용자 기본 정보';



-- 2. 소셜 로그인 연동 정보 테이블

CREATE TABLE social_account (

  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '소셜 연동 정보 고유 ID',

  user_id BIGINT NOT NULL COMMENT '연결된 사용자 ID',

  provider ENUM('KAKAO','GOOGLE','NAVER') NOT NULL COMMENT '소셜 로그인 제공자',

  provider_user_id VARCHAR(255) NOT NULL COMMENT '소셜 제공자의 사용자 고유 ID',

  linked_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '소셜 계정 연결 시간',



  UNIQUE KEY uq_social (provider, provider_user_id),

  CONSTRAINT fk_social_user FOREIGN KEY (user_id) REFERENCES user(id) ON DELETE CASCADE

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='소셜 로그인 연동 정보';



-- 3. 사용자 동의 관리 테이블

CREATE TABLE user_consent (

  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '동의 정보 고유 ID',

  user_id BIGINT NOT NULL COMMENT '대상 사용자 ID',

  consent_type ENUM(

    'TERMS_OF_SERVICE',

    'CALENDAR_PERSONALIZATION',

    'DIARY_PERSONALIZATION',

    'TODO_PERSONALIZATION',

    'BUCKET_PERSONALIZATION'

  ) NOT NULL COMMENT '동의 항목 타입',

  agreed TINYINT(1) NOT NULL COMMENT '동의 여부',

  agreed_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '동의 처리 시간',



  UNIQUE KEY uq_user_consent (user_id, consent_type),

  CONSTRAINT fk_consent_user FOREIGN KEY (user_id) REFERENCES user(id) ON DELETE CASCADE

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='사용자 동의 관리';



-- 4. 공휴일 정보 테이블

CREATE TABLE holiday (

  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '공휴일 정보 고유 ID',

  country_code CHAR(2) NOT NULL DEFAULT 'KR' COMMENT '국가 코드',

  start_date DATE NOT NULL COMMENT '공휴일 시작 날짜',

  end_date DATE NOT NULL COMMENT '공휴일 종료 날짜',

  name_ko VARCHAR(100) NOT NULL COMMENT '공휴일 이름 (한국어)',

  is_substitute TINYINT(1) NOT NULL DEFAULT 0 COMMENT '대체공휴일 여부',

  year SMALLINT NOT NULL COMMENT '연도',



  UNIQUE KEY uq_holiday (country_code, start_date, end_date, name_ko),

  KEY idx_holiday_year (country_code, year, start_date)

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='공휴일 정보';



-- 5. 일정 테이블

CREATE TABLE schedule (

  schedule_id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '일정 고유 ID',

  user_id BIGINT NOT NULL COMMENT '사용자 ID',

  title VARCHAR(255) NOT NULL COMMENT '일정 제목',

  memo TEXT NULL COMMENT '일정 관련 메모',

  schedule_date DATE NOT NULL COMMENT '일정 날짜',

  start_time TIME NULL COMMENT '일정 시작 시간',

  end_time TIME NULL COMMENT '일정 종료 시간',

  location VARCHAR(100) NULL COMMENT '일정 장소',

  attendees VARCHAR(255) NULL COMMENT '참여자',

  is_completed TINYINT(1) NOT NULL DEFAULT 0 COMMENT '일정 완료 여부',

  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '생성 시간',

  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '수정 시간',



  CONSTRAINT fk_schedule_user FOREIGN KEY (user_id) REFERENCES user(id) ON DELETE CASCADE

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='일정 관리';



-- 6. 일기 테이블

CREATE TABLE diary (

  diary_id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '일기 고유 ID',

  user_id BIGINT NOT NULL COMMENT '사용자 ID',

  title VARCHAR(255) NOT NULL COMMENT '일기 제목',

  content TEXT NOT NULL COMMENT '일기 본문 내용',

  emotion VARCHAR(20) NOT NULL COMMENT '감정',

  weather VARCHAR(20) NOT NULL COMMENT '날씨',

  diary_date DATE NOT NULL COMMENT '일기 날짜',

  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '생성 시간',

  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '수정 시간',



  CONSTRAINT fk_diary_user FOREIGN KEY (user_id) REFERENCES user(id) ON DELETE CASCADE

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='일기 관리';



-- 7. TODO 리스트 테이블

CREATE TABLE todo (

  todo_id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT 'Todo 고유 ID',

  user_id BIGINT NOT NULL COMMENT '사용자 ID',

  content TEXT NOT NULL COMMENT '할 일 내용',

  due_date DATE NULL COMMENT '마감 기한',

  is_completed TINYINT(1) NOT NULL DEFAULT 0 COMMENT '완료 여부',

  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '생성 시간',

  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '수정 시간',



  CONSTRAINT fk_todo_user FOREIGN KEY (user_id) REFERENCES user(id) ON DELETE CASCADE

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='TODO 리스트';



-- 8. 버킷리스트 테이블

CREATE TABLE bucket_list (

  bucket_id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '버킷리스트 고유 ID',

  user_id BIGINT NOT NULL COMMENT '사용자 ID',

  content TEXT NOT NULL COMMENT '버킷리스트 내용',

  is_completed TINYINT(1) NOT NULL DEFAULT 0 COMMENT '완료 여부',

  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '생성 시간',

  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '수정 시간',



  CONSTRAINT fk_bucket_user FOREIGN KEY (user_id) REFERENCES user(id) ON DELETE CASCADE

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='버킷리스트';



-- 9. AI 스몰토크 주제 테이블

CREATE TABLE smalltalk_topic (

  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '주제 고유 ID',

  user_id BIGINT NOT NULL COMMENT '사용자 ID',

  topic_type VARCHAR(50) NOT NULL COMMENT '주제 유형',

  topic_content VARCHAR(200) NOT NULL COMMENT 'AI가 생성한 주제 문장',

  example_question VARCHAR(255) NOT NULL COMMENT 'AI가 생성한 예시 질문',

  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '생성 시간',



  CONSTRAINT fk_smalltalk_user FOREIGN KEY (user_id) REFERENCES user(id) ON DELETE CASCADE

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='AI 스몰토크 주제';



-- 10. 스몰토크 원본 데이터 출처 매핑 테이블

CREATE TABLE smalltalk_sources (

  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '매핑 고유 ID',

  smalltalk_id BIGINT NOT NULL COMMENT '스몰토크 주제 ID',

  source_type ENUM('SCHEDULE','DIARY','TODO','BUCKET') NOT NULL COMMENT '출처 테이블 구분',

  source_id BIGINT NOT NULL COMMENT '출처 테이블의 레코드 ID',

  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '생성 시간',



  CONSTRAINT fk_smalltalk_sources_topic FOREIGN KEY (smalltalk_id) REFERENCES smalltalk_topic(id) ON DELETE CASCADE

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='스몰토크 원본 데이터 출처 매핑';



-- 11. 약관 및 정책 관리 테이블

CREATE TABLE terms_and_policies (

  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '약관 고유 ID',

  type ENUM('TERMS_OF_SERVICE','PRIVACY_POLICY','CALENDAR_PERSONALIZATION','DIARY_PERSONALIZATION','TODO_PERSONALIZATION','BUCKET_PERSONALIZATION') NOT NULL COMMENT '약관 타입',

  version VARCHAR(20) NOT NULL COMMENT '약관 버전',

  title VARCHAR(200) NOT NULL COMMENT '약관 제목',

  content LONGTEXT NOT NULL COMMENT '약관 내용',

  effective_date DATETIME NOT NULL COMMENT '시행일',

  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '생성 시간',



  UNIQUE KEY uq_terms_type_version (type, version)

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='약관 및 정책 관리';

```



```mariadb

```