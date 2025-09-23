# 트러블슈팅: ConsentType Enum 일관성 문제 해결

본 문서는 **Tropical 백엔드** 개발 과정에서 발생한 **ConsentType Enum의 PathVariable과 RequestBody 간 불일치** 문제의 원인과 해결 과정을 정리한 기술 문서입니다.

Spring Boot에서 enum 처리 방식이 JSON과 URL에서 다르게 동작하는 문제를 해결하여 API 일관성과 프론트엔드 개발자 경험을 개선하는 것을 목표로 합니다.

**작성자:** [왕택준](https://github.com/TJK98)

**작성일:** 2025년 9월 19일

**문서 버전:** v1.0

---

## 1. 문제 현상

### 1-1. API 호출 방식별 서로 다른 동작

* **RequestBody(JSON)**: `"consentType": "termsOfService"` (camelCase) → 정상 작동
* **PathVariable(URL)**: `/api/v1/terms/termsOfService` → 500 에러 발생

### 1-2. 에러 메시지

```
ERROR: Method parameter 'consentType': Failed to convert value of type 'java.lang.String' to required type 'ConsentType'
Failed to convert from type [java.lang.String] to type [@PathVariable ConsentType] for value [calendarPersonalization]
```

### 1-3. 기존 상황의 문제점

```bash
# URL에서만 대문자 상수명 사용 가능
GET /api/v1/terms/CALENDAR_PERSONALIZATION
GET /api/v1/terms/TERMS_OF_SERVICE

# JSON에서는 camelCase 사용
{
  "consentType": "calendarPersonalization"
}
```

**문제점**:

- 프론트엔드에서 URL과 JSON에 서로 다른 형식 사용해야 함
- 개발자가 매번 변환 로직 작성 필요
- API 일관성 부족으로 혼란 증가

---

## 2. 원인 분석

### 2-1. ConsentType Enum 구조

```java
public enum ConsentType {
    @JsonProperty("termsOfService")
    TERMS_OF_SERVICE("서비스 이용약관", true),

    @JsonProperty("privacyPolicy")
    PRIVACY_POLICY("개인정보처리방침", true),
}
```

### 2-2. Spring Boot Enum 처리 방식 차이점

**Jackson (JSON 처리)**:

- `@JsonProperty` 어노테이션을 인식
- camelCase로 매핑된 값으로 역직렬화 가능

**Spring MVC (PathVariable 처리)**:

- 기본적으로 `Enum.valueOf()` 사용
- enum 상수명(UPPER_CASE)만 인식 가능
- `@JsonProperty` 값을 인식하지 못함

### 2-3. 불일치 매핑 테이블

| Enum 상수          | @JsonProperty  | RequestBody | PathVariable |
|------------------|----------------|-------------|--------------|
| TERMS_OF_SERVICE | termsOfService | 동작          | 에러           |
| PRIVACY_POLICY   | privacyPolicy  | 동작          | 에러           |

---

## 3. 해결 과정

### 3-1. ConsentTypeConverter 구현

```java

@Component
public class ConsentTypeConverter implements Converter<String, ConsentType> {

    @Override
    public ConsentType convert(String source) {
        // 1차: 대문자 상수명으로 시도 (기존 호환성)
        try {
            return ConsentType.valueOf(source.toUpperCase());
        } catch (IllegalArgumentException e) {
            // 2차: camelCase 매핑으로 시도
            for (ConsentType type : ConsentType.values()) {
                if (getJsonPropertyValue(type).equals(source)) {
                    return type;
                }
            }
            throw new IllegalArgumentException("Invalid ConsentType: " + source);
        }
    }

    private String getJsonPropertyValue(ConsentType type) {
        return switch (type) {
            case TERMS_OF_SERVICE -> "termsOfService";
            case PRIVACY_POLICY -> "privacyPolicy";
            case CALENDAR_PERSONALIZATION -> "calendarPersonalization";
        };
    }
}
```

### 3-2. WebConfig에 컨버터 등록

```java

@Configuration
@RequiredArgsConstructor
public class WebConfig implements WebMvcConfigurer {

    private final ConsentTypeConverter consentTypeConverter;

    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(consentTypeConverter);
    }
}
```

### 3-3. Swagger 문서 일관성 확보

```java
// TermsController PathVariable 문서화
@Parameter(
        description = "동의 항목 타입",
        example = "termsOfService",
        schema = @Schema(allowableValues = {
                "termsOfService", "privacyPolicy", "calendarPersonalization"
        })
)
@PathVariable
ConsentType consentType

// TermsRequest 문서화
@Schema(
        description = "동의 항목 타입",
        example = "termsOfService",
        allowableValues = {
                "termsOfService", "privacyPolicy", "calendarPersonalization"
        }
)
private ConsentType consentType;
```

---

## 4. 테스트 검증

### 4-1. PathVariable 변환 테스트

```java

@Test
@DisplayName("PathVariable ConsentType 변환 테스트")
void testPathVariableConversion() {
    // camelCase 형식 테스트
    ConsentType result1 = converter.convert("termsOfService");
    assertThat(result1).isEqualTo(ConsentType.TERMS_OF_SERVICE);

    ConsentType result2 = converter.convert("calendarPersonalization");
    assertThat(result2).isEqualTo(ConsentType.CALENDAR_PERSONALIZATION);

    // 기존 UPPER_CASE 호환성 테스트
    ConsentType result3 = converter.convert("TERMS_OF_SERVICE");
    assertThat(result3).isEqualTo(ConsentType.TERMS_OF_SERVICE);
}
```

### 4-2. API 통합 테스트

```java

@Test
@DisplayName("PathVariable과 RequestBody 일관성 테스트")
void testApiConsistency() throws Exception {
    // PathVariable - camelCase
    mockMvc.perform(get("/api/v1/terms/termsOfService"))
            .andExpect(status().isOk());

    // RequestBody - camelCase
    TermsRequest request = new TermsRequest();
    request.setConsentType(ConsentType.TERMS_OF_SERVICE);

    mockMvc.perform(post("/api/v1/terms")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(request)))
            .andExpected(status().isOk());
}
```

---

## 5. 결과 및 개선사항

### 5-1. 변경 전후 비교

**변경 전 (문제 상황)**:

```bash
# PathVariable - 대문자만 가능
GET /api/v1/terms/CALENDAR_PERSONALIZATION  # 성공
GET /api/v1/terms/calendarPersonalization    # 500 에러

# RequestBody - camelCase만 가능  
{"consentType": "calendarPersonalization"}   # 성공
```

**변경 후 (개선된 상황)**:

```bash
# PathVariable - 양쪽 형식 모두 지원
GET /api/v1/terms/CALENDAR_PERSONALIZATION  # 성공 (기존 호환성)
GET /api/v1/terms/calendarPersonalization    # 성공 (새로 지원)
GET /api/v1/terms/termsOfService             # 성공 (새로 지원)

# RequestBody - 기존과 동일
{"consentType": "calendarPersonalization"}   # 성공
```

### 5-2. 프론트엔드 개발자 경험 개선

**변경 전**: 서로 다른 형식 사용 필요

```javascript
const urlFormat = 'CALENDAR_PERSONALIZATION';
const jsonFormat = 'calendarPersonalization';

fetch(`/api/v1/terms/${urlFormat}`);
fetch('/api/v1/terms', {
    body: JSON.stringify({consentType: jsonFormat})
});
```

**변경 후**: 일관된 형식 사용 가능

```javascript
const consentType = 'calendarPersonalization';

fetch(`/api/v1/terms/${consentType}`);
fetch('/api/v1/terms', {
    body: JSON.stringify({consentType})
});
```

---

## 6. 성능 및 호환성 분석

### 6-1. 변환 성능

- **1차 시도**: `valueOf()` 사용으로 O(1) 성능
- **2차 시도**: enum 순회로 O(n) 성능 (n은 enum 개수, 6개 내외)
- **전체적**: 무시할 수 있는 수준의 오버헤드

### 6-2. 하위 호환성

- 기존 UPPER_CASE 형식 완전 지원
- 신규 camelCase 형식 추가 지원
- Breaking Change 없음

### 6-3. 확장성

- 새로운 enum 값 추가 시 `getJsonPropertyValue()` 메서드만 수정
- 다른 enum 타입에도 동일한 패턴 적용 가능

---

## 7. 관련 이슈 및 예방책

### 7-1. enum PathVariable 사용 시 주의사항

**위험한 패턴들**:

```java
// @JsonProperty만 사용하고 컨버터 없이 PathVariable 사용
@GetMapping("/api/items/{itemType}")
public ResponseEntity<?> getItem(@PathVariable ItemType itemType) {
    // camelCase URL 접근 시 500 에러 발생
}
```

**안전한 패턴들**:

```java
// 커스텀 컨버터와 함께 사용
@Component
public class ItemTypeConverter implements Converter<String, ItemType> {
    // 변환 로직 구현
}

@GetMapping("/api/items/{itemType}")
public ResponseEntity<?> getItem(@PathVariable ItemType itemType) {
    // 다양한 형식으로 안전하게 접근 가능
}
```

### 7-2. API 설계 가이드라인

**enum 사용 시 체크리스트**:

- PathVariable로 사용할 enum은 커스텀 컨버터 구현
- JSON과 URL에서 일관된 형식 사용 가능하도록 설계
- Swagger 문서에 올바른 예시값 명시
- 하위 호환성을 위한 기존 형식 지원 유지

**코드 리뷰 체크포인트**:

- `@PathVariable`로 enum 사용 시 컨버터 존재 여부 확인
- `@JsonProperty`와 실제 PathVariable 예상값 일치 여부
- API 문서의 예시값이 실제 동작하는 형식인지 검증

---

## 8. 결론 및 배운 점

### 8-1. 주요 성과

1. **API 일관성 확보**: URL과 JSON에서 동일한 camelCase 형식 사용 가능
2. **개발자 경험 개선**: 프론트엔드에서 변환 로직 불필요
3. **하위 호환성 유지**: 기존 UPPER_CASE 형식도 지원
4. **문서화 개선**: Swagger에서 일관된 예시 제공

### 8-2. 기술적 학습

**Spring Boot enum 처리 메커니즘**:

- Jackson (JSON): `@JsonProperty` 어노테이션 우선 적용
- Spring MVC (PathVariable): 기본적으로 `Enum.valueOf()` 사용
- 해결책: 커스텀 `Converter` 구현으로 통합 처리

**API 설계 원칙**:

- 일관성: 동일한 리소스는 동일한 식별자 사용
- 호환성: 기존 API 사용자를 위한 하위 호환성 유지
- 문서화: Swagger에서 올바른 사용법 명시

### 8-3. 프로세스 개선

**체계적 문제 해결**:

1. 문제 현상 정확한 파악 (PathVariable vs RequestBody)
2. 근본 원인 분석 (Spring 처리 방식 차이)
3. 해결책 설계 (커스텀 컨버터 + 호환성 고려)
4. 테스트 검증 (기존/신규 형식 모두 테스트)

**설계 시 고려사항**:

- API 설계 초기에 JSON과 URL 형식 통일성 고려
- enum을 PathVariable로 사용할 때는 항상 컨버터 구현 검토
- 문서화에서 실제 사용 가능한 예시 정확히 명시

### 8-4. 장기적 개선 방향

**다른 enum 타입 적용**:

- 동일한 패턴을 다른 enum PathVariable에 적용
- 공통 컨버터 인터페이스 추출 검토

**API 표준화**:

- 프로젝트 전체 enum 처리 방식 표준화
- 신규 enum 추가 시 일관된 패턴 적용

이러한 해결 과정을 통해 **API의 일관성과 사용성**을 크게 개선할 수 있었으며, Spring Boot에서 enum 처리 시 발생할 수 있는 함정을 이해하고 체계적으로 해결하는 방법론을 확립했습니다.