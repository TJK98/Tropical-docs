# 트러블슈팅: Holiday API 연도 파라미터 유효성 검증 최적화 및 설정 중앙화

본 문서는 **Tropical 백엔드**의 공휴일 조회 API 개발 과정에서 발생한 **미래 연도 조회 제한으로 인한 400 Bad Request 에러** 문제의 원인과 해결 과정을 정리한 기술 문서입니다.

경직된 매직 넘버와 분산된 유효성 검증 로직으로 인해 발생한 시스템 유연성 부족 문제를 해결하여 단일 진실 공급원 원칙을 준수하는 유지보수 가능한 설정 관리 체계를 구축하는 것을 목표로 합니다.

**작성자:** [왕택준](https://github.com/TJK98)

**작성일:** 2025년 9월 19일

**문서 버전:** v1.0

---

## 1. 문제 현상

### 1-1. 미래 연도 조회 시 API 요청 실패

* **증상**: 프론트엔드 캘린더에서 2030년으로 이동 시 공휴일 데이터 로딩 실패
* **HTTP 응답**: `400 Bad Request` 에러 발생
* **프론트엔드 에러 로그**:
  ```javascript
  GET http://localhost:9005/api/v1/holidays/monthly?year=2030&month=12 400 (Bad Request)
  월별 공휴일 조회 실패: Error: HTTP error! status: 400
      at Object.getMonthlyHolidays (holidayService.js:13:23)
      at async loadHolidays (Calendar.jsx:61:26)
  ```

### 1-2. 백엔드 유효성 검증 실패

* **증상**: 서버에서 연도 범위 초과로 요청 거부
* **백엔드 에러 로그**:
  ```
  WARN 12575 --- [nio-9005-exec-4] c.t.b.c.e.CalendarExceptionHandler: 
  캘린더 API 클라이언트 요청 오류 (400 Bad Request): 
  IllegalArgumentException - 연도는 1900–2027 사이여야 합니다. 입력값: 2030

  WARN 12575 --- [nio-9005-exec-4] .m.m.a.ExceptionHandlerExceptionResolver: 
  Resolved [java.lang.IllegalArgumentException: 연도는 1900–2027 사이여야 합니다. 입력값: 2030]
  ```

### 1-3. 설정 관리의 구조적 문제

```java
// 문제가 된 기존 설계 - 매직 넘버와 중복 설정
// HolidayService.java
private int dynamicMaxYear() {
    return Year.now(clock).getValue() + 2; // 매직 넘버 +2
}

// HolidayController.java  
@YearWithin(min = 1900, aheadYears = 2) // 동일한 정책 중복
        int year
```

### 1-4. 환경 정보

- **백엔드**: Spring Boot 3.5.5, Java 17
- **데이터베이스**: MariaDB
- **추가 라이브러리**: Spring WebFlux, Spring Data JPA, Spring Validation, Jackson, springdoc-openapi
- **외부 API**: 한국천문연구원(KASI) 특일 정보 API
- **테스트 환경**: Postman
- **브라우저**: Chrome 140+
- **운영체제**: Windows 11, macOS Sequoia

---

## 2. 원인 분석

### 2-1. 1단계: 매직 넘버로 인한 정책 불투명성 의심

**가설**: 코드 내 하드코딩된 `+2` 값이 정책 변경을 어렵게 만듦

```java
// 문제가 된 매직 넘버
private int dynamicMaxYear() {
    return Year.now(clock).getValue() + 2; // 왜 2인지 명확하지 않음
}
```

**결과**: 비즈니스 정책이 코드에 암묵적으로 숨겨져 있어 변경 의도 파악 어려움

### 2-2. 2단계: 설정 중복으로 인한 일관성 부족 의심

**가설**: Controller와 Service에서 동일한 정책을 각각 구현하여 불일치 위험 존재

```java
// Controller 레이어 검증
@YearWithin(min = 1900, aheadYears = 2) // +2년 제한

// Service 레이어 검증  
return Year.

now(clock).

getValue() +2; // 동일한 +2년 제한
```

**결과**: 동일한 비즈니스 규칙이 두 곳에서 중복 관리되어 수정 시 누락 위험 확인

### 2-3. 3단계: 유연성 부족으로 인한 확장성 제약 의심

**가설**: 현재 연도+2년으로 고정된 정책이 비즈니스 요구사항 변화에 대응 불가

```java
// 2025년 기준 최대 조회 가능 연도: 2027년
// 사용자가 2030년 조회 → 실패
GET /api/v1/holidays/monthly?year=2030 → 400
Bad Request
```

**결과**: 미래 계획 수립이나 장기 일정 관리 시 사용성 제약 확인

### 2-4. 4단계: 설정 변경 시 다중 파일 수정 필요성 의심

**가설**: 정책 변경 시 여러 파일을 동시에 수정해야 하는 유지보수 복잡성

```java
// 정책 변경 시 수정 필요한 파일들
1.HolidayService.java -

dynamicMaxYear() 메서드
2.HolidayController.java -@YearWithin 어노테이션
3.
에러 메시지
문자열 -
동적 생성
로직
// 하나라도 누락시 버그 발생
```

**결과**: 정책 변경 시 산발적으로 흩어진 코드 수정으로 인한 높은 실수 확률 확인

### 2-5. 5단계: 근본 원인 발견

**핵심 발견**: 단일 진실 공급원(Single Source of Truth) 원칙 위반

**구조적 문제들**:

- 동일한 비즈니스 정책이 여러 위치에 하드코딩
- 설정 변경 시 일관성을 보장할 수 있는 중앙 관리 체계 부재
- 매직 넘버로 인한 정책 의도 불투명성
- 계층 간 중복된 유효성 검증 로직

---

## 3. 디버깅 과정

### 3-1. 체계적 디버깅 방법론

**API 호출 플로우 추적**: 요청부터 응답까지 전체 처리 과정 분석

```java
// 1. Controller 레이어에서 유효성 검증
@GetMapping("/monthly")
public ResponseEntity<List<HolidayResponse>> getMonthlyHolidays(
        @RequestParam @YearWithin(min = 1900, aheadYears = 2) int year, // 첫 번째 검증
        @RequestParam @Min(1) @Max(12) int month) {

    log.debug("=== 월별 공휴일 조회 요청 ===");
    log.debug("요청 파라미터: year={}, month={}", year, month);

    // Service 호출
    List<HolidayResponse> holidays = holidayService.getMonthlyHolidays(year, month);

    return ResponseEntity.ok(holidays);
}
```

**유효성 검증 로직 상세 분석**: 각 계층별 검증 규칙 확인

```java
// YearWithinValidator.java - Controller 검증
@Override
public boolean isValid(Integer value, ConstraintValidatorContext context) {
    log.debug("=== Controller 연도 유효성 검증 ===");
    log.debug("입력값: {}", value);

    if (value == null) return true;

    int currentYear = Year.now(clock).getValue();
    int maxYear = currentYear + yearWithin.aheadYears(); // aheadYears = 2

    log.debug("현재 연도: {}, 최대 허용 연도: {}", currentYear, maxYear);

    boolean isValid = value >= yearWithin.min() && value <= maxYear;
    log.debug("검증 결과: {}", isValid);

    if (!isValid) {
        String message = String.format("연도는 %d–%d 사이여야 합니다. 입력값: %d",
                yearWithin.min(), maxYear, value);
        context.disableDefaultConstraintViolation();
        context.buildConstraintViolationWithTemplate(message).addConstraintViolation();
    }

    return isValid;
}
```

**Service 레이어 검증 로직 분석**: 중복 검증 확인

```java
// HolidayService.java - Service 검증
private void validateYear(int year) {
    log.debug("=== Service 연도 유효성 검증 ===");
    log.debug("입력값: {}", year);

    if (year < MIN_YEAR || year > dynamicMaxYear()) {
        String message = String.format("연도는 %d–%d 사이여야 합니다. 입력값: %d",
                MIN_YEAR, dynamicMaxYear(), year);
        log.warn("Service 계층 유효성 검증 실패: {}", message);
        throw new IllegalArgumentException(message);
    }
}

private int dynamicMaxYear() {
    int maxYear = Year.now(clock).getValue() + 2; // 매직 넘버
    log.debug("동적 최대 연도 계산: {}", maxYear);
    return maxYear;
}
```

**설정값 추적**: 동일한 정책이 적용되는 모든 위치 확인

```
설정 추적 결과:
1. HolidayController.java:47 - @YearWithin(aheadYears = 2)
2. HolidayService.java:156 - + 2 (매직 넘버)  
3. YearWithinValidator.java:23 - yearWithin.aheadYears() 사용

모든 곳에서 동일한 '+2' 정책 사용하지만 중앙 관리 체계 없음
```

---

## 4. 해결 과정

### 4-1. 실패한 해결책들

**A. 개별 파일에서 값만 수정**

```java
// 임시방편: HolidayService에서만 값 변경
private int dynamicMaxYear() {
    return Year.now(clock).getValue() + 5; // 2 → 5로 변경
}

// 하지만 Controller의 @YearWithin(aheadYears = 2)는 그대로 유지
```

**실패 이유**: Controller와 Service 간 검증 기준 불일치로 예측하지 못한 동작 발생

**B. 어노테이션 속성값만 변경**

```java
// Controller에서만 수정
@YearWithin(min = 1900, aheadYears = 5) // 2 → 5로 변경
```

**실패 이유**: Service 계층 검증 로직은 여전히 +2년 사용하여 일관성 부족

**C. 하드코딩된 상수로 통일 시도**

```java
// 각 파일에 동일한 상수 정의
// HolidayService.java
private static final int AHEAD_YEARS = 5;

// HolidayController.java  
private static final int AHEAD_YEARS = 5; // 중복 정의
```

**실패 이유**: 여전히 여러 곳에 분산된 정의로 단일 진실 공급원 원칙 위반

### 4-2. 최종 해결책: 중앙화된 설정 관리 체계 구축

**1. 단일 진실 공급원으로 TimeConfig 클래스 생성**

```java

@Configuration
public class TimeConfig {

    /**
     * 공휴일 조회 시 허용되는 최대 미래 연도의 범위 (현재 연도 + N년)
     *
     * 비즈니스 정책:
     * - 사용자가 장기적인 계획 수립을 위해 미래 공휴일 정보를 조회할 수 있도록 함
     * - 너무 먼 미래는 공휴일 정책 변경 가능성으로 인해 제한
     * - 현재 정책: 5년 후까지 조회 허용 (예: 2025년 기준 2030년까지)
     *
     * 변경 시 영향 범위:
     * - HolidayController의 @YearWithin 검증
     * - HolidayService의 validateYear() 검증  
     * - 에러 메시지의 동적 최대값 표시
     */
    public static final int HOLIDAY_AHEAD_YEARS = 5;

    /**
     * 시스템 전체에서 사용할 Clock 인스턴스
     * 테스트 시 시간을 조작할 수 있도록 Bean으로 관리
     */
    @Bean
    public Clock clock() {
        return Clock.systemDefaultZone();
    }
}
```

**2. YearWithinValidator를 중앙 설정 사용하도록 리팩토링**

```java

@Component
public class YearWithinValidator implements ConstraintValidator<YearWithin, Integer> {

    private final Clock clock;
    private int min;

    public YearWithinValidator(Clock clock) {
        this.clock = clock;
    }

    @Override
    public void initialize(YearWithin constraintAnnotation) {
        this.min = constraintAnnotation.min();
        // aheadYears 속성 제거하고 TimeConfig 사용으로 통일
    }

    @Override
    public boolean isValid(Integer value, ConstraintValidatorContext context) {
        if (value == null) return true;

        int currentYear = Year.now(clock).getValue();
        // 중앙 설정 사용: 어노테이션 속성 대신 TimeConfig 상수 사용
        int max = currentYear + TimeConfig.HOLIDAY_AHEAD_YEARS;

        boolean isValid = value >= min && value <= max;

        if (!isValid) {
            // 동적으로 생성되는 에러 메시지도 중앙 설정 기반
            String message = String.format("연도는 %d–%d 사이여야 합니다. 입력값: %d",
                    min, max, value);
            context.disableDefaultConstraintViolation();
            context.buildConstraintViolationWithTemplate(message)
                    .addConstraintViolation();
        }

        return isValid;
    }
}
```

**3. HolidayService를 중앙 설정 사용하도록 수정**

```java

@Service
public class HolidayService {

    private final Clock clock;
    private static final int MIN_YEAR = 1900;

    // 매직 넘버 제거하고 중앙 설정 사용
    private int dynamicMaxYear() {
        return Year.now(clock).getValue() + TimeConfig.HOLIDAY_AHEAD_YEARS;
    }

    private void validateYear(int year) {
        if (year < MIN_YEAR || year > dynamicMaxYear()) {
            // 에러 메시지도 중앙 설정 기반으로 일관성 확보
            String message = String.format("연도는 %d–%d 사이여야 합니다. 입력값: %d",
                    MIN_YEAR, dynamicMaxYear(), year);
            throw new IllegalArgumentException(message);
        }
    }
}
```

**4. Controller 어노테이션 단순화**

```java

@RestController
@RequestMapping("/api/v1/holidays")
public class HolidayController {

    @GetMapping("/monthly")
    @Operation(summary = "월별 공휴일 조회",
            description = "지정된 연도/월의 공휴일 목록을 반환합니다. (1900년 이상, 현재 연도+5년 이하)")
    public ResponseEntity<List<HolidayResponse>> getMonthlyHolidays(
            @Parameter(description = "조회할 연도", example = "2025")
            @RequestParam
            @YearWithin(min = 1900) // aheadYears 속성 제거로 중앙 설정에 위임
            int year,

            @Parameter(description = "조회할 월 (1-12)", example = "12")
            @RequestParam
            @Min(1) @Max(12)
            int month) {

        log.debug("월별 공휴일 조회 요청: year={}, month={}", year, month);

        List<HolidayResponse> holidays = holidayService.getMonthlyHolidays(year, month);

        log.debug("월별 공휴일 조회 완료: {}개", holidays.size());
        return ResponseEntity.ok(holidays);
    }
}
```

**5. YearWithin 어노테이션 간소화**

```java

@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = YearWithinValidator.class)
@Documented
public @interface YearWithin {

    String message() default "연도가 유효한 범위를 벗어났습니다.";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};

    /**
     * 허용되는 최소 연도
     */
    int min() default 1900;

    // aheadYears 속성 제거: TimeConfig.HOLIDAY_AHEAD_YEARS를 직접 사용하도록 변경
}
```

**성공 이유**:

- 단일 진실 공급원: 모든 연도 정책이 `TimeConfig.HOLIDAY_AHEAD_YEARS` 하나에서 관리
- 매직 넘버 제거: `+2` 같은 의미 불분명한 숫자를 명확한 상수명으로 대체
- 계층 간 일관성: Controller와 Service가 동일한 중앙 설정을 참조
- 유지보수성 향상: 정책 변경 시 한 곳만 수정하면 전체 시스템에 반영
- 확장성 확보: 기존 2년에서 5년으로 확장하여 사용성 개선

---

## 5. 테스트 검증

### 5-1. Postman을 통한 API 동작 검증

**2030년 조회 성공 테스트**:
```
GET http://localhost:9005/api/v1/holidays/monthly?year=2030&month=12

Response:
Status: 200 OK
Body: [
  {
    "date": "2030-12-25",
    "name": "크리스마스",
    "isHoliday": true
  }
]
```

**허용 범위 초과시 에러 테스트**:
```
GET http://localhost:9005/api/v1/holidays/monthly?year=2031&month=1

Response:
Status: 400 Bad Request
Body: {
  "error": "연도는 1900–2030 사이여야 합니다. 입력값: 2031"
}
```

### 5-2. Swagger 문서를 통한 파라미터 검증

**Swagger UI 확인 결과**:
- API 문서에서 year 파라미터 설명이 "조회할 연도 (1900 이상, 현재 연도+5년 이하)"로 자동 업데이트됨
- 실제 요청 시 범위 초과 연도 입력하면 즉시 400 에러 응답 확인

### 5-3. 서버 로그를 통한 설정 일관성 확인

**설정 중앙화 전후 로그 비교**:

**개선 전 로그** (2030년 요청 시):
```
WARN --- [nio-9005-exec-4] c.t.b.c.e.CalendarExceptionHandler: 
캘린더 API 클라이언트 요청 오류 (400 Bad Request): 
IllegalArgumentException - 연도는 1900–2027 사이여야 합니다. 입력값: 2030
```

**개선 후 로그** (2030년 요청 시):
```
DEBUG --- [nio-9005-exec-2] c.t.b.service.HolidayService: 
월별 공휴일 조회 요청: year=2030, month=12
DEBUG --- [nio-9005-exec-2] c.t.b.service.HolidayService: 
월별 공휴일 조회 완료: 1개
```

**2031년 요청 시 로그** (여전히 제한됨):
```
WARN --- [nio-9005-exec-3] c.t.b.c.e.CalendarExceptionHandler: 
캘린더 API 클라이언트 요청 오류 (400 Bad Request): 
IllegalArgumentException - 연도는 1900–2030 사이여야 합니다. 입력값: 2031
```

---

## 6. 성능 영향 분석

### 6-1. 설정 중앙화로 인한 성능 변화

**변경 전후 성능 비교**:
- 런타임 성능: 무시할 수 있는 수준 (상수 참조는 컴파일러 최적화)
- 메모리 사용: 중복 상수 제거로 미미한 개선
- 유지보수성: 정책 변경 시 수정 지점 감소로 개발 효율성 향상

### 6-2. API 응답 시간 측정

**Postman 응답 시간**:
- 2030년 조회: 평균 45-60ms (정상 범위)
- 범위 초과 요청: 평균 15-25ms (빠른 검증 실패)

---

## 7. 관련 이슈 및 예방책

### 7-1. 설정 관리 안티패턴

**위험한 패턴들**:
```java
// 매직 넘버 남용
return Year.now().getValue() + 2; // 2는 무엇을 의미하는가?

// 설정 중복과 불일치
@YearWithin(aheadYears = 2) // Controller
return currentYear + 5;     // Service - 불일치!
```

**안전한 패턴들**:
```java
// 중앙 집중식 설정 관리
public static final int HOLIDAY_AHEAD_YEARS = 5;

// 모든 계층에서 동일한 중앙 설정 참조
return Year.now(clock).getValue() + TimeConfig.HOLIDAY_AHEAD_YEARS;
```

### 7-2. 유지보수성 강화

**설정 중앙화 검증**:
- 비즈니스 규칙이 단일 위치에서 정의
- 매직 넘버가 명확한 상수명으로 대체 
- 설정 변경 시 영향 범위가 명확히 문서화
- 계층 간 검증 로직 일관성 보장

---

## 8. 결론 및 배운 점

### 8-1. 주요 성과

1. **사용자 경험 개선**: 2030년 조회 불가 문제 해결로 미래 계획 수립 지원
2. **설정 관리 체계화**: 매직 넘버 제거 및 단일 진실 공급원 원칙 적용  
3. **유지보수성 향상**: 정책 변경 시 한 곳만 수정하면 되는 구조 확립
4. **코드 품질 개선**: 의미 있는 상수명과 명확한 문서화로 가독성 증대

### 8-2. 기술적 학습

**단일 진실 공급원(SSOT)의 중요성**:
- 동일한 비즈니스 규칙이 여러 곳에 중복 정의되면 불일치 위험 증가
- 중앙 집중식 설정 관리로 일관성 보장 및 변경 영향도 최소화

**매직 넘버 제거의 효과**:
- `+2`, `+5` 같은 의미 불분명한 숫자를 `HOLIDAY_AHEAD_YEARS`로 명시
- 코드 리뷰 시 비즈니스 의도 파악 용이성 증대

### 8-3. 향후 개선 방향

**동적 설정 관리**:
- `application.yml`을 통한 외부 설정 파일 활용
- 관리자 API를 통한 실시간 정책 변경 기능

**모니터링 강화**:
- 검증 실패 빈도 모니터링
- 설정 변경 이력 관리

이러한 개선을 통해 **매직 넘버와 분산된 설정으로 인한 유지보수 문제**를 근본적으로 해결하고, **단일 진실 공급원 원칙을 준수하는 견고하고 확장 가능한 설정 관리 체계**를 구축할 수 있었습니다.
