# **API 명세서 - TropiCal**

| 항목 | 내용 |
| :--- | :--- |
| **팀명** | Aicemelt |
| **프로젝트명** | WarmUpdate |
| **플랫폼명** | TropiCal |
| **문서 버전** | v1.1 |
| **작성자** | [신동준](https://github.com/sdj3959), [백승현](https://github.com/sirosho) |
| **최종 수정일**| 2025-09-20 |
| **Base URL** | `https://localhost:9005` |
| **인증 방식** | JWT Bearer Token (쿠키 전달) |
| **응답 형식** | JSON |



## 인증
모든 API 엔드포인트는 Bearer 토큰 인증이 필요합니다.

```
Authorization: Bearer {token}
```

## API 태그 분류

### 1. Authentication (인증/회원가입 API)
### 2. Todo (할 일 API)
### 3. Schedule (일정 API)
### 4. Diary (일기 API)
### 5. BucketList (버킷리스트 API)
### 6. User Preferences (사용자 선호 설정 통합 관리 API)
### 7. Terms (약관 및 동의서 조회 API)
### 8. Holiday (공휴일 조회 API)
### 9. Admin (관리자 API)

---

## 1. Authentication API

### 1.1 회원가입
**POST** `/api/v1/auth/signup`

로컬 계정 회원가입. 이메일과 비밀번호를 사용하는 로컬 계정을 생성하고 이메일 인증 메일을 발송합니다.

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "Password123!",
  "nickname": "홍길동",
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

**Responses:**
- `200`: 회원가입 성공 및 이메일 인증 대기
- `400`: 회원가입 실패 (이메일 중복, 필수 동의 누락 등)

### 1.2 로그인
**POST** `/api/v1/auth/login`

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

### 1.3 로그아웃
**POST** `/api/v1/auth/logout`

### 1.4 이메일 인증
**GET** `/api/v1/auth/verify`

**Query Parameters:**
- `token` (required): 인증 토큰

### 1.5 인증 메일 재발송
**POST** `/api/v1/auth/verify/resend`

### 1.6 토큰 갱신
**POST** `/api/v1/auth/token/refresh`

**Cookies:**
- `REFRESH_TOKEN`: 리프레시 토큰

### 1.7 소셜 온보딩 완료
**POST** `/api/v1/auth/onboarding`

**Request Body:**
```json
{
  "requiredConsents": {
    "termsOfService": true,
    "privacyPolicy": true
  },
  "optionalConsents": {
    "diaryPersonalization": true
  }
}
```

### 1.8 인증 상태 조회
**GET** `/api/v1/auth/status`

---

## 2. Todo API

### 2.1 할 일 목록 조회
**GET** `/api/v1/todos`

전체 할 일 목록을 반환합니다.

**Response:**
```json
[
  {
    "todoId": 1,
    "content": "회의 준비하기",
    "dueDate": "2025-09-25",
    "isCompleted": false,
    "createdAt": "2025-09-22T10:00:00",
    "updatedAt": "2025-09-22T10:00:00"
  },
  ...
]
```

### 2.2 할 일 생성
**POST** `/api/v1/todos`

**Request Body:**
```json
{
  "content": "새로운 할 일",
  "dueDate": "2025-09-25"
}
```

### 2.3 할 일 단건 조회
**GET** `/api/v1/todos/{todoId}`

**Path Parameters:**
- `todoId` (required): 할 일 ID

### 2.4 할 일 수정
**PUT** `/api/v1/todos/{todoId}`

**Path Parameters:**
- `todoId` (required): 할 일 ID

**Request Body:**
```json
{
  "content": "수정된 할 일",
  "dueDate": "2025-09-26"
}
```

### 2.5 할 일 삭제
**DELETE** `/api/v1/todos/{todoId}`

**Path Parameters:**
- `todoId` (required): 할 일 ID

### 2.6 할 일 완료/미완료 토글
**PUT** `/api/v1/todos/{todoId}/complete`

**Path Parameters:**
- `todoId` (required): 할 일 ID

**Request Body:**
```json
{
  "isCompleted": true
}
```

### 2.7 할 일 마감일 제거
**DELETE** `/api/v1/todos/{todoId}/due-date`

**Path Parameters:**
- `todoId` (required): 할 일 ID

### 2.8 미완료 할 일 조회
**GET** `/api/v1/todos/incomplete`

### 2.9 완료된 할 일 조회
**GET** `/api/v1/todos/completed`

### 2.10 연체된 할 일 조회
**GET** `/api/v1/todos/overdue`

---

## 3. Schedule API

### 3.1 일정 생성
**POST** `/api/v1/schedules`

**Request Body:**
```json
{
  "title": "회의",
  "memo": "프로젝트 회의",
  "scheduleDate": "2025-09-25",
  "startTime": "09:00",
  "endTime": "10:00",
  "location": "회의실 A",
  "attendees": "홍길동, 김철수"
}
```

### 3.2 일정 조회
**GET** `/api/v1/schedules/{scheduleId}`

**Path Parameters:**
- `scheduleId` (required): 일정 ID

**Response:**
```json
{
  "scheduleId": 1,
  "title": "회의",
  "memo": "프로젝트 회의",
  "scheduleDate": "2025-09-25",
  "startTime": "09:00",
  "endTime": "10:00",
  "location": "회의실 A",
  "attendees": "홍길동, 김철수",
  "isCompleted": false
}
```

### 3.3 일정 수정
**PUT** `/api/v1/schedules/{scheduleId}`

**Path Parameters:**
- `scheduleId` (required): 일정 ID

**Request Body:**
```json
{
  "title": "수정된 회의",
  "memo": "수정된 내용",
  "scheduleDate": "2025-09-25",
  "startTime": "10:00",
  "endTime": "11:00",
  "location": "회의실 B",
  "attendees": "홍길동"
}
```

### 3.4 일정 삭제
**DELETE** `/api/v1/schedules/{scheduleId}`

**Path Parameters:**
- `scheduleId` (required): 일정 ID

### 3.5 일정 완료 상태 토글
**PUT** `/api/v1/schedules/{scheduleId}/complete`

**Path Parameters:**
- `scheduleId` (required): 일정 ID

---

## 4. Diary API

### 4.1 일기 생성
**POST** `/api/v1/diaries`

**Request Body:**
```json
{
  "title": "오늘의 일기",
  "content": "오늘은 좋은 하루였다.",
  "emotion": "행복",
  "weather": "맑음",
  "diaryDate": "2025-09-22"
}
```

### 4.2 일기 조회
**GET** `/api/v1/diaries/{diaryId}`

**Path Parameters:**
- `diaryId` (required): 일기 ID

**Response:**
```json
{
  "diaryId": 1,
  "title": "오늘의 일기",
  "content": "오늘은 좋은 하루였다.",
  "emotion": "행복",
  "weather": "맑음",
  "diaryDate": "2025-09-22"
}
```

### 4.3 일기 수정
**PUT** `/api/v1/diaries/{diaryId}`

**Path Parameters:**
- `diaryId` (required): 일기 ID

**Request Body:**
```json
{
  "title": "수정된 일기",
  "content": "수정된 내용",
  "emotion": "평온",
  "weather": "흐림",
  "diaryDate": "2025-09-22"
}
```

### 4.4 일기 삭제
**DELETE** `/api/v1/diaries/{diaryId}`

**Path Parameters:**
- `diaryId` (required): 일기 ID

---

## 5. BucketList API

### 5.1 버킷리스트 목록 조회
**GET** `/api/v1/buckets`

**Response:**
```json
[
  {
    "bucketId": 1,
    "content": "세계여행 가기",
    "isCompleted": false,
    "createdAt": "2025-09-22T10:00:00",
    "updatedAt": "2025-09-22T10:00:00"
  }
]
```

### 5.2 버킷리스트 생성
**POST** `/api/v1/buckets`

**Request Body:**
```json
{
  "content": "새로운 버킷리스트"
}
```

### 5.3 버킷리스트 조회
**GET** `/api/v1/buckets/{bucketId}`

**Path Parameters:**
- `bucketId` (required): 버킷리스트 ID

### 5.4 버킷리스트 수정
**PUT** `/api/v1/buckets/{bucketId}`

**Path Parameters:**
- `bucketId` (required): 버킷리스트 ID

**Request Body:**
```json
{
  "content": "수정된 버킷리스트"
}
```

### 5.5 버킷리스트 삭제
**DELETE** `/api/v1/buckets/{bucketId}`

**Path Parameters:**
- `bucketId` (required): 버킷리스트 ID

### 5.6 버킷리스트 완료/미완료 토글
**PUT** `/api/v1/buckets/{bucketId}/complete`

**Path Parameters:**
- `bucketId` (required): 버킷리스트 ID

**Request Body:**
```json
{
  "isCompleted": true
}
```

### 5.7 미완료 버킷리스트 조회
**GET** `/api/v1/buckets/incomplete`

### 5.8 완료된 버킷리스트 조회
**GET** `/api/v1/buckets/completed`

---

## 6. User Preferences API

### 6.1 내 선호 설정 통합 조회
**GET** `/api/v1/users/me/preferences`

프로필 정보, 캘린더 설정, 알림 설정, 동의 상태를 포함한 사용자의 전체 개인화 설정을 반환합니다.

**Response:**
```json
{
  "nickname": "홍길동",
  "birthDate": "1990-05-15",
  "weekStart": "MON",
  "timezone": "Asia/Seoul",
  "showHolidays": true,
  "dateSystem": "SOLAR",
  "smalltalkNotificationEnabled": true,
  "smalltalkNotificationTime": "08:00:00",
  "smalltalkNotificationDays": "daily",
  "scheduleNotificationEnabled": true,
  "scheduleNotificationMinutes": 30,
  "todoNotificationEnabled": true,
  "todoNotificationTime": "08:00:00",
  "todoNotificationDaysBefore": 1,
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

### 6.2 내 선호 설정 통합 부분 수정
**PATCH** `/api/v1/users/me/preferences`

프로필, 캘린더, 알림 설정, 선택 동의 등 사용자의 개인화 설정을 부분적으로 수정합니다. null 값인 필드는 기존 설정을 유지합니다.

**Request Body:**
```json
{
  "nickname": "새로운닉네임",
  "birthDate": "1990-05-15",
  "weekStart": "MON",
  "timezone": "Asia/Seoul",
  "showHolidays": true,
  "dateSystem": "LUNAR",
  "smalltalkNotificationEnabled": true,
  "smalltalkNotificationTime": "08:00:00",
  "smalltalkNotificationDays": "daily",
  "scheduleNotificationEnabled": true,
  "scheduleNotificationMinutes": 30,
  "todoNotificationEnabled": true,
  "todoNotificationTime": "08:00:00",
  "todoNotificationDaysBefore": 1,
  "optionalConsents": {
    "diaryPersonalization": true,
    "todoPersonalization": false,
    "bucketPersonalization": true
  }
}
```

---

## 7. Terms API

### 7.1 활성 약관 조회
**GET** `/api/v1/terms`

쿼리 파라미터를 통해 필수/선택 약관을 구분 조회하거나 요약 정보만 가져올 수 있습니다.

**Query Parameters:**
- `type` (optional): 약관 타입 필터 (`ALL`, `REQUIRED`, `OPTIONAL`)
- `format` (optional): 응답 형식 (`DETAIL`, `SUMMARY`)

### 7.2 새 약관 버전 생성(배포)
**POST** `/api/v1/terms`

기존 활성 약관을 비활성화한 후, 새 버전을 활성으로 저장합니다.

**Request Body:**
```json
{
  "consentType": "termsOfService",
  "title": "서비스 이용약관",
  "content": "제1조 (목적)\n본 약관은...",
  "version": "1.2"
}
```

**Responses:**
- `201`: 약관 생성 성공
- `400`: 잘못된 요청 데이터

### 7.3 특정 약관 상세 조회
**GET** `/api/v1/terms/{consentType}`

지정된 동의 항목의 약관 내용을 상세 조회합니다. 마이페이지에서 약관 재확인 시 사용됩니다.

**Path Parameters:**
- `consentType` (required): 동의 타입 (`termsOfService`, `privacyPolicy`, `calendarPersonalization`, `diaryPersonalization`, `todoPersonalization`, `bucketPersonalization`)

**Response:**
```json
{
  "consentType": "termsOfService",
  "title": "서비스 이용약관",
  "content": "제1조 (목적)\n본 약관은...",
  "version": "1.2",
  "lastUpdated": "2025-09-17T10:30:00"
}
```

---

## 8. Holiday API

### 8.1 월별 공휴일 조회
**GET** `/api/v1/holidays/monthly`

캐시 우선 전략으로 지정된 월의 모든 공휴일, 기념일, 24절기 정보를 조회합니다.

**Query Parameters:**
- `year` (required): 조회할 연도 (1900 이상, 현재 연도 +5 이하)
- `month` (required): 조회할 월 (1-12)

**Response:**
```json
[
  {
    "countryCode": "KR",
    "date": "2025-01-01",
    "startDate": "2025-01-01",
    "endDate": "2025-01-01",
    "nameKo": "신정",
    "holidayType": "PUBLIC_HOLIDAY",
    "isSubstitute": false
  }
]
```

**Responses:**
- `200`: 조회 성공
- `400`: 잘못된 요청 파라미터
- `500`: 서버 내부 오류

### 8.2 월 단위 휴무일 상태 조회
**GET** `/api/v1/holidays/month-status`

지정된 월의 각 날짜에 대해 주말/공휴일/전체 휴무일 여부와 공휴일명을 반환합니다. 캘린더 UI 구현에 유용합니다.

**Query Parameters:**
- `year` (required): 조회할 연도
- `month` (required): 조회할 월 (1-12)

**Response:**
```json
[
  {
    "date": "2025-01-01",
    "isWeekend": false,
    "isHoliday": true,
    "isDayOff": true,
    "holidayName": "신정"
  }
]
```

### 8.3 공휴일 배경 이벤트 조회
**GET** `/api/v1/holidays/events`

FullCalendar 배경 이벤트로 표시할 공휴일 데이터를 반환합니다. end는 미포함(exclusive) 처리됩니다.

**Query Parameters:**
- `start` (required): 시작 날짜 (포함, yyyy-MM-dd)
- `end` (required): 종료 날짜 (미포함, yyyy-MM-dd)

**Response:**
```json
[
  {
    "start": "2025-01-01",
    "end": "2025-01-02",
    "title": "신정",
    "display": "background",
    "className": "fc-holiday-bg",
    "allDay": true
  }
]
```

### 8.4 특정 날짜 휴일 여부 확인
**GET** `/api/v1/holidays/check`

해당 날짜가 실제 휴무일(법정공휴일, 국경일, 대체공휴일)인지 여부를 반환합니다. 기념일이나 24절기는 제외됩니다.

**Query Parameters:**
- `date` (required): 확인할 날짜 (yyyy-MM-dd)

**Response:**
```json
{
  "date": "2025-01-01",
  "isHoliday": true,
  "holidayName": "신정"
}
```

---

## 9. Admin API

### 9.1 사용자 목록 조회
**GET** `/api/v1/admin/users`

### 9.2 대시보드 조회
**GET** `/api/v1/admin/dashboard`

### 9.3 핑 테스트
**GET** `/api/v1/admin/ping`

---

## 10. Test API

### 10.1 API 테스트
**GET** `/api/test`

### 10.2 헬스 체크
**GET** `/api/health`

### 10.3 홈
**GET** `/`

---

## 공통 응답 형식

### 성공 응답
- `200 OK`: 요청 성공
- `201 Created`: 리소스 생성 성공

### 오류 응답
- `400 Bad Request`: 잘못된 요청
- `401 Unauthorized`: 인증 실패
- `403 Forbidden`: 권한 없음
- `404 Not Found`: 리소스 없음
- `500 Internal Server Error`: 서버 내부 오류

## 데이터 타입 및 제약사항

### 공통 제약사항
- 이메일: 최대 255자, 이메일 형식
- 비밀번호: 8-20자, 대소문자, 숫자, 특수문자 포함
- 닉네임: 2-50자
- 제목: 최대 255자
- 메모/내용: 최대 1000자
- 위치: 최대 100자
- 참석자: 최대 255자

### 날짜/시간 형식
- 날짜: `yyyy-MM-dd` (예: `2025-09-22`)
- 시간: `HH:mm:ss` (예: `14:30:00`)
- 일시: ISO 8601 형식 (예: `2025-09-22T14:30:00`)

### 열거형(Enum) 값

#### 약관 타입 (ConsentType)
- `termsOfService`: 서비스 이용약관
- `privacyPolicy`: 개인정보처리방침
- `calendarPersonalization`: 캘린더 개인화
- `diaryPersonalization`: 일기 개인화
- `todoPersonalization`: 할 일 개인화
- `bucketPersonalization`: 버킷리스트 개인화

#### 공휴일 타입 (HolidayType)
- `PUBLIC_HOLIDAY`: 법정공휴일
- `NATIONAL_HOLIDAY`: 국경일
- `SUBSTITUTE_HOLIDAY`: 대체공휴일
- `MEMORIAL_DAY`: 기념일
- `SEASONAL_DIVISION`: 24절기
- `TRADITIONAL_DAY`: 전통 기념일

#### 주 시작 요일 (WeekStart)
- `SUN`: 일요일
- `MON`: 월요일

#### 날짜 체계 (DateSystem)
- `SOLAR`: 양력
- `LUNAR`: 음력