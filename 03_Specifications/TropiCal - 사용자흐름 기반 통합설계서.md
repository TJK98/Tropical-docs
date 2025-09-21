# Tropical 사용자 흐름기반 통합 명세서

본 문서는 Tropical에서 사용자가 **회원가입 → 온보딩 → 캘린더 → 일정/다이어리 작성 → AI 감정 분석 → 소소한 대화**로 이어지는 전 과정을, **일반 시나리오**와 **AI 보조 시나리오** 두 갈래 모두에 대해 **UI 동작, 프론트 코드 흐름, API 계약, 상태 관리, 데이터 변화**까지 한 눈에 파악할 수 있도록 정리했습니다.

**작성자**: [백승현](https://github.com/sirosho)

**문서 버전:** v1.1

**최종 수정일:** 2025.09.21

**대상 독자:**
- **프론트엔드/백엔드 개발자**: 시스템 흐름 및 API 계약 이해, 구현/유지보수 참고
- **QA(테스터)**: 사용자 시나리오 기반 테스트 케이스 설계 참고
- **운영자/관리자**: 서비스 운영 중 발생할 수 있는 시나리오 파악 및 대응
- **신규 합류자**: Tropical 플랫폼 전체 흐름을 빠르게 이해하기 위한 학습 자료

---

## 목차

1. [공통 사전 지식](#공통-사전-지식)
2. [데이터 모델 개요](#데이터-모델-개요)
3. [시나리오 A: 일반 흐름](#시나리오-a-일반-흐름)
   * 회원가입 & 온보딩
   * 캘린더 & 일정 관리
   * 다이어리 작성
   * Todo & 버킷리스트
4. [시나리오 B: AI 보조 흐름](#시나리오-b-ai-보조-흐름)
   * AI 스몰토크 주제 추천
5. [프론트엔드 라우팅 & 가드 동작](#프론트엔드-라우팅--가드-동작)
6. [엔드포인트 카탈로그](#엔드포인트-카탈로그)
7. [AI 통합 & 상태 관리](#ai-통합--상태-관리)
8. [운영/보안/품질 체크리스트](#운영보안품질-체크리스트)
9. [부록 A: API 요청/응답 예시](#부록-a-api-요청응답-예시)
10. [부록 B: 상태 관리 스토어 설계](#부록-b-상태-관리-스토어-설계)

---

## 공통 사전 지식

* **인증**: 

    JWT(Bearer). 쿠키의 `ACCESS_TOKEN`에 저장.(15분 후 만료)

    토큰 만료시 `REFRESH_TOKEN` (14일 후 만료) 을 통해 토큰 재발급

* **상태 관리**:
    * Zustand 스토어
    * 전역: 인증/사용자
    * 기능별: 캘린더/다이어리/todo리스트/버킷리스트

* **AI**:
    * OpenAI ChatGPT: 스몰토크 주제 추천 (일정/일기/투두/버킷리스트 기반)


---

## 데이터 모델 개요

* `user`(사용자)
  - `id`, `email`, `nickname`
  - `account_type`, `onboarding_completed`
  - `week_start`, `timezone`, `show_holidays`
  - 알림 설정 (스몰토크, 일정, 투두)
* `social_account`(소셜 계정)
  - `id`, `user_id`
  - `provider`, `provider_user_id`
* `user_consent`(사용자 동의)
  - `id`, `user_id`
  - `consent_type`, `agreed`
* `schedule`(일정)
  - `schedule_id`, `user_id`
  - `title`, `memo`
  - `schedule_date`, `start_time`, `end_time`
  - `location`, `attendees`, `is_completed`
* `diary`(일기)
  - `diary_id`, `user_id`
  - `title`, `content`
  - `emotion`, `weather`, `diary_date`
* `todo`(할일)
  - `todo_id`, `user_id`
  - `content`, `due_date`, `is_completed`
* `bucket_list`(버킷리스트)
  - `bucket_id`, `user_id`
  - `content`, `is_completed`
* `smalltalk_topic`(스몰토크 주제)
  - `id`, `user_id`
  - `topic_type`, `topic_content`
  - `example_question`, `embedding`
* `smalltalk_sources`(주제 소스)
  - `id`, `smalltalk_id`
  - `source_type`, `source_id`
* `holiday`(공휴일)
  - `id`, `country_code`
  - `start_date`, `end_date`, `name_ko`
* `terms_and_policies`(약관 정책)
  - `id`, `type`, `version`
  - `title`, `content`, `effective_date`

---

## 시나리오 A: 일반 흐름

### 1) 회원가입 & 온보딩

#### UI
* `/welcome` → "회원가입" / 소셜 로그인 버튼
* 이메일/비밀번호/닉네임 입력
* 약관 동의
* 서비스 소개 슬라이드

#### 요청
`POST /api/v1/auth/signup`
```json
{
  "email": "user@example.com",
  "password": "Password123!",
  "nickname": "트로피컬",
  "requiredConsents": {
    "termsOfService": true,
    "privacyPolicy": true,
    "calendarPersonalization": true
  },
  "optionalConsents": {
    "diaryPersonalization": true,
    "todoPersonalization": false,
    "bucketPersonalization": true
  }
}
```

#### 응답
```json
{
  "message": "회원가입 성공 및 이메일 인증 메일 발송",
  "emailVerificationRequired": true
}
```

#### 상태 변화
```javascript
authStore: {
  isAuthenticated: true,
  user: { id, email, nickname }
}
```

### 2) 캘린더 & 일정 관리

#### UI 흐름
* `/calendar` 진입
  - 좌측: 스몰토크 추천
  - 우측: 캘린더 및 일정 추가
* 날짜 클릭 → 해당 날짜 선택
* 날짜 더블클릭 → 해당 날짜로 이동
* 일정 추가하기 버튼 → 일정 생성
* 일정 리스트 클릭 → 일정 상세 및 수정

#### 데이터 로딩

`GET /api/v1/schedules?year={year}&month={month}`

```json
[
  {
    "scheduleId": 1,
    "title": "병원 예약",
    "memo": "정기 검진",
    "scheduleDate": "2025-09-22",
    "startTime": "14:00",
    "endTime": "15:00",
    "location": "○○병원",
    "attendees": null,
    "isCompleted": false
  }
]
```


#### 일정 추가

`POST /api/v1/schedules`

```json
{
  "title": "운동",
  "memo": "헬스장",
  "scheduleDate": "2025-09-23",
  "startTime": "18:00",
  "endTime": "19:00",
  "location": "헬스장",
  "attendees": null
}
```

#### 상태 관리

- 일정 목록 상태 관리
- 선택된 날짜/일정 상태 관리
- 모달 상태 관리 (등록/수정/상세보기)
- 로딩/에러 상태 관리
- 일정 CRUD 액션 함수들


### 3) 일기 작성

#### [일기 작성]

* 날짜 클릭 → 해당 날짜 선택
* 날짜 더블클릭 → 해당 날짜로 이동
* 일기 쓰기 버튼 클릭
* 내용 작성
* 감정 선택
* 날씨 선택

#### [일기 저장]
`POST /api/v1/diaries`
```json
{
  "title": "오늘의 일기",
  "content": "오늘은 정말 좋은 하루였다...",
  "emotion": "행복",
  "weather": "맑음",
  "diaryDate": "2025-09-22"
}
```

#### [상태 관리]

날짜 별 일기

### 4) Todo 리스트

#### [Todo UI]

* Todo 헤더 선택
* 할 일 입력
* 마감일 설정
* 완료 체크

#### Todo 추가
`POST /api/v1/todos`
```json
{
  "content": "주간 보고서 작성",
  "dueDate": "2025-09-23"
}
```

#### 상태 관리
할 일 목록 및 완료 상태 관리



### 4) 버킷리스트

#### 버킷리스트 추가
`POST /api/v1/buckets`
```json
{
  "content": "제주도 여행"
}
```

#### 상태 관리
버킷리스트 목록 및 완료 상태 관리

---

## 시나리오 B: AI 보조 흐름

### 1) AI 스몰토크 주제 추천

#### 기능 개요
* 사용자의 일정, 일기, 투두리스트, 버킷리스트 데이터를 분석
* 일상 대화에서 활용할 수 있는 자연스러운 주제 추천
* 이미 추천된 주제와의 유사도 검사를 통한 중복 방지

#### UI 흐름
* AI가 사용자 데이터 기반으로 대화 주제 생성
* 주제별 예시 질문과 함께 표시

#### 데이터 수집 범위

* **일정**: 최근 7일간의 스케줄 데이터
* **일기(동의 필요)**: 최근 7일간의 일기 내용
* **투두(동의 필요)**: 최근 7일간의 할 일 목록
* **버킷리스트(동의 필요)**: 전체 버킷리스트 항목

#### 사용자 동의 기반 데이터 활용
```javascript
// 사용자가 동의한 개인화 항목만 AI 분석에 사용
consentTypes: {
  DIARY_PERSONALIZATION: boolean,
  TODO_PERSONALIZATION: boolean, 
  BUCKET_PERSONALIZATION: boolean
}
```

#### AI 주제 추천 요청
`POST /api/smalltalk/generate-topics`
```json
{
  "totalCount": 5,
  "activities": {
    "SCHEDULE": [
      {
        "id": 123,
        "title": "헬스장 운동",
        "memo": "하체 운동",
        "createdAt": "2025-09-22T18:00:00"
      }
    ],
    "DIARY": [
      {
        "id": 456,
        "title": "오늘의 기록",
        "content": "새로운 카페를 발견했다...",
        "createdAt": "2025-09-22T20:00:00"
      }
    ],
    "TODO": [...],
    "BUCKET": [...]
  }
}
```

#### AI 응답 형식
```json
[
  {
    "topicType": "COMPLEX",
    "topicContent": "헬스장 운동과 영어 학습 병행",
    "exampleQuestion": "운동과 공부를 병행할 때 어떤 방법을 사용하시나요?",
    "embedding": [0.12, 0.34, 0.56, ...],
    "sources": [
      { "sourceType": "SCHEDULE", "sourceId": 123 },
      { "sourceType": "TODO", "sourceId": 456 }
    ]
  },
  {
    "topicType": "LIFESTYLE", 
    "topicContent": "카페 방문과 여행 계획",
    "exampleQuestion": "최근 방문한 카페나 여행에서 추천할 만한 경험이 있나요?",
    "embedding": [0.98, 0.45, 0.32, ...],
    "sources": [
      { "sourceType": "DIARY", "sourceId": 789 },
      { "sourceType": "BUCKET", "sourceId": 101 }
    ]
  }
]
```

#### 중복 방지 로직
기존 주제와의 코사인 유사도 계산을 통한 중복 필터링 (임계값: 기존 주제 10개 미만시 0.9, 이상시 0.85)

```
1. 주제 생성 → 2. 임베딩 계산 → 3. 기존 주제들과 유사도 검사
→ 4. 임계값(0.85) 초과시 제외 → 5. 키워드 중복도 추가 검사
→ 6. 통과한 주제만 최종 선정 → 7. 요청 개수만큼 반환
```



#### [상태 관리]
스몰토크 주제 목록 관리 및 생성 요청 처리

#### [주제 카테고리]
* **일상적**: 오늘 하루, 영화/드라마, 음식, 날씨, 주말 계획
* **관심사**: 취미, 배우고 있는 것, 음악/책, 여행 경험, 동물
* **생각거리**: 재미있는 사실, 추억, 꿈/목표, 소소한 깨달음, 감사
* **창의적**: "만약에..." 상상 질문, 딜레마/선택 질문, 미래 상상, 아이디어 나누기
* **복합**: 운동+건강식, 학습+여가, 여행+새 경험, 친구+취미 공유, 루틴+성장

---

## 프론트엔드 라우팅 & 가드 동작

### 1) 인증 관련 라우팅

| 경로            | 컴포넌트         | 인증 필요 | 가드 동작                  |
|-----------------|-----------------|----------|---------------------------|
| `/welcome`      | Welcome         | X        | -                         |
| `/onboarding`   | Onboarding      | X        | -                         |
| `/calendar`     | Calendar        | O        | 토큰 유효성 검사          |
| `/diary`        | Diary           | O        | 토큰 유효성 검사          |
| `/todo`         | Todo            | O        | 토큰 유효성 검사          |
| `/bucket`       | Bucket          | O        | 토큰 유효성 검사          |
| `/chat`         | Chat            | O        | 토큰 유효성 검사          |

### 2) 가드 동작 상세

#### [인증 체크]
쿠키의 ACCESS_TOKEN 존재 여부 확인 후 라우팅

#### [토큰 만료 리프레시]
```javascript
const refreshToken = async () => {
  try {
    const response = await api.post('/auth/refresh', {
      token: cookieUtils.getToken()
    });
    cookieUtils.setToken(response.data.token);
  } catch (error) {
    console.error('토큰 갱신 실패', error);
    cookieUtils.removeToken();
    navigate('/welcome');
  }
};
```

---

## 엔드포인트 카탈로그

### 1) 인증 관련

| 엔드포인트 | 메서드 | 설명 | 인증 필요 |
|------------|--------|------|-----------|
| `/api/v1/auth/signup` | POST | 회원가입 | X |
| `/api/v1/auth/login` | POST | 로그인 | X |
| `/api/v1/auth/logout` | POST | 로그아웃 | O |
| `/api/v1/auth/verify` | GET | 이메일 인증 | X |
| `/api/v1/auth/verify/resend` | POST | 인증 메일 재발송 | X |
| `/api/v1/auth/token/refresh` | POST | 토큰 갱신 | X |
| `/api/v1/auth/onboarding` | POST | 소셜 온보딩 완료 | O |
| `/api/v1/auth/status` | GET | 인증 상태 조회 | O |

### 2) 사용자 관련

| 엔드포인트 | 메서드 | 설명 | 인증 필요 |
|------------|--------|------|-----------|
| `/api/v1/users/me/preferences` | GET | 사용자 선호 설정 조회 | O |
| `/api/v1/users/me/preferences` | PATCH | 사용자 선호 설정 수정 | O |

### 3) 기능별 API

| 엔드포인트 | 메서드 | 설명 | 인증 필요 |
|------------|--------|------|-----------|
| `/api/v1/schedules` | GET/POST | 일정 관리 | O |
| `/api/v1/schedules/{scheduleId}` | GET/PUT/DELETE | 일정 상세 관리 | O |
| `/api/v1/schedules/{scheduleId}/complete` | PUT | 일정 완료 토글 | O |
| `/api/v1/diaries` | POST | 일기 생성 | O |
| `/api/v1/diaries/{diaryId}` | GET/PUT/DELETE | 일기 관리 | O |
| `/api/v1/todos` | GET/POST | 할일 관리 | O |
| `/api/v1/todos/{todoId}` | GET/PUT/DELETE | 할일 상세 관리 | O |
| `/api/v1/todos/{todoId}/complete` | PUT | 할일 완료 토글 | O |
| `/api/v1/todos/incomplete` | GET | 미완료 할일 조회 | O |
| `/api/v1/todos/completed` | GET | 완료된 할일 조회 | O |
| `/api/v1/todos/overdue` | GET | 연체된 할일 조회 | O |
| `/api/v1/buckets` | GET/POST | 버킷리스트 관리 | O |
| `/api/v1/buckets/{bucketId}` | GET/PUT/DELETE | 버킷리스트 상세 관리 | O |
| `/api/v1/buckets/{bucketId}/complete` | PUT | 버킷리스트 완료 토글 | O |
| `/api/v1/buckets/incomplete` | GET | 미완료 버킷리스트 조회 | O |
| `/api/v1/buckets/completed` | GET | 완료된 버킷리스트 조회 | O |

### 4) 약관 및 공휴일 관련

| 엔드포인트 | 메서드 | 설명 | 인증 필요 |
|------------|--------|------|-----------|
| `/api/v1/terms` | GET | 활성 약관 조회 | X |
| `/api/v1/terms/{consentType}` | GET | 특정 약관 상세 조회 | X |
| `/api/v1/holidays/monthly` | GET | 월별 공휴일 조회 | X |
| `/api/v1/holidays/month-status` | GET | 월 단위 휴무일 상태 조회 | X |
| `/api/v1/holidays/events` | GET | 공휴일 배경 이벤트 조회 | X |
| `/api/v1/holidays/check` | GET | 특정 날짜 휴일 여부 확인 | X |

### 5) AI 관련

| 엔드포인트 | 메서드 | 설명 | 인증 필요 |
|------------|--------|------|-----------|
| `/api/v1/smalltalk/generate-topics` | POST | 스몰토크 주제 생성 | O |
| `/api/v1/smalltalk/topics` | GET | 사용자 주제 목록 조회 | O |

---


## 운영/보안/품질 체크리스트

### 1) 운영

-  **AI API 사용량 모니터링**: OpenAI API 호출 횟수 및 비용 추적

### 2) 보안

- **JWT 토큰 쿠키 보안**: HttpOnly, Secure, SameSite 속성 적용
- **HTTPS 적용**: 모든 통신 암호화
- **CORS 설정**: 허용된 도메인만 접근
- **XSS/CSRF 방지**: 입력 검증 및 토큰 보호
- **민감 데이터 암호화**: 비밀번호 해싱, 개인정보 보호
- **환경변수 관리**: 민감한 설정 정보 분리

### 3) 품질

- **단위 테스트**: 테스트 커버리지 80% 이상
- **통합 테스트**: API 엔드포인트별 테스트
- **E2E 테스트**: 주요 사용자 시나리오 자동 테스트
- **AI 품질 검증**: 스몰토크 주제 추천 정확도 평가

---

## 부록: 텍스트 다이어그램

```
[사용자 인증]
  → 회원가입/로그인
  → JWT 토큰 발급
  → 프로필 정보 로드

[캘린더/일정]
  → 날짜 선택
  → 일정 CRUD


[일기]
  → 날짜 선택
  → 감정 선택
  → 날씨 선택
  → 내용 작성
  → 저장

[Todo/버킷]
  → 목록 조회
  → 항목 추가/수정
  → 달성 상태 관리

[AI 스몰토크 주제 추천]
  → 개인화 동의 확인
  → 사용자 활동 데이터 수집
  → AI 주제 생성
  → 유사도 검사
  → 새 주제 저장
```