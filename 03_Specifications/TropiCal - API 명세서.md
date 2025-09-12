# **API 명세서 - TropiCal (상세 버전)**

| 항목 | 내용 |
| :--- | :--- |
| **팀명** | Aicemelt |
| **프로젝트명** | WarmUpdate |
| **플랫폼명** | TropiCal |
| **문서 버전** | v1.0 |
| **작성자** | [신동준](https://github.com/sdj3959) |
| **최종 수정일**| 2025-09-12 |
| **Base URL** | `https://localhost:9005/api` |
| **인증 방식** | JWT Bearer Token (쿠키 전달) |
| **응답 형식** | JSON |

---

## **API 카테고리**

- [1. 회원 인증 관리 (Auth)](#1-회원-인증-관리-auth)
- [2. 사용자 프로필 및 설정 관리 (MyPage/Settings)](#2-사용자-프로필-및-설정-관리-mypagesettings)
- [3. 약관 관리 (Terms)](#3-약관-관리-terms)
- [4. 일정 관리 (Schedules)](#4-일정-관리-schedules)
- [5. 일기 관리 (Diaries)](#5-일기-관리-diaries)
- [6. 할 일(To-do) 관리 (Todos)](#6-할-일to-do-관리-todos)
- [7. 버킷리스트 관리 (Buckets)](#7-버킷리스트-관리-buckets)
- [8. AI 스몰토크 주제 관리 (AI Topics)](#8-ai-스몰토크-주제-관리-ai-topics)
- [9. 캘린더 공통 기능 (Calendar)](#9-캘린더-공통-기능-calendar)
- [10. 오류 처리](#10-오류-처리)

---

## **1. 회원 인증 관리 (Auth)**

| 메서드 | 경로 | 설명 | 담당자 | 인증 |
| :--- | :--- | :--- | :--- | :--- |
| `POST` | `/auth/signup` | 로컬 회원가입 | 왕택준 | ❌ |
| `POST` | `/auth/email/send-verification`| 이메일 인증 토큰(JWT) 발송 | 왕택준 | ❌ |
| `GET` | `/auth/email/verify` | 이메일 인증 토큰(JWT) 검증 | 왕택준 | ❌ |
| `POST` | `/auth/login` | 로컬 로그인 | 왕택준 | ❌ |
| `POST` | `/auth/logout` | 로그아웃 | 왕택준 | ✅ |
| `GET` | `/oauth2/authorization/{provider}`| 소셜 로그인 시작 (kakao, google, naver) | 왕택준 | ❌ |
| `POST` | `/auth/onboarding` | 신규 사용자 온보딩 완료 | 왕택준 | ✅ |
| `DELETE`| `/me/account` | 회원 탈퇴 (논리적 삭제) | 왕택준 | ✅ |

### **POST `/auth/signup`**

- **설명:** 이메일, 비밀번호, 닉네임으로 신규 계정을 생성 **(이메일 인증 전)**.
- **요청:**
  ```json
  {
    "email": "localuser@example.com",
    "password": "Password123!",
    "nickname": "로컬유저"
  }
  ```
- **응답 (201 Created):**
  ```json
  {
    "message": "인증 이메일이 발송되었습니다. 이메일을 확인해주세요."
  }
  ```

### **GET `/auth/email/verify?token={jwt}`**

- **설명:** 사용자가 이메일에서 인증 링크를 클릭하면 호출됩니다. 토큰 검증 후 계정 상태를 활성화합니다.
- **응답 (200 OK):**
  ```json
  {
    "message": "인증이 완료되었습니다. 로그인해주세요."
  }
  ```

### **POST `/auth/login`**

- **설명:** 로그인 성공 시 Access/Refresh 쿠키를 발급하고, 온보딩 완료 여부를 반환합니다.
- **요청:**
  ```json
  {
    "email": "localuser@example.com",
    "password": "Password123!"
  }
  ```
- **응답 (200 OK):**
  ```json
  {
    "onboardingCompleted": false,
    "user": {
      "id": 1,
      "nickname": "로컬유저"
    }
  }
  ```
  *(토큰은 HttpOnly, Secure 쿠키로 전달)*

### **POST `/auth/onboarding`**

- **설명:** 신규 가입자가 약관 동의를 완료했음을 서버에 알립니다.
- **요청:**
  ```json
  {
    "requiredConsents": {
      "TERMS_OF_SERVICE": true,
      "CALENDAR_PERSONALIZATION": true
    },
    "optionalConsents": {
      "DIARY_PERSONALIZATION": false,
      "TODO_PERSONALIZATION": true,
      "BUCKET_PERSONALIZATION": false
    }
  }
  ```
- **응답 (200 OK):**
  ```json
  {
    "message": "온보딩이 완료되었습니다."
  }
  ```

---

## **2. 사용자 프로필 및 설정 관리 (MyPage/Settings)**

| 메서드 | 경로 | 설명 | 담당자 | 인증 |
| :--- | :--- | :--- | :--- | :--- |
| `GET` | `/me/profile` | 내 프로필 및 설정 조회 | 왕택준 | ✅ |
| `PATCH`| `/me/profile` | 내 프로필 및 설정 수정 | 왕택준 | ✅ |
| `GET` | `/me/consents`| 내 동의 정보 조회 | 왕택준 | ✅ |
| `PATCH`| `/me/consents`| 내 선택 동의 정보 수정 | 왕택준 | ✅ |

### **GET `/me/profile`**

- **응답 (200 OK):**
  ```json
  {
    "email": "user@example.com",
    "nickname": "홍길동",
    "accountType": "SOCIAL", // "LOCAL" or "SOCIAL"
    "provider": "GOOGLE", // SOCIAL일 경우 "KAKAO", "GOOGLE", "NAVER"
    "weekStart": "MON", // "SUN" or "MON"
    "timezone": "Asia/Seoul",
    "showHolidays": true,
    "notifications": {
      "smalltalk": { "enabled": true, "time": "08:00", "days": "daily" },
      "schedule": { "enabled": true, "minutesBefore": 30 },
      "todo": { "enabled": true, "time": "08:00", "daysBefore": 1 }
    }
  }
  ```

### **PATCH `/me/profile`**

- **설명:** 수정할 필드만 요청 본문에 포함하여 전송합니다.
- **요청:**
  ```json
  {
    "nickname": "새로운닉네임",
    "showHolidays": false,
    "notifications": {
      "smalltalk": { "enabled": false }
    }
  }
  ```
- **응답 (200 OK):** 수정된 전체 프로필 정보 반환

### **GET `/me/consents`**

- **응답 (200 OK):**
  ```json
  {
    "requiredConsents": {
      "TERMS_OF_SERVICE": { "agreed": true, "agreedAt": "..." },
      "CALENDAR_PERSONALIZATION": { "agreed": true, "agreedAt": "..." }
    },
    "optionalConsents": {
      "DIARY_PERSONALIZATION": { "agreed": false, "agreedAt": null },
      ...
    }
  }
  ```

### **PATCH `/me/consents`**

- **요청:**
  ```json
  {
    "DIARY_PERSONALIZATION": true,
    "TODO_PERSONALIZATION": false
  }
  ```
- **응답 (200 OK):** 수정된 전체 동의 정보 반환

---

## **3. 약관 관리 (Terms)**

| 메서드 | 경로 | 설명 | 담당자 | 인증 |
| :--- | :--- | :--- | :--- | :--- |
| `GET` | `/terms/{type}` | 종류별 최신 약관 내용 조회 | 왕택준 | ❌ |

### **GET `/terms/{type}`**

- **경로 변수 `{type}`:** `service`, `privacy`, `calendar-personalization` 등
- **응답 (200 OK):**
  ```json
  {
    "type": "TERMS_OF_SERVICE",
    "version": "v1.0",
    "title": "서비스 이용약관",
    "content": "제1조 (목적) ...",
    "effectiveDate": "2025-01-01T00:00:00Z"
  }
  ```

---

## **4. 일정 관리 (Schedules)**

| 메서드 | 경로 | 설명 | 담당자 | 인증 |
| :--- | :--- | :--- | :--- | :--- |
| `POST` | `/schedules` | 새 일정 생성 | 신동준 | ✅ |
| `GET` | `/schedules/{scheduleId}` | 특정 일정 상세 조회 | 신동준 | ✅ |
| `PUT` | `/schedules/{scheduleId}` | 특정 일정 수정 | 신동준 | ✅ |
| `DELETE`| `/schedules/{scheduleId}` | 특정 일정 삭제 | 신동준 | ✅ |
| `PUT` | `/schedules/{scheduleId}/complete` | 일정 완료/미완료 처리 | 신동준 | ✅ |

### **POST `/schedules`**

- **요청:**
  ```json
  {
    "title": "Aicemelt 팀 주간 회의",
    "memo": "API 명세서 최종 검토",
    "scheduleDate": "2025-09-12",
    "startTime": "14:00",
    "endTime": "15:00",
    "location": "온라인 (Google Meet)",
    "attendees": "왕택준, 진도희, 백승현"
  }
  ```
- **응답 (201 Created):**
  ```json
  {
    "scheduleId": 1,
    "title": "Aicemelt 팀 주간 회의",
    "memo": "API 명세서 최종 검토",
    ...
    "isCompleted": false,
    "createdAt": "..."
  }
  ```

### **PUT `/schedules/{scheduleId}/complete`**

- **요청:**
  ```json
  {
    "isCompleted": true
  }
  ```- **응답 (200 OK):** 수정된 일정 정보 반환

---

## **5. 일기 관리 (Diaries)**

| 메서드 | 경로 | 설명 | 담당자 | 인증 |
| :--- | :--- | :--- | :--- | :--- |
| `POST` | `/diaries` | 새 일기 작성 | 신동준 | ✅ |
| `GET` | `/diaries/{diaryId}` | 특정 일기 상세 조회 | 신동준 | ✅ |
| `PUT` | `/diaries/{diaryId}` | 특정 일기 수정 | 신동준 | ✅ |
| `DELETE`| `/diaries/{diaryId}` | 특정 일기 삭제 | 신동준 | ✅ |

### **POST `/diaries`**

- **요청:**
  ```json
  {
    "title": "프로젝트 첫 기능 완성!",
    "content": "오늘 로그인 기능을 드디어 완성했다. 뿌듯하면서도 앞으로 할 일이 많아 긴장된다.",
    "emotion": "JOY", // JOY, SADNESS, ANGER, ...
    "weather": "SUNNY", // SUNNY, CLOUDY, RAINY, ...
    "diaryDate": "2025-09-11"
  }
  ```
- **응답 (201 Created):** 생성된 일기 정보 반환

---

## **6. 할 일(To-do) 관리 (Todos)**

| 메서드 | 경로 | 설명 | 담당자 | 인증 |
| :--- | :--- | :--- | :--- | :--- |
| `POST` | `/todos` | 새 할 일 생성 | 백승현 | ✅ |
| `GET` | `/todos` | 내 모든 할 일 조회 | 백승현 | ✅ |
| `PUT` | `/todos/{todoId}` | 특정 할 일 수정 | 백승현 | ✅ |
| `DELETE`| `/todos/{todoId}` | 특정 할 일 삭제 | 백승현 | ✅ |
| `PUT` | `/todos/{todoId}/complete`| 할 일 완료/미완료 처리 | 백승현 | ✅ |

### **GET `/todos`**

- **응답 (200 OK):**
  ```json
  [
    {
      "todoId": 1,
      "content": "API 명세서 작성",
      "dueDate": "2025-09-12",
      "isCompleted": true
    },
    {
      "todoId": 2,
      "content": "기능 개발 시작",
      "dueDate": null,
      "isCompleted": false
    }
  ]
  ```

---

## **7. 버킷리스트 관리 (Buckets)**

| 메서드 | 경로 | 설명 | 담당자 | 인증 |
| :--- | :--- | :--- | :--- | :--- |
| `POST` | `/buckets` | 새 버킷리스트 생성 | 백승현 | ✅ |
| `GET` | `/buckets` | 내 모든 버킷리스트 조회 | 백승현 | ✅ |
| `PUT` | `/buckets/{bucketId}` | 특정 버킷리스트 수정 | 백승현 | ✅ |
| `DELETE`| `/buckets/{bucketId}` | 특정 버킷리스트 삭제 | 백승현 | ✅ |
| `PUT` | `/buckets/{bucketId}/complete`| 버킷리스트 완료/미완료 처리| 백승현 | ✅ |

---

## **8. AI 스몰토크 주제 관리 (AI Topics)**

| 메서드 | 경로 | 설명 | 담당자 | 인증 |
| :--- | :--- | :--- | :--- | :--- |
| `POST` | `/ai/topics/generate` | 새 스몰토크 주제 생성 요청 | 진도희 | ✅ |
| `GET` | `/ai/topics` | 내 스몰토크 주제 목록 조회 | 진도희 | ✅ |

### **POST `/ai/topics/generate`**

- **설명:** 서버에 저장된 사용자의 최신 데이터를 기반으로 AI에게 새로운 주제 생성을 비동기적으로 요청합니다.
- **요청:**
  ```json
  {
    "contextRangeDays": 7 // 최근 7일 데이터만 사용
  }
  ```
- **응답 (202 Accepted):**
  ```json
  {
    "message": "주제 생성 요청이 접수되었습니다. 잠시 후 확인해주세요."
  }
  ```

### **GET `/ai/topics`**

- **설명:** 현재 사용자에게 추천된 최신 스몰토크 주제 5개를 조회합니다.
- **응답 (200 OK):**
  ```json
  {
   "topics": [
  {
    "id": 1,
    "topicType": "WORK",
    "topicContent": "주간 회의가 있으셨네요! 회의 전에 아이스브레이킹으로 이런 주제는 어떨까요?",
    "createdAt": "...",
    "example_question": "회의에서 가장 기억에 남는 아이스브레이킹 경험이 있으신가요?"
    },
  }

  ```

---

## **9. 캘린더 공통 기능 (Calendar)**

| 메서드 | 경로 | 설명 | 담당자 | 인증 |
| :--- | :--- | :--- | :--- | :--- |
| `GET` | `/calendar` | 월별 데이터 통합 조회 | 신동준 | ✅ |
| `GET` | `/calendar/holidays` | 공휴일 정보 조회 | 왕택준 | ❌ |

### **GET `/calendar?year={year}&month={month}`**

- **설명:** 특정 연월의 일정, 일기, 공휴일 정보를 통합하여 캘린더 뷰에 표시할 데이터를 조회합니다.
- **응답 (200 OK):**
  ```json
  {
    "schedules": [ { "scheduleId": 1, "title": "팀 회의", "scheduleDate": "2025-09-12", "startTime": "14:00", ... } ],
    "diaries": [ { "diaryId": 1, "diaryDate": "2025-09-11", "emotion": "JOY" } ],
    "holidays": [ { "name": "추석", "date": "2025-09-29" } ]
  }
  ```

---

## **10. 오류 처리**

- **클라이언트 오류 (4xx):**
  - `400 Bad Request`: 입력값 유효성 검증 실패 (비밀번호 정책 위반 등)
  - `401 Unauthorized`: 인증 실패 (유효하지 않은/만료된 토큰)
  - `403 Forbidden`: 권한 없음 (온보딩 미완료, 동의 누락, 이메일 미인증 등)
  - `404 Not Found`: 요청한 리소스 없음
  - `409 Conflict`: 데이터 중복 (닉네임 등)
- **서버 오류 (5xx):**
  - `500 Internal Server Error`
  - `503 Service Unavailable`

#### **오류 응답 형식**

```json
{
  "timestamp": "2025-09-12T10:30:00Z",
  "status": 400,
  "errorCode": "AUTH-202",
  "message": "유효하지 않은 이메일 인증 토큰입니다.",
  "path": "/api/auth/email/verify"
}
```