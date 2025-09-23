# Tropical 스몰토크 AI 활용 및 프롬프트 설계 문서

본 문서는 Tropical에서 AI를 활용하여 **스몰토크 주제를 생성**할 때 **어떤 역할/규칙/제약을 부여**하고, **무슨 데이터를 입력으로 주며**, **어떤 형식으로 응답을 받아 파싱**하여 **UI에 뿌리는지**를 기술합니다.
FE/BE/운영/QA가 공통 이해로 유지·개선할 수 있도록 작성되었습니다.

**작성자**: [진도희](https://github.com/dohee-jin)

**문서 버전**: v1.2

**대상 독자:**

* **백엔드/프론트엔드 개발자**: AI 호출 로직, 프롬프트 구조, 파싱 방식 이해 및 유지보수
* **QA 담당자**: 출력 형식 검증, 예외 케이스 테스트, 품질 기준 정의
* **기획/운영자**: AI가 어떤 규칙·제약으로 동작하는지 이해하여 서비스 기획/운영 반영
* **신규 합류자**: Tropical의 Smalltalk AI 활용 구조와 프롬프트 설계를 빠르게 이해하기 위한 온보딩 자료

---

## 1) 아키텍처 개요

### 진입점: `SmalltalkAIService`

* **스몰토크 주제 생성**: `generateSmallTalk(User user)`
  * 입력: `User` 객체
  * 출력: 생성된 주제들을 DB에 저장 후 처리 결과 반환
  * 스케줄러 지원: `generateTopicsForMultipleUsers(List<User> users)`로 배치 처리

### 데이터 흐름

1. **사용자 활동 데이터 수집** (`UserActivityService`)
   * 일정(SCHEDULE), 할일(TODO), 일기(DIARY), 버킷리스트(BUCKET) 조회
   * 사용자 동의 여부에 따라 데이터 포함/제외
   * 최근 7일간 데이터 기준으로 컨텍스트 구성

2. **AI 주제 생성 요청**
   * ChatClient를 통한 AI 모델 호출
   * JSON 형식으로 주제 리스트 응답 받음

3. **중복 방지 로직 적용**
   * 임베딩 벡터 생성 및 코사인 유사도 계산
   * 기존 주제와의 중복성 검사

4. **DB 저장 및 UI 제공**
   * 중복 검사 통과한 주제만 `SmalltalkTopic` 엔티티로 저장
   * `SmallTalkService`를 통해 UI에 최신 5개 주제 제공

---

## 2) AI 역할(Role)·규칙(Rules)·제약(Constraints)

### A. 역할(Role) 선언

```
당신은 Tropical 사용자를 위한 스몰토크 주제 추천 AI입니다.
사용자가 제공한 활동 데이터(SCHEDULE, TODO, DIARY, BUCKET)나 정보가 없으면 
일반적 관심사(GENERAL)를 참고하여, 사용자가 타인과 자연스럽게 나눌 수 있는 
대화형 질문 주제를 추천해야 합니다.
```

> 스몰토크 특화: 가벼운 일상 대화 주제 생성에 특화된 역할 정의

### B. 지시사항(Instructions)

대표 예시:

1. **상황 파악**: 시간대, 요일, 계절 등 고려
2. **관심사 반영**: 취미, 관심 분야, 최근 경험 기반
3. **데이터 활용**: 일정(SCHEDULE), 할일(TODO), 일기(DIARY), 버킷리스트(BUCKET) 활용
4. **복합 주제 생성**: 자연스럽게 연결되는 경우만
5. **데이터 없는 경우**: 보편적 주제로 추천
6. **편안한 분위기**: 부담 없는 가벼운 스몰토크
7. **대화형 의문문 강조**: exampleQuestion은 항상 타인과 나눌 수 있는 의문문 형태이며 20자 이내의 간결한 형태로 제공
8. **일반화**: 특정 사용자 전용 활동은 타인과 공유할 수 있도록 일반화
9. **소스 우선순위**: sources 배열은 가장 많이 참고한 소스부터 순서대로 나열

### C. 제약(Constraints)

* `topicContent`: 의미있는 주제 내용 (길이 제한 없음)
* `exampleQuestion`: ≤ 20자, 대화형 의문문
* `topicType`: COMPLEX, LIFESTYLE 등 카테고리 분류
* `sources`: 참고한 사용자 활동 데이터의 소스 정보

---

## 3) 입력 데이터 구성(Input Contract)

### 스몰토크 주제 생성 (`SmalltalkAIService.generateSmallTalk()`)

* **필수 입력**: `User` 객체
* **프롬프트 포함 정보**
  * 역할/지시/제약/출력형식(고정)
  * 사용자 활동 데이터: `TopicGenerateRequest` DTO 형태
  * 기존 주제 리스트 (중복 방지용)

### 활동 데이터 구성 (`UserActivityService.makeAIRequestDto()`)

```json
{
    "totalCount": 5,
    "activities": {
        "SCHEDULE": [
            {
                "id": 123,
                "title": "헬스장 운동",
                "content": "하체 운동 집중",
                "createdAt": "2024-09-15T10:00:00"
            }
        ],
        "TODO": [...],
        "DIARY": [...],
        "BUCKET": [...]
    }
}
```

* **데이터 범위**: 현재 날짜 기준 7일 전 ~ 현재
* **동의 기반**: 사용자가 동의한 항목만 포함
* **필수 포함**: SCHEDULE (일정은 항상 포함)

---

## 4) 출력 형식(Output Contract) & 파싱

### JSON 배열 형태 응답

```json
[
    {
        "topicType": "COMPLEX",
        "topicContent": "헬스장 운동과 영어 학습 병행",
        "exampleQuestion": "운동과 공부를 병행할 때 어떤 방법을 사용하시나요?",
        "sources": [
            { "sourceType": "SCHEDULE", "sourceId": 123 },
            { "sourceType": "TODO", "sourceId": 456 }
        ]
    },
    {
        "topicType": "LIFESTYLE",
        "topicContent": "카페 방문과 여행 계획",
        "exampleQuestion": "최근 방문한 카페나 여행에서 추천할 만한 경험이 있나요?",
        "sources": [
            { "sourceType": "DIARY", "sourceId": 789 },
            { "sourceType": "BUCKET", "sourceId": 101 }
        ]
    }
]
```

### 파싱 처리

```java
// ObjectMapper를 사용한 JSON 파싱
List<AISmallTalkResponse> responses = objectMapper.readValue(
    raw, new TypeReference<List<AISmallTalkResponse>>() {}
);
```

> **견고성 권장**: JSON 파싱 실패 시 로그 기록 후 RuntimeException 발생, 운영에서는 재시도 로직 필요

---

## 5) 중복 방지 로직 (Embedding + Cosine Similarity)

### 5.1 기본 원리

1. **기존 주제 수집**: DB에서 사용자의 기존 주제 조회
2. **임베딩 생성**: AI 생성 주제의 벡터 생성
3. **유사도 계산**: 코사인 유사도로 의미적 유사성 평가
4. **임계값 비교**: threshold 이상이면 저장 스킵

### 5.2 임베딩 생성 과정

```java
// 1. 주제 내용과 예시 질문 결합
List<String> topicContents = responses.stream()
    .map(res -> res.topicContent() + " " + res.exampleQuestion())
    .collect(Collectors.toList());

// 2. 배치 임베딩 생성
List<double[]> embeddings = batchEmbed(topicContents);

// 3. L2 정규화
double[] normalized = l2normalize(embedding);
```

### 5.3 코사인 유사도 계산

```java
private double cosineSimilarity(double[] vectorA, double[] vectorB) {
    double dotProduct = 0.0;
    double normA = 0.0;
    double normB = 0.0;
    
    for (int i = 0; i < vectorA.length; i++) {
        dotProduct += vectorA[i] * vectorB[i];
        normA += vectorA[i] * vectorA[i];
        normB += vectorB[i] * vectorB[i];
    }
    
    double denominator = Math.sqrt(normA) * Math.sqrt(normB);
    return denominator == 0.0 ? 0.0 : dotProduct / denominator;
}
```

### 5.4 임계값 설정

* **저장된 주제 < 10개**: threshold = 0.85
* **저장된 주제 ≥ 10개**: threshold = 0.9
* **최대 저장 개수**: 5개

---

## 6) 스케줄링 & 배치 처리

### 배치 스케줄러 (`SmallTalkScheduler`)

```java
@Scheduled(cron = "${schedules.cron.reward.publish:0 0 9 * * *}", zone = "Asia/Seoul")
public void generateDailyTopics() {
    // 모든 사용자 조회
    List<User> users = userReadService.getAllUser();
    
    // AI 서비스에 배치 처리 요청
    smalltalkAIService.generateTopicsForMultipleUsers(users);
}
```

* **기본 스케줄**: 매일 오전 9시 (Asia/Seoul)
* **동시 실행 방지**: `AtomicBoolean`으로 중복 실행 차단
* **예외 처리**: 개별 사용자 처리 실패 시에도 전체 배치 중단 방지

---

## 7) API 연계 요약

### 주요 엔드포인트

* **GET `/smalltalk/topics`** (추정)
  * 입력: `userId` (인증을 통한 사용자 식별)
  * 출력: `TopicResponse` - 최신 스몰토크 주제 5개

### 처리 흐름

1. **사용자 활동 확인** → 활동이 없으면 웰컴 주제 반환
2. **저장된 주제 조회** → 최신 5개 주제 반환
3. **DTO 매핑** → UI 친화적 형태로 변환

---

## 8) 성능/비용 최적화 전략

### 8.1 현재 최적화 사항

1. **배치 임베딩**: 개별 호출 대신 배치 처리 (batchSize = 64)
2. **L2 정규화**: 코사인 유사도 계산 안정화
3. **임계값 조정**: 기존 주제 수에 따른 동적 임계값

### 8.2 향후 개선 로드맵

1. **임베딩 캐시**: 기존 주제 임베딩을 캐시하여 재계산 방지
2. **활동 기반 사용자 필터링**: 활동이 있는 사용자만 배치 처리
3. **토큰 사용량 모니터링**: AI 호출 비용 추적 및 최적화
4. **실시간 생성**: 사용자 요청 시점 실시간 생성 옵션

---

## 9) 예외 처리 & 품질 가드

### 9.1 예외 상황별 처리

* **JSON 파싱 실패**: 로그 기록 후 RuntimeException 발생
* **임베딩 생성 실패**: 해당 주제 스킵 후 계속 진행
* **유사도 계산 오류**: 안전값(0.0) 반환
* **AI 호출 실패**: 전체 프로세스 중단, 로그 기록

### 9.2 품질 보장

* **입력 검증**: 사용자 존재 여부, 권한 확인
* **출력 검증**: JSON 구조 검증, 필수 필드 존재 확인
* **길이 제한**: exampleQuestion 20자 제한 준수
* **중복 방지**: 임베딩 기반 의미적 중복 차단

---

## 10) 테스트 관점 (샘플 케이스)

### 10.1 기능 테스트

* **신규 사용자**: 웰컴 주제 정상 반환
* **활동 있는 사용자**: 개인화된 주제 생성
* **중복 주제**: 유사도 임계값 기준 필터링
* **빈 응답**: AI 응답 없을 때 예외 처리

### 10.2 성능 테스트

* **배치 처리**: 대량 사용자 처리 시간 측정
* **임베딩 생성**: 벡터 생성 속도 및 메모리 사용량
* **DB 저장**: 대량 데이터 저장 성능

### 10.3 예외 상황 테스트

* **AI 서비스 다운**: 타임아웃 및 재시도 로직
* **DB 연결 실패**: 트랜잭션 롤백 확인
* **메모리 부족**: 대용량 임베딩 처리 시 메모리 관리

---

## 11) 프롬프트 원문 (현행)

### 시스템 템플릿

```
## ROLE & GOAL
당신은 Tropical 사용자를 위한 스몰토크 주제 추천 AI입니다.
사용자가 제공한 활동 데이터(SCHEDULE, TODO, DIARY, BUCKET)나 정보가 없으면 일반적 관심사(GENERAL)를 참고하여,
**사용자가 타인과 자연스럽게 나눌 수 있는 대화형 질문 주제를 추천**해야 합니다.
대화 주제는 요청한 개수만큼 제공하면 됩니다.

## 입력 데이터 형식
사용자 데이터는 다음 JSON 형식으로 제공됩니다:

**데이터가 있는 경우:**
{
    "totalCount": 5,
    "activities": {
        "SCHEDULE": [ ... ],
        "TODO": [ ... ],
        "DIARY": [ ... ],
        "BUCKET": [ ... ]
    }
}

## 기본 지침
1. 상황 파악: 시간대, 요일, 계절 등 고려
2. 관심사 반영: 취미, 관심 분야, 최근 경험 기반
3. 데이터 활용: 일정(SCHEDULE), 할일(TODO), 일기(DIARY), 버킷리스트(BUCKET) 활용
4. 복합 주제 생성: 자연스럽게 연결되는 경우만
5. 데이터 없는 경우: 보편적 주제로 추천
6. 편안한 분위기: 부담 없는 가벼운 스몰토크
7. 대화형 의문문 강조: exampleQuestion은 항상 타인과 나눌 수 있는 의문문 형태이며 20자 이내의 간결한 형태로 제공
8. 일반화: 특정 사용자 전용 활동은 타인과 공유할 수 있도록 일반화
9. 소스 우선순위: sources 배열은 가장 많이 참고한 소스부터 순서대로 나열

## 주제 카테고리
- 일상적: 오늘 하루, 영화/드라마, 음식, 날씨, 주말 계획
- 관심사: 취미, 배우고 있는 것, 음악/책, 여행 경험, 동물
- 생각거리: 재미있는 사실, 추억, 꿈/목표, 소소한 깨달음, 감사
- 창의적: "만약에…" 상상 질문, 딜레마/선택 질문, 미래 상상, 아이디어 나누기
- 복합 예시: 운동+건강식, 학습+여가, 여행+새 경험, 친구+취미 공유, 루틴+성장

## 응답 형식
**JSON 배열로 요청된 개수만큼 주제 반환, 다른 텍스트 금지**
```

---

## 12) 향후 변경에 강한 포인트

### 12.1 확장성 고려사항

* **프롬프트 템플릿 외부화**: 하드코딩 방지, 버전 관리 용이
* **임베딩 모델 교체 가능**: EmbeddingModel 인터페이스 활용
* **다양한 AI 모델 지원**: ChatClient 추상화 활용
* **스케줄링 설정 외부화**: `application.yml`을 통한 크론 설정

### 12.2 모니터링 & 운영

* **성능 메트릭**: AI 호출 시간, 임베딩 생성 시간, DB 저장 시간
* **품질 메트릭**: 중복 방지 효과, 사용자 만족도
* **비용 추적**: AI 모델 호출 횟수 및 토큰 사용량
* **오류 추적**: 예외 발생 빈도 및 원인 분석

### 12.3 사용자 경험 개선

* **개인화 강화**: 사용자 피드백 기반 주제 선호도 학습
* **실시간 업데이트**: 새로운 활동 발생 시 즉시 주제 갱신
* **다양성 보장**: 카테고리별 균형 잡힌 주제 분배
* **계절성 반영**: 시기별 맞춤 주제 생성