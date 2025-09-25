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

**6. 단위 테스트로 설정 일관성 검증**

```java

@ExtendWith(MockitoExtension.class)
class TimeConfigIntegrationTest {

    @Mock
    private Clock mockClock;

    @Test
    void 중앙_설정이_모든_계층에서_일관되게_적용됨() {
        // Given: 2025년 1월 1일로 시간 고정
        when(mockClock.instant()).thenReturn(Instant.parse("2025-01-01T00:00:00Z"));
        when(mockClock.getZone()).thenReturn(ZoneOffset.UTC);

        // When: YearWithinValidator와 HolidayService의 최대 연도 계산
        YearWithinValidator validator = new YearWithinValidator(mockClock);
        HolidayService service = new HolidayService(mockClock, holidayRepository);

        int validatorMaxYear = Year.now(mockClock).getValue() + TimeConfig.HOLIDAY_AHEAD_YEARS;
        int serviceMaxYear = Year.now(mockClock).getValue() + TimeConfig.HOLIDAY_AHEAD_YEARS;

        // Then: 두 계층에서 동일한 최대 연도 사용
        assertThat(validatorMaxYear).isEqualTo(serviceMaxYear);
        assertThat(validatorMaxYear).isEqualTo(2030); // 2025 + 5
    }

    @Test
    void TimeConfig_상수_변경시_전체_시스템에_반영됨() {
        // TimeConfig.HOLIDAY_AHEAD_YEARS를 다른 값으로 변경했을 때
        // 모든 의존하는 코드가 새로운 값을 사용하는지 검증

        // 이 테스트는 컴파일 타임에 상수 값이 인라인되므로
        // 실제 운영에서는 리플렉션이나 설정 파일을 통한 동적 로딩 고려 필요
    }
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

### 5-1. 미래 연도 조회 성공 테스트

```java

@SpringBootTest
@AutoConfigureTestDatabase
class HolidayApiIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void 미래_연도_조회_성공_2030년() {
        // Given: 2030년 12월 공휴일 조회 요청 (중앙 설정으로 허용됨)
        String url = "/api/v1/holidays/monthly?year=2030&month=12";

        // When: API 호출
        ResponseEntity<List> response = restTemplate.getForEntity(url, List.class);

        // Then: 정상 응답 (이전에는 400 에러)
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody()).isInstanceOf(List.class);
    }

    @Test
    void 허용_범위_초과시_동적_에러_메시지() {
        // Given: 허용 범위 초과 연도 (2031년, 2025년 기준 +6년)
        String url = "/api/v1/holidays/monthly?year=2031&month=1";

        // When: API 호출
        ResponseEntity<String> response = restTemplate.getForEntity(url, String.class);

        // Then: 400 에러와 동적으로 생성된 에러 메시지
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
        assertThat(response.getBody()).contains("연도는 1900–2030 사이여야 합니다. 입력값: 2031");
    }
}
```

### 5-2. 설정 일관성 테스트

```java

@ExtendWith(SpringExtension.class)
@MockBean({HolidayRepository.class})
class SettingConsistencyTest {

    @MockBean
    private Clock mockClock;

    @Test
    void Controller와_Service의_연도_검증_일관성() {
        // Given: 2025년 고정
        when(mockClock.instant()).thenReturn(Instant.parse("2025-01-01T00:00:00Z"));
        when(mockClock.getZone()).thenReturn(ZoneOffset.UTC);

        // When: Controller 검증기와 Service 검증 로직 모두 테스트
        YearWithinValidator validator = new YearWithinValidator(mockClock);
        HolidayService service = new HolidayService(mockClock, holidayRepository);

        // Then: 경계값 테스트 - 일관된 동작 확인
        assertValidationConsistency(validator, service, 2030, true);   // 허용 (2025 + 5)
        assertValidationConsistency(validator, service, 2031, false);  // 거부 (2025 + 6)
        assertValidationConsistency(validator, service, 1900, true);   // 허용 (최소값)
        assertValidationConsistency(validator, service, 1899, false);  // 거부 (최소값 미만)
    }

    private void assertValidationConsistency(YearWithinValidator validator,
                                             HolidayService service,
                                             int year,
                                             boolean expected) {
        // Controller 계층 검증 결과
        boolean validatorResult = validator.isValid(year, mock(ConstraintValidatorContext.class));

        // Service 계층 검증 결과 (예외 발생 여부로 판단)
        boolean serviceResult;
        try {
            service.validateYear(year); // private 메서드는 리플렉션으로 호출
            serviceResult = true;
        } catch (IllegalArgumentException e) {
            serviceResult = false;
        }

        // 두 계층의 검증 결과가 일치해야 함
        assertThat(validatorResult).isEqualTo(expected);
        assertThat(serviceResult).isEqualTo(expected);
        assertThat(validatorResult).isEqualTo(serviceResult);
    }
}
```

### 5-3. 동적 에러 메시지 테스트

```java

@Test
void 시간_변화에_따른_동적_에러_메시지() {
    // 2025년과 2026년에 따른 에러 메시지 변화 확인

    // Given: 2025년으로 설정
    when(mockClock.instant()).thenReturn(Instant.parse("2025-01-01T00:00:00Z"));
    when(mockClock.getZone()).thenReturn(ZoneOffset.UTC);

    // When: 범위 초과 연도로 검증
    YearWithinValidator validator2025 = new YearWithinValidator(mockClock);
    ConstraintValidatorContext context = mock(ConstraintValidatorContext.class);
    ConstraintViolationBuilder builder = mock(ConstraintViolationBuilder.class);

    when(context.buildConstraintViolationWithTemplate(anyString())).thenReturn(builder);
    when(builder.addConstraintViolation()).thenReturn(context);

    validator2025.isValid(2031, context);

    // Then: 2025년 기준 에러 메시지 확인
    verify(context).buildConstraintViolationWithTemplate("연도는 1900–2030 사이여야 합니다. 입력값: 2031");

    // Given: 시간을 2026년으로 변경
    when(mockClock.instant()).thenReturn(Instant.parse("2026-01-01T00:00:00Z"));

    YearWithinValidator validator2026 = new YearWithinValidator(mockClock);
    validator2026.isValid(2032, context);

    // Then: 2026년 기준 에러 메시지로 동적 변경
    verify(context).buildConstraintViolationWithTemplate("연도는 1900–2031 사이여야 합니다. 입력값: 2032");
}
```

---

## 6. 성능 영향 분석

### 6-1. 설정 중앙화로 인한 성능 변화

**변경 전**:

- 각 클래스에서 독립적으로 상수 사용 (컴파일 타임 인라인)
- 메모리 사용량: 각 클래스별로 상수 복사본 존재
- 계산 오버헤드: 동일한 `Year.now().getValue() + 2` 연산 중복 수행

**변경 후**:

- 중앙 상수 참조 (여전히 컴파일 타임 최적화 가능)
- 메모리 사용량: 단일 상수 정의로 메모리 절약
- 계산 오버헤드: 동일하지만 논리적으로 통합됨

**성능 영향**:

- 런타임 성능: 무시할 수 있는 수준 (상수 참조는 컴파일러 최적화)
- 메모리 사용: 미미한 개선 (중복 상수 제거)
- 코드 로딩: 클래스 수 감소로 약간의 개선

### 6-2. 유효성 검증 로직 변화

**검증 로직 복잡도**:

```java
// 변경 전후 모두 동일한 계산 복잡도 O(1)
int maxYear = Year.now(clock).getValue() + TimeConfig.HOLIDAY_AHEAD_YEARS;
```

**캐싱 최적화 가능성**:

```java
// 향후 개선 가능한 캐싱 전략
@Component
public class YearValidationConfig {

    private volatile int cachedMaxYear = -1;
    private volatile int cachedForYear = -1;

    public int getMaxAllowedYear(Clock clock) {
        int currentYear = Year.now(clock).getValue();

        // 연도가 바뀌지 않았으면 캐시된 값 사용
        if (cachedForYear == currentYear) {
            return cachedMaxYear;
        }

        // 연도 변경 시에만 재계산
        cachedForYear = currentYear;
        cachedMaxYear = currentYear + TimeConfig.HOLIDAY_AHEAD_YEARS;

        return cachedMaxYear;
    }
}
```

### 6-3. 메모리 및 유지보수 효율성

**코드 복잡도 감소**:

- 매직 넘버 제거로 코드 이해도 향상
- 중복 로직 제거로 유지보수 지점 감소
- 설정 변경 시 수정 범위 최소화

**장기적 성능 이점**:

- 버그 발생 확률 감소로 디버깅 시간 단축
- 일관된 정책 적용으로 예측 가능한 동작 보장

---

## 7. 관련 이슈 및 예방책

### 7-1. 설정 관리 안티패턴

**위험한 패턴들**:

```java
// 매직 넘버 남용
public class BadHolidayService {
    public List<Holiday> getHolidays(int year) {
        if (year > getCurrentYear() + 3) { // 3은 어디서 온 값인가?
            throw new IllegalArgumentException("너무 먼 미래입니다.");
        }
        // ...
    }
}

// 설정 중복과 불일치
public class BadController {
    @YearWithin(min = 1900, aheadYears = 2) // 2년
    int year;
}

public class BadService {
    private boolean isValidYear(int year) {
        return year <= getCurrentYear() + 5; // 5년 - 불일치!
    }
}
```

**안전한 패턴들**:

```java
// 중앙 집중식 설정 관리
@Configuration
public class BusinessConfig {

    // 명확한 네이밍과 문서화
    public static final int HOLIDAY_MAX_FUTURE_YEARS = 5;
    public static final int USER_SESSION_TIMEOUT_MINUTES = 30;
    public static final int MAX_FILE_UPLOAD_SIZE_MB = 10;

    // 관련 설정을 그룹화
    public static class HolidayPolicy {
        public static final int MIN_YEAR = 1900;
        public static final int MAX_FUTURE_YEARS = 5;
        public static final int DEFAULT_CACHE_TTL_HOURS = 24;
    }
}

// 설정 사용 시 명확한 참조
public class HolidayService {
    private boolean isValidYear(int year) {
        int maxYear = getCurrentYear() + BusinessConfig.HolidayPolicy.MAX_FUTURE_YEARS;
        return year >= BusinessConfig.HolidayPolicy.MIN_YEAR && year <= maxYear;
    }
}
```

### 7-2. 유효성 검증 일관성 체크리스트

**계층별 검증 일관성**:

- [ ] Controller와 Service에서 동일한 검증 규칙 사용
- [ ] 에러 메시지가 일관된 형식과 내용으로 구성
- [ ] 경계값(boundary value) 처리가 모든 계층에서 일치
- [ ] 예외 상황 처리 방식의 통일성

**설정 중앙화 검증**:

- [ ] 비즈니스 규칙이 단일 위치에서 정의됨
- [ ] 매직 넘버가 명확한 상수명으로 대체됨
- [ ] 설정 변경 시 영향 범위가 명확히 문서화됨
- [ ] 설정 의미와 변경 사유가 주석으로 기록됨

```java
// 체크리스트 적용 예시
@Configuration
public class ValidationConfig {

    /**
     * 사용자 입력 연도 유효성 검증 정책
     *
     * 비즈니스 근거:
     * - 과거: 1900년 이후 (공휴일 데이터 보유 시작점)
     * - 미래: 현재+5년 (정책 변경 가능성 고려한 합리적 범위)
     *
     * 영향 범위:
     * - HolidayController.@YearWithin
     * - HolidayService.validateYear()
     * - 모든 연도 관련 에러 메시지
     *
     * 변경 이력:
     * - v1.0: +2년 제한 (초기 정책)
     * - v1.1: +5년 확장 (사용자 요구사항 반영)
     */
    public static final int HOLIDAY_MAX_FUTURE_YEARS = 5;
    public static final int HOLIDAY_MIN_YEAR = 1900;
}
```

### 7-3. 동적 설정 확장 방안

**현재 한계점**: 컴파일 타임 상수로 인한 런타임 변경 불가

```java
// 현재 방식의 제약
public static final int HOLIDAY_AHEAD_YEARS = 5; // 재배포 없이 변경 불가
```

**확장 가능한 설계**:

```yaml
# application.yml - 외부 설정 파일 활용
app:
  holiday:
    validation:
      min-year: 1900
      max-future-years: 5
      cache-ttl-hours: 24

  # 환경별 다른 정책 적용 가능
---
spring:
  profiles: development
app:
  holiday:
    validation:
      max-future-years: 10 # 개발환경에서는 더 넓은 범위 허용

---
spring:
  profiles: production
app:
  holiday:
    validation:
      max-future-years: 5 # 운영환경에서는 보수적 정책
```

```java
// 동적 설정 로딩
@Component
@ConfigurationProperties(prefix = "app.holiday.validation")
@Data
public class HolidayValidationProperties {

    private int minYear = 1900;
    private int maxFutureYears = 5;
    private int cacheTtlHours = 24;

    // Setter를 통한 런타임 변경 지원
    public void setMaxFutureYears(int maxFutureYears) {
        if (maxFutureYears < 1 || maxFutureYears > 20) {
            throw new IllegalArgumentException("maxFutureYears는 1-20 사이여야 합니다.");
        }
        this.maxFutureYears = maxFutureYears;
    }
}

// 설정 변경 API
@RestController
@RequestMapping("/admin/config")
public class ConfigController {

    private final HolidayValidationProperties properties;

    @PutMapping("/holiday/max-future-years")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<String> updateMaxFutureYears(@RequestBody int newValue) {
        properties.setMaxFutureYears(newValue);
        return ResponseEntity.ok("설정이 변경되었습니다: " + newValue);
    }
}
```

### 7-4. 테스트 전략 고도화

**설정 기반 테스트**:

```java

@TestConfiguration
public class TestTimeConfig {

    // 테스트용 시간 고정
    @Bean
    @Primary
    public Clock testClock() {
        return Clock.fixed(Instant.parse("2025-06-15T00:00:00Z"), ZoneOffset.UTC);
    }
}

@ExtendWith(SpringExtension.class)
@Import(TestTimeConfig.class)
class HolidayValidationTest {

    @ParameterizedTest
    @ValueSource(ints = {1900, 2025, 2030})
        // 경계값 테스트
    void 유효한_연도_검증_성공(int validYear) {
        assertThat(validator.isValid(validYear, context)).isTrue();
    }

    @ParameterizedTest
    @ValueSource(ints = {1899, 2031})
        // 경계 밖 값 테스트
    void 무효한_연도_검증_실패(int invalidYear) {
        assertThat(validator.isValid(invalidYear, context)).isFalse();
    }

    @Test
    void 설정_변경시_모든_계층_일관성_유지() {
        // TimeConfig 값을 동적으로 변경했을 때의 일관성 검증
        // (실제로는 @TestPropertySource 등을 활용)
    }
}
```

**통합 테스트 자동화**:

```java

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class HolidayApiValidationIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @DynamicTest
    Stream<DynamicTest> 연도별_API_응답_검증() {
        return IntStream.of(1899, 1900, 2025, 2030, 2031)
                .mapToObj(year -> DynamicTest.dynamicTest(
                        "Year " + year + " 검증",
                        () -> testYearValidation(year)
                ));
    }

    private void testYearValidation(int year) {
        String url = "/api/v1/holidays/monthly?year=" + year + "&month=1";
        ResponseEntity<String> response = restTemplate.getForEntity(url, String.class);

        if (year >= 1900 && year <= 2030) { // 2025 + 5
            assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        } else {
            assertThat(response.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
            assertThat(response.getBody()).contains("연도는 1900–2030 사이여야 합니다");
        }
    }
}
```

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
- `TimeConfig` 클래스를 통한 설정 중앙화로 전체 시스템 정책 투명성 확보

**매직 넘버 제거의 효과**:

- `+2`, `+5` 같은 의미 불분명한 숫자를 `HOLIDAY_AHEAD_YEARS`로 명시
- 코드 리뷰 시 비즈니스 의도 파악 용이성 증대
- 신규 개발자의 코드 이해도 및 온보딩 효율성 향상

**계층별 유효성 검증 일관성**:

- Controller(@YearWithin)과 Service(validateYear) 간 중복 검증 로직 통합
- 동일한 중앙 설정을 참조하여 검증 기준의 일치성 보장
- 에러 메시지 동적 생성으로 설정 변경 시 자동 반영

### 8-3. 프로세스 개선

**설정 변경 영향도 분석 방법론**:

1. **영향 범위 사전 조사**: 특정 값을 사용하는 모든 코드 위치 파악
2. **계층별 일관성 검증**: Controller-Service-Repository 간 동일 정책 적용 여부 확인
3. **테스트 기반 검증**: 경계값 테스트로 설정 변경 후 동작 검증
4. **문서화**: 설정 변경 사유와 영향 범위를 코드 주석에 명시

**점진적 리팩토링 전략**:

- 기존 기능 유지하면서 설정 중앙화부터 시작
- 각 계층별로 순차적으로 중앙 설정 적용
- 테스트를 통한 회귀 버그 방지 및 일관성 검증
- 문서화를 통한 변경 사유 및 방법 공유

### 8-4. 장기적 개선 방향

**동적 설정 관리 시스템 구축**:

```java
// 1단계: 외부 설정 파일 활용 (현재 → 중기)
@ConfigurationProperties(prefix = "app.holiday")
public class HolidayProperties {
    private int maxFutureYears = 5;
    // application.yml에서 값 주입
}

// 2단계: 관리자 API를 통한 실시간 변경 (중기 → 장기)
@RestController
public class AdminConfigController {
    @PutMapping("/config/holiday/max-future-years")
    public ResponseEntity<?> updateMaxFutureYears(@RequestBody int newValue) {
        // 검증 후 실시간 적용
    }
}

// 3단계: 설정 변경 히스토리 관리 (장기)
@Entity
public class ConfigChangeHistory {
    private String configKey;
    private String oldValue;
    private String newValue;
    private LocalDateTime changedAt;
    private String changedBy;
}
```

**모니터링 및 알림 체계**:

```java

@Component
public class HolidayValidationMetrics {

    private final MeterRegistry meterRegistry;

    @EventListener
    public void handleValidationFailure(ValidationFailureEvent event) {
        // 검증 실패 빈도 모니터링
        meterRegistry.counter("holiday.validation.failure",
                        "reason", event.getReason(),
                        "year", String.valueOf(event.getYear()))
                .increment();

        // 임계치 초과 시 알림
        if (isFailureRateHigh(event)) {
            alertService.sendAlert("공휴일 API 검증 실패 급증");
        }
    }
}
```

**다국가 지원 확장**:

```java
// 국가별 다른 정책 적용 가능한 구조
@Configuration
public class CountrySpecificHolidayConfig {

    @Bean
    public Map<String, HolidayPolicy> countryPolicies() {
        Map<String, HolidayPolicy> policies = new HashMap<>();

        policies.put("KR", HolidayPolicy.builder()
                .minYear(1900)
                .maxFutureYears(5)
                .build());

        policies.put("US", HolidayPolicy.builder()
                .minYear(1950) // 미국은 다른 최소 연도
                .maxFutureYears(3) // 미국은 보수적 정책
                .build());

        return policies;
    }
}
```

### 8-5. 실무 적용 가이드

**설정 중앙화 체크리스트**:

**설계 단계**:

- [ ] 비즈니스 규칙이 여러 곳에 중복 정의되지 않도록 설계
- [ ] 매직 넘버 대신 의미 있는 상수명 사용
- [ ] 설정 변경 시 영향 범위를 사전에 파악하고 문서화
- [ ] 계층 간 일관성을 보장할 수 있는 구조 설계

**구현 단계**:

- [ ] 중앙 설정 클래스 생성 및 명확한 네이밍
- [ ] 각 사용 지점에서 중앙 설정 참조로 변경
- [ ] 설정 변경에 따른 에러 메시지 동적 생성
- [ ] 단위 테스트 및 통합 테스트로 일관성 검증

**운영 단계**:

- [ ] 설정 변경 시 전체 시스템 영향도 점검
- [ ] 모니터링을 통한 설정 변경 효과 추적
- [ ] 정기적인 설정 정책 리뷰 및 개선
- [ ] 장애 발생 시 설정 원복 절차 수립

이러한 체계적인 접근을 통해 **매직 넘버와 분산된 설정으로 인한 유지보수 문제**를 근본적으로 해결하고, **단일 진실 공급원 원칙을 준수**하는 견고하고 확장 가능한 설정 관리 체계를 구축할 수 있습니다. 특히
중앙 집중식 설정 관리 패턴은 다양한 비즈니스 규칙이 존재하는 엔터프라이즈 애플리케이션에서 필수적인 설계 원칙으로 활용될 수 있습니다.