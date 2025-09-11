# 기능 명세서 — AI 스몰토크 주제 생성

**담당자: 진도희** 

**문서 버전:** v0.1

**최종 수정일:** 2025.09.09

---

## 1. 기능 개요

본 문서는 AI 모델(Gemini)을 이용하여 사용자 맞춤형 스몰토크 주제를 생성하고 애플리케이션에 통합하는 기능 명세를 정의합니다. DB에서 수집된 비정형 텍스트(일정, 일기 등)를 분석하여 즉시 활용 가능한 대화 주제를 제공하는 것을 목표로 합니다.

### 1.1 범위(Scope)

- MVP: DB에 저장된 사용자 텍스트 원문을 그대로 프롬프트에 포함하여 주제 생성

---

## 2. 시스템 구성 및 처리 흐름

1. 백엔드가 사용자 텍스트 데이터(일정 본문, 일기 등)를 수집합니다.
2. 프롬프트 생성 규칙에 따라 AI(Gemini) API 요청을 구성합니다.
3. AI가 JSON 형식으로 주제 5개를 반환합니다.
4. 백엔드는 응답을 검증/파싱하여 DB에 저장하고 캐시합니다.
5. 프론트엔드는 주제 목록을 조회/표시하며, 필요 시 "다른 주제 보기"를 요청합니다.(추후 업데이트)

---

## 3. 입력 데이터(백엔드 → AI)

- 사용자 ID 기준으로 수집된 텍스트 데이터 묶음
  - 일정(Schedule) 본문/제목
  - 일기(Diary) 본문/제목
  - 투두리스트(Todo) 내용/메모
  - 버킷리스트(Bucket) 내용/메모
- 데이터 전처리(MVP)
  - 개인정보 민감정보 제거(이메일/전화번호/식별자 마스킹)
  - 너무 긴 텍스트는 토큰 한도에 맞춰 샘플링/요약(선택)

---

## 4. 프롬프트 설계

- 지시문: JSON 포맷 요구를 명시하고, 각 항목의 키 이름을 고정합니다.
- 톤/가이드: 일상적이고 부담 없는 스몰토크, 중립적이고 배려 있는 어조
- 수량: 정확히 5개 항목

예시 프롬프트:

```
다음 데이터를 기반으로 스몰토크 주제 5개를 추천해 줘.
응답은 JSON 형식으로, 각 항목은 "topicType"(주제 유형)과 "topicContent"(사용자에게 제공할 문장) 키를 포함해야 해.

데이터: [사용자 DB에서 가져온 텍스트]
```

---

## 5. AI 응답 형식(JSON)

- 백엔드 파싱 기준이 되는 스키마입니다.
- 모든 응답은 아래 스키마를 만족해야 하며, 검증 실패 시 재시도/오류 처리합니다.

```json
{
  "topics": [
    {
      "topicType": "TRAVEL",
      "topicContent": "최근에 다녀온 여행지 어땠어? 다음에 가고 싶은 곳 있어?"
    }
    // ... 총 5개 항목
  ]
}
```

### 5.1 검증 규칙

- 최상위 키는 `topics`이며 배열 길이 = 5
- 각 항목은 `topicType`(ENUM 유사 카테고리 값)과 `topicContent`(string, 1~200자)를 포함
- `topicType` 예시: `TRAVEL`, `FOOD`, `WORK`, `HOBBY`, `WEATHER`, `SPORTS`, `ETC` (필요 시 확장)
- 중복 주제 최소화(가능하면 서로 다른 카테고리)

---

## 6. API 설계(백엔드)

### 6.1 주제 생성 요청 (백엔드 <=> AI) 

- Method/URL: `POST /api/ai/topics/generate`
- Request Body:
  - `userId`: number
  - `contextRangeDays`(선택): 최근 N일 데이터만 사용
- Response Body: 위 5장의 JSON 스키마 준수
- 처리:
  1. 사용자 텍스트 수집 → 전처리
  2. 프롬프트 구성 → Gemini 호출
  3. 응답 검증/파싱 → DB 저장 → 응답 반환


### 6.2 주제 조회 (프론트엔드 <=> 백엔드)

- Method/URL: `GET /api/ai/topics?userId={id}`
- Query: 
- Response Body: 항상 5개 반환
  - `topics`: [{ id, topicType, topicContent, createdAt }]
  - 메모: (추후 개발 예정) "다른 주제 추천" 기능도 5개를 반환하도록 구성 예정

---

## 7. 데이터베이스 테이블 정의

### 7.1. `smalltalk_topic` 테이블 (주제 정보)

이 테이블은 AI가 생성한 스몰토크 주제의 기본 정보를 저장합니다. 원본 데이터 참조(`source_type`, `source_id`)는 중간 매핑 테이블로 분리하여 본 테이블에서 제거했으며, `topic_type`은 애플리케이션 코드에서 Enum으로 관리할 수 있도록 DB에서는 `VARCHAR` 타입으로 지정했습니다.

| 컬럼명 | 데이터 타입 | 제약 조건 | 설명 |
| :--- | :--- | :--- | :--- |
| `id` | `BIGINT` | `PK`, `AUTO_INCREMENT` | 주제 고유 ID |
| `user_id` | `BIGINT` | `FK`, `NOT NULL` | 사용자 ID (`users.id` 참조) |
| `topic_type` | `VARCHAR(50)` | `NOT NULL` | 주제 유형 (예: `TRAVEL`, `FOOD`) |
| `topic_content` | `VARCHAR(200)` | `NOT NULL` | AI가 생성한 주제 문장 |
| `example_question` | `VARCHAR(255)` | `NOT NULL` | AI가 생성한 예시 질문 |
| `created_at` | `DATETIME` | `DEFAULT CURRENT_TIMESTAMP` | 생성 시각 |
- 관계: `FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE`

### 7.2. `smalltalk_sources` — 스몰토크 ↔ 원본 출처 매핑(중간 테이블)

| 컬럼명 | 데이터 타입 | 제약 조건 | 설명 |
| :--- | :--- | :--- | :--- |
| `id` | `BIGINT` | `PK`, `AUTO_INCREMENT` | 매핑 고유 ID |
| `smalltalk_id` | `BIGINT` | `FK`, `NOT NULL` | `smalltalk_topic.id` 참조 |
| `source_type` | `ENUM('SCHEDULE','DIARY','TODO','BUCKET')` | `NOT NULL` | 출처 테이블 구분 |
| `source_id` | `BIGINT` | `NOT NULL` | 출처 테이블의 레코드 ID |
| `created_at` | `DATETIME` | `DEFAULT CURRENT_TIMESTAMP` | 생성 시각 |

- 관계: `FOREIGN KEY (smalltalk_id) REFERENCES smalltalk_topic(id) ON DELETE CASCADE`

---

## 8. 예외 처리 및 재시도 정책

- 응답 스키마 불일치 → 1차 시스템 재프롬프트(포맷 수정 유도) → 실패 시 1회 재호출
- 모델 오류/타임아웃 → 지수 백오프(예: 2s, 4s) 최대 2회 재시도
- 전처리 실패/민감정보 검출 → 로그 기록 후 요청 차단, 사용자 데이터 정제 후 재시도 안내

---

## 9. 보안 및 개인정보

- 프롬프트에 포함되는 원문은 마스킹/필터링(이메일, 전화번호, 계좌 등) 후 전송
- 프롬프트/응답 전문은 평문 저장 금지, 필요 시 암호화 저장 또는 해시로 대체
- API 키/모델 설정은 서버 측 안전한 비밀 관리(환경변수/Secret Manager)

---

## 10. 성능/비용/모니터링

- 요청당 토큰 사용량/비용을 `smalltalks_logs`(선택) 또는 애플리케이션 로깅/APM(OpenTelemetry 등)으로 집계
- 모니터링 지표: 성공률, 재시도율, 평균 지연, 토큰/비용, 주제 클릭률(프론트 이벤트)

---

## 11. 프론트엔드 연동 가이드(요약)

- 첫 진입 시 `GET /ai/topics`로 최신 5개 조회
- UI 입력 방지(비속어/민감어)와 중복 주제 시각적 제거 권장

---



