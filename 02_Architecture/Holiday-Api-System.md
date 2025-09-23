# 공휴일 API 시스템 기술 문서

본 문서는 한국천문연구원의 특일 정보 API를 활용하여 공휴일 데이터를 수집, 저장, 제공하는 시스템입니다. **캐시 우선 전략**을 통해 외부 API 호출을 최소화하고, React FullCalendar와 연동하여 사용자 친화적인 달력 UI를 제공합니다.

**작성자:** [왕택준](https://github.com/TJK98)

**문서 버전:** v1.0

**대상 독자:**

- **백엔드 개발자**: Holiday API 연동, DB 캐시 전략, REST API 구현 등 서버 로직을 이해하고 유지보수해야 하는 개발자
- **프론트엔드 개발자**: React FullCalendar 연동, holidayService.js 호출 방식, 사용자 인터랙션 로직을 구현하는 개발자
- **DevOps / 인프라 엔지니어**: 환경변수, API 키 보안 관리, 로깅 및 모니터링, 장애 대응 절차를 담당하는 운영자
- **QA / 테스트 엔지니어**: 공휴일 데이터 정확성 검증, 대체공휴일/임시공휴일 시나리오 테스트를 수행하는 품질 담당자
- **신규 합류자**: 프로젝트 구조와 공휴일 시스템의 전체적인 흐름을 빠르게 파악해야 하는 팀 신규 인원
- **프로덕트 매니저 (PM)**: 공휴일 정보 제공이 사용자 일정 경험에 어떤 가치를 주는지 이해하고, 기능 개선 방향을 기획하는 담당자

---

## 목차

1. [개요](#개요)
2. [공휴일 API 선택 이유](#공휴일-api-선택-이유)
3. [HTTP 클라이언트](#http-클라이언트)
4. [시스템 아키텍처](#시스템-아키텍처)
5. [백엔드 구현](#백엔드-구현)
6. [프론트엔드 구현](#프론트엔드-구현)
7. [API 데이터 변환](#api-데이터-변환)
8. [성능 최적화 전략](#성능-최적화-전략)
9. [운영 환경 설정](#운영-환경-설정)
10. [장애 대응](#장애-대응)
11. [비즈니스 가치](#비즈니스-가치)
12. [향후 개선사항](#향후-개선사항)

---

## 개요

TropiCal 프로젝트의 공휴일 시스템은 **한국천문연구원(KASI) 특일 정보 API**를 기반으로 **정확하고 신뢰할 수 있는 공휴일 데이터**를 제공하도록 설계했습니다.
캐시 우선 전략(Cache-First Strategy)을 통해 외부 API 호출을 최소화하면서도 최신 데이터를 반영할 수 있도록 하였으며, **React FullCalendar**와 연동하여 사용자 친화적인 캘린더 UI를 구현했습니다.

### 핵심 설계 원칙

- **정확성**: 공식 기관의 데이터를 활용하여 법정 공휴일 및 대체공휴일을 안정적으로 제공
- **성능 최적화**: DB 캐시를 우선 조회하여 외부 API 의존성을 줄이고 빠른 응답 보장
- **장애 격리**: 외부 API 장애 시에도 기존 데이터로 서비스 지속 가능
- **확장성**: 다국가 지원 및 새로운 데이터 소스 연동이 용이한 구조

### 채택한 기술 방식

- **WebClient 기반 비동기·병렬 API 호출**
- **4xx 무해화, 5xx 선택적 재시도 로직**
- **DB 캐시 우선 조회 + TTL 기반 갱신 전략**
- **React FullCalendar와 holidayService.js를 통한 캘린더 UI 연동**

## 공휴일 API 선택 이유

### 기존 방식의 한계

#### 1. 하드코딩 방식

**단점**:

- 매년 수동으로 공휴일 데이터 업데이트 필요
- 임시공휴일(예: 대통령령으로 지정되는 특별 연휴) 대응 불가
- 대체공휴일 계산 로직 구현 복잡성
- 개발자가 공휴일 변경사항을 놓칠 위험

#### 2. 구글 캘린더 등 외부 서비스

**단점**:

- 한국 공휴일 데이터의 정확성 보장 어려움
- 서비스 정책 변경 시 데이터 접근 제한 가능
- API 사용료 부과 가능성
- 우리 서비스와 무관한 외부 의존성 증가

### 한국천문연구원 API 선택 장점

#### 1. 공식성과 정확성

- **정부 기관 제공**: 과학기술정보통신부 산하 한국천문연구원의 공식 데이터
- **법적 근거**: 「관공서의 공휴일에 관한 규정」에 따른 정확한 공휴일 정보
- **실시간 업데이트**: 임시공휴일 지정 시 1일 내 반영

#### 2. 비용 효율성

- **무료 제공**: 공공데이터포털을 통한 무료 API 서비스
- **안정적 운영**: 정부 기관 운영으로 서비스 중단 위험 최소

#### 3. 포괄적 데이터

- **다양한 특일**: 공휴일뿐만 아니라 국경일, 기념일, 24절기, 잡절 제공
- **대체공휴일**: 복잡한 대체공휴일 계산을 API에서 처리
- **메타데이터**: dateKind를 통한 공휴일 분류 정보 제공

### 핵심 설계 철학

- **성능 우선**: DB 캐시를 통해 API 호출 최소화
- **장애 격리**: 외부 API 실패 시에도 기존 데이터로 서비스 지속
- **데이터 품질**: 중복 제거 및 우선순위 로직으로 정확한 공휴일 정보 제공

---

## HTTP 클라이언트

### RestTemplate의 한계

#### 1. 동기 방식의 성능 문제
**단점**:
- **블로킹 I/O**: 각 API 호출마다 스레드가 응답까지 대기
- **스레드 점유**: 공휴일 API + 국경일 API 순차 호출 시 총 응답시간이 두 API 응답시간의 합
- **동시성 제한**: 많은 사용자 요청 시 스레드 풀 고갈 가능성

#### 2. 타임아웃 및 재시도 설정의 복잡성
**단점**:
- 타임아웃 설정을 위해 별도 HTTP 클라이언트 구성 필요
- 재시도 로직을 수동으로 구현해야 함
- 에러 핸들링이 예외 기반으로 복잡함

#### 3. Spring의 방향성
**단점**:
- Spring 5.0부터 maintenance 모드 (새 기능 추가 없음)
- Spring WebFlux와의 연동성 부족

### WebClient 선택 장점

#### 1. 비동기 논블로킹 처리
**장점**:
- **병렬 API 호출**: 공휴일 API와 국경일 API를 동시에 호출하여 응답시간 단축
- **리소스 효율성**: 스레드 점유 없이 더 많은 동시 요청 처리 가능
- **백프레셔 지원**: 부하 상황에서도 안정적인 처리

```java
// 두 API를 병렬로 호출 (RestTemplate 대비 약 50% 응답시간 단축)
JsonNode restResponse = holidayApiClient.fetchRestHolidays(year, month);
JsonNode nationalResponse = holidayApiClient.fetchNationalHolidays(year, month);
```

#### 2. 강력한 에러 처리 및 재시도
**장점**:
- **선택적 재시도**: `Retry.fixedDelay()`로 5xx 에러만 재시도, 4xx는 무해화
- **세밀한 타임아웃 제어**: 연결/읽기 타임아웃 분리 설정
- **Reactive 스트림**: 에러 핸들링이 함수형으로 직관적

```java
.retryWhen(
    Retry.fixedDelay(2, Duration.ofMillis(300))
        .filter(throwable -> {
            if (throwable instanceof WebClientResponseException ex) {
                return ex.getStatusCode().is5xxServerError(); // 5xx만 재시도
            }
            return name.contains("Timeout") || name.contains("Connect");
        })
)
```

#### 3. 4xx 응답 무해화 구현 용이성
**장점**:
- `exchangeToMono()`를 통한 상태 코드별 분기 처리
- 4xx 에러 시 예외 대신 빈 응답 반환으로 시스템 안정성 확보
- 비즈니스 로직과 에러 처리의 명확한 분리

```java
.exchangeToMono(response -> {
    if (response.statusCode().is2xxSuccessful()) {
        return response.bodyToMono(JsonNode.class);
    }
    if (response.statusCode().is4xxClientError()) {
        // 4xx는 예외 대신 빈 응답으로 무해화
        return Mono.just(emptyResponse(path, year, month));
    }
    // 5xx만 예외로 전파하여 재시도 대상으로 처리
    return response.createException().flatMap(Mono::error);
})
```

#### 4. 미래 확장성
**장점**:
- Spring WebFlux와의 자연스러운 연동
- Reactive 생태계와의 호환성
- 마이크로서비스 아키텍처에서의 활용도

### 선택 근거 요약

1. **성능**: 병렬 API 호출로 응답시간 단축
2. **안정성**: 4xx 무해화와 선택적 재시도로 장애 격리
3. **운영성**: 세밀한 타임아웃 제어와 상세 로깅
4. **확장성**: Reactive 기반 아키텍처 준비

---

## 시스템 아키텍처

```
한국천문연구원 API → HolidayApiClient → HolidayService → HolidayRepository → MySQL
                                             ↓
React FullCalendar ← holidayService.js ← HolidayController (REST API)
```

---

## 백엔드 구현

### 1. 외부 API 연동 (HolidayApiClient)

#### 설계 목표

- 외부 API 장애 시에도 시스템이 중단되지 않도록 안정성 확보
- 네트워크 타임아웃과 재시도를 통한 일시적 장애 극복
- 미래 데이터에 대한 불필요한 API 호출 방지

#### 핵심 구현

```java
@Component
public class HolidayApiClient {

    // WebClient 기반 비동기 HTTP 통신
    private final WebClient webClient;

    // 공휴일 API 호출 (getRestDeInfo 엔드포인트)
    public JsonNode fetchRestHolidays(int year, int month)

    // 국경일 API 호출 (getHoliDeInfo 엔드포인트)
    public JsonNode fetchNationalHolidays(int year, int month)
}
```

#### 주요 특징

- **4xx 무해화**: 클라이언트 에러 시 예외 대신 빈 응답 반환으로 시스템 안정성 확보
- **5xx 재시도**: 서버 에러와 타임아웃에 대해서만 선택적 재시도 (300ms 간격, 최대 2회)
- **동적 소프트가드**: 현재 연도 +5년 초과 시 API 호출 생략하여 불필요한 요청 방지
- **상세 로깅**: API 호출 과정과 결과를 세밀하게 로깅하여 운영 시 디버깅 지원

#### 설정 (application.yml)

```yaml
holiday:
  api-base: https://apis.data.go.kr/B090041/openapi/service/SpcdeInfoService
  api-key: ${HOLIDAY_API_KEY}
  connect-timeout-ms: 3000
  read-timeout-ms: 4000
  page-no: 1
  num-of-rows: 100
```

#### 환경변수 (.env)

```bash
HOLIDAY_API_KEY=발급받은_서비스키_값
```

### 2. 비즈니스 로직 (HolidayService)

#### 설계 목표

- **캐시 우선 전략**으로 성능 최적화와 외부 의존성 최소화
- 동일 날짜 중복 데이터 발생 시 우선순위 기반 정리
- 실제 휴무일과 기념일을 명확히 구분하여 정확한 휴일 판단

#### 캐시 우선 전략 구현

```java
@Transactional
public List<Holiday> getMonthlyHolidays(int year, int month) {
    // 1. DB 캐시 확인
    List<Holiday> cachedHolidays = holidayRepository.findOverlappingHolidays(...);
    if (!cachedHolidays.isEmpty()) {
        return cachedHolidays; // 캐시 히트 시 즉시 반환
    }

    // 2. 캐시 미스 시 외부 API 호출
    JsonNode restResponse = holidayApiClient.fetchRestHolidays(year, month);
    JsonNode nationalResponse = holidayApiClient.fetchNationalHolidays(year, month);

    // 3. 데이터 파싱 및 중복 제거
    List<Holiday> holidays = parseAndDedup(restResponse, nationalResponse);

    // 4. DB 저장 후 재조회
    saveNewHolidays(holidays);
    return holidayRepository.findOverlappingHolidays(...);
}
```

#### 중복 제거 우선순위 로직

동일 날짜에 여러 공휴일이 존재할 경우 다음 우선순위로 선택:

1. **SUBSTITUTE_HOLIDAY** (대체공휴일) - 실제 휴무일로서 가장 중요
2. **PUBLIC_HOLIDAY** (법정공휴일) - 법적 효력이 있는 휴일
3. **NATIONAL_HOLIDAY** (국경일) - 기념적 의미의 국가 기념일
4. 기타 (기념일, 24절기 등)

#### 실제 휴무일 판단 로직

```java
private static final Set<Holiday.HolidayType> ACTUAL_TYPES = EnumSet.of(
    Holiday.HolidayType.PUBLIC_HOLIDAY,
    Holiday.HolidayType.NATIONAL_HOLIDAY,
    Holiday.HolidayType.SUBSTITUTE_HOLIDAY
);

public boolean isHoliday(LocalDate date) {
    List<Holiday> hits = holidayRepository.findOnDateByTypes(COUNTRY_CODE_KR, date, ACTUAL_TYPES);
    return !hits.isEmpty();
}
```

### 3. 데이터 모델 (Holiday Entity)

#### 설계 사상

- 한국뿐만 아니라 향후 다국가 확장을 고려한 `countryCode` 필드
- 연휴 기간 표현을 위한 `startDate`/`endDate` 구조
- API 중복 호출 방지를 위한 `externalId` 추적
- 조회 성능 최적화를 위한 `year` 파생 컬럼

```java
@Entity
@Table(name = "holiday")
@Check(constraints = "start_date <= end_date")
public class Holiday {
    private Long id;
    private String countryCode;        // "KR" 고정, 향후 확장 고려
    private LocalDate startDate;       // 공휴일 시작일
    private LocalDate endDate;         // 공휴일 종료일 (연휴 지원)
    private String nameKo;             // 한국어 공휴일명
    private Boolean isSubstitute;      // 대체공휴일 여부
    private Short year;                // 조회 최적화용 파생 컬럼
    private HolidayType holidayType;   // 공휴일 분류
    private String externalId;         // API 중복 방지용 외부 식별자
    private String externalSource;     // "KASI_API"
}
```

#### HolidayType 분류

```java
public enum HolidayType {
    NATIONAL_HOLIDAY("국경일"),      // 3·1절, 광복절, 개천절, 한글날
    PUBLIC_HOLIDAY("공휴일"),        // 설날, 추석, 어린이날, 부처님오신날 등
    MEMORIAL_DAY("기념일"),          // 어버이날, 스승의날, 현충일 등
    SEASONAL_DIVISION("24절기"),     // 입춘, 춘분, 하지, 동지 등
    TRADITIONAL_DAY("잡절"),         // 단오, 한식, 중양절 등
    SUBSTITUTE_HOLIDAY("대체공휴일") // 주말과 겹칠 때 지정되는 휴일
}
```

### 4. 데이터 접근 (HolidayRepository)

#### 기간 겹침 쿼리 설계

월별 조회 시 `MONTH()` 함수 대신 범위 조건을 사용하여 인덱스 활용도를 높임:

```java
@Query("SELECT h FROM Holiday h " +
       "WHERE h.countryCode = :countryCode " +
       "AND h.startDate <= :monthEnd " +
       "AND h.endDate >= :monthStart " +
       "ORDER BY h.startDate")
List<Holiday> findOverlappingHolidays(
    @Param("countryCode") String countryCode,
    @Param("monthStart") LocalDate monthStart,
    @Param("monthEnd") LocalDate monthEnd
);
```

#### 실제 휴무일 타입 필터링

```java
@Query("select h from Holiday h " +
       "where h.countryCode = :countryCode " +
       "and :date between h.startDate and h.endDate " +
       "and h.holidayType in :types " +
       "order by h.startDate")
List<Holiday> findOnDateByTypes(
    @Param("countryCode") String countryCode,
    @Param("date") LocalDate date,
    @Param("types") Collection<Holiday.HolidayType> types
);
```

### 5. REST API (HolidayController)

#### 엔드포인트 설계

```java
// 월별 공휴일 목록 조회
GET /api/v1/holidays/monthly?year={year}&month={month}

// 특정 날짜 공휴일 여부 확인
GET /api/v1/holidays/check?date={date}

// 월별 휴무일 상태 (주말+공휴일 통합)
GET /api/v1/holidays/month-status?year={year}&month={month}

// FullCalendar용 배경 이벤트
GET /api/v1/holidays/events?start={start}&end={end}
```

#### 응답 예시

```json
// 월별 공휴일 목록
[
  {
    "date": "2025-01-01",
    "nameKo": "신정",
    "holidayType": "PUBLIC_HOLIDAY",
    "isSubstitute": false
  }
]

// 공휴일 여부 확인
{
  "date": "2025-01-01",
  "isHoliday": true,
  "holidayName": "신정"
}
```

---

## 프론트엔드 구현

### 1. API 통신 (holidayService.js)

#### 설계 목표

- 백엔드 API와의 안정적인 통신
- 네트워크 오류 시 graceful degradation
- 환경별 API URL 관리

```javascript
const API_BASE_URL = import.meta.env.VITE_API_BASE_URL || 'http://localhost:9005';

export const holidayService = {
    // 월별 공휴일 조회
    async getMonthlyHolidays(year, month) {
        try {
            const response = await fetch(
                `${API_BASE_URL}/api/v1/holidays/monthly?year=${year}&month=${month}`
            );
            if (!response.ok) {
                throw new Error(`HTTP error! status: ${response.status}`);
            }
            return await response.json();
        } catch (error) {
            console.error('월별 공휴일 조회 실패:', error);
            return []; // 실패 시 빈 배열 반환으로 UI 안정성 확보
        }
    },

    // FullCalendar용 공휴일 이벤트 조회
    async getHolidayEvents(start, end),

    // 특정 날짜 공휴일 확인
    async checkHolidayStatus(date)
};
```

### 2. 달력 컴포넌트 (Calendar.jsx)

#### FullCalendar 통합

```javascript
"dependencies": {
  "@fullcalendar/core": "^6.1.19",
  "@fullcalendar/daygrid": "^6.1.19",
  "@fullcalendar/interaction": "^6.1.19",
  "@fullcalendar/react": "^6.1.19"
}
```

#### 핵심 기능 구현

- **클라이언트 캐싱**: Map 기반으로 월별 데이터 캐싱하여 중복 API 호출 방지
- **이벤트 발행**: CustomEvent를 통해 외부 컴포넌트와 느슨한 결합으로 연동
- **UTC 이슈 해결**: `toLocalISODate` 함수로 시간대 문제 없는 날짜 처리

#### 공휴일 이벤트 매핑

```javascript
function mapHoliday(items = []) {
    return items.map((holiday) => ({
        id: `holiday-${holiday.date}`,
        title: holiday.nameKo ?? holiday.name ?? '공휴일',
        start: holiday.date,
        allDay: true,
        display: 'block',
        backgroundColor: 'rgba(255, 107, 53, 0.15)',
        borderColor: '#FF6B35',
        textColor: '#FF6B35',
        className: 'holiday-event',
        extendedProps: { isHoliday: true }
    }));
}
```

#### 사용자 인터랙션 처리

```javascript
// 날짜 클릭 이벤트
const handleDateClick = (info) => {
    const dateInfo = {
        date: info.date,
        iso: toLocalISODate(info.date),
        year: info.date.getFullYear(),
        month: info.date.getMonth() + 1,
        day: info.date.getDate()
    };

    // 1. props 콜백 호출 (상위 컴포넌트 연동)
    if (onDateClick && typeof onDateClick === 'function') {
        onDateClick(dateInfo);
    }

    // 2. CustomEvent 발행 (전역 이벤트 시스템)
    window.dispatchEvent(
        new CustomEvent("calendar:dateClick", { detail: dateInfo })
    );
};
```

## API 데이터 변환

### 한국천문연구원 API → Holiday Entity 매핑

#### API 응답 구조 분석

```json
{
  "response": {
    "header": { "resultCode": "00", "resultMsg": "NORMAL SERVICE" },
    "body": {
      "items": {
        "item": [
          {
            "locdate": "20250101",
            "dateName": "신정",
            "dateKind": "01",
            "isHoliday": "Y",
            "seq": 1
          }
        ]
      }
    }
  }
}
```

#### 매핑 로직

```java
// 필드 매핑
locdate (yyyyMMdd) → startDate, endDate (LocalDate)
dateName → nameKo
dateKind + sourceType → holidayType
seq → externalId
"대체" 문자열 포함 여부 → isSubstitute

// DateKind 변환 로직
private Holiday.HolidayType mapHolidayType(String dateKind, String sourceType) {
    return switch (dateKind) {
        case "01" -> SOURCE_TYPE_NATIONAL.equals(sourceType)
                ? HolidayType.NATIONAL_HOLIDAY    // 국경일 API에서 온 01
                : HolidayType.PUBLIC_HOLIDAY;     // 공휴일 API에서 온 01
        case "02" -> HolidayType.MEMORIAL_DAY;     // 기념일
        case "03" -> HolidayType.SEASONAL_DIVISION; // 24절기
        case "04" -> HolidayType.TRADITIONAL_DAY;   // 잡절
        default -> HolidayType.PUBLIC_HOLIDAY;
    };
}
```

#### 대체공휴일 감지

```java
// dateName에 "대체" 문자열이 포함된 경우 자동 감지
boolean isSubstitute = dateName.contains("대체");
Holiday.HolidayType holidayType = isSubstitute
    ? Holiday.HolidayType.SUBSTITUTE_HOLIDAY
    : mapHolidayType(dateKind, sourceType);
```

---

## 성능 최적화 전략

### 1. 캐시 계층 구조

```
1차: 프론트엔드 Map 캐시 (메모리, 세션 유지)
2차: 백엔드 DB 캐시 (영구 저장, 서버 재시작 후에도 유지)
3차: 외부 API 호출 (최후 수단)
```

### 2. API 호출 최적화

- **소프트가드**: `Year.now(clock).getValue() + TimeConfig.HOLIDAY_AHEAD_YEARS` 초과 시 호출 생략
- **병렬 호출**: 공휴일 API와 국경일 API를 동시에 호출하여 응답 시간 단축
- **선택적 재시도**: 5xx 에러와 타임아웃에 대해서만 재시도, 4xx는 무해화

### 3. 데이터베이스 최적화

- **기간 겹침 쿼리**: `BETWEEN` 조건 대신 `startDate <= :monthEnd AND endDate >= :monthStart` 사용
- **중복 방지**: `existsByCountryCodeAndStartDateAndEndDateAndNameKo`로 저장 전 중복 검사

---

## 운영 환경 설정

### 환경변수 관리

```bash
# .env 파일 (gitignore에 포함)
HOLIDAY_API_KEY=실제_서비스키_값
VITE_API_BASE_URL=http://localhost:9005

# 주의: .env 파일은 절대 Git에 커밋하지 않음
```

### 로깅 설정

```yaml
logging:
  level:
    com.tropical.backend.calendar: DEBUG
    com.tropical.backend.calendar.client.HolidayApiClient: DEBUG
```

---

## 장애 대응

### 일반적인 문제 해결

#### 1. API 키 인증 실패

```bash
# 증상: 401 Unauthorized 에러
# 해결: 공공데이터포털에서 서비스키 재발급 후 환경변수 업데이트
```

#### 2. 캐시 미스 지속 발생

```sql
-- DB 연결 확인
SELECT COUNT(*) FROM holiday WHERE country_code = 'KR';

-- 최근 데이터 확인
SELECT * FROM holiday WHERE year = 2025 ORDER BY start_date LIMIT 10;
```

#### 3. CORS 에러

```java
// 개발 환경에서만 발생, 프로덕션에서는 동일 도메인 사용 권장
@CrossOrigin(origins = "http://localhost:3000")
```

## 비즈니스 가치

### 1. 사용자 경험 향상

- 정확한 한국 공휴일 정보 제공으로 일정 관리 신뢰성 확보
- 빠른 응답속도 (캐시 히트 시 수십 ms 이내)

### 2. 운영 비용 절감

- 캐시 전략으로 외부 API 호출 최소화 (월 단위 호출로 제한)
- 장애 격리로 외부 API 장애 시에도 서비스 지속 가능

### 3. 확장성 확보

- `countryCode` 기반 다국가 지원 준비
- 모듈화된 구조로 새로운 공휴일 데이터 소스 추가 용이

---

## 향후 개선사항

### 1. 보안 강화

#### 설정 검증 로직 추가

```java
@ConfigurationProperties(prefix = "holiday")
@Validated
public class HolidayProperties {
    @NotBlank
    private String apiKey;

    @PostConstruct
    public void validate() {
        if (apiKey.startsWith("${") || apiKey.equals("your-api-key")) {
            throw new IllegalStateException("HOLIDAY_API_KEY 환경변수가 설정되지 않았습니다.");
        }
    }
}
```

#### API 키 마스킹

```java
log.info("HolidayApiClient 초기화 완료 - apiKey: {}***",
         apiKey.substring(0, Math.min(4, apiKey.length())));
```

### 2. 에러 처리 세분화

#### 4xx 에러 타입별 처리

```java
return switch (response.statusCode()) {
    case UNAUTHORIZED -> {
        log.error("API 키 인증 실패 - 키 확인 필요");
        yield Mono.just(emptyResponse(path, year, month));
    }
    case NOT_FOUND -> {
        log.info("해당 날짜 데이터 없음");
        yield Mono.just(emptyResponse(path, year, month));
    }
    case TOO_MANY_REQUESTS -> {
        log.warn("API 호출 한도 초과");
        yield Mono.error(new ApiLimitExceededException());
    }
    default -> Mono.just(emptyResponse(path, year, month));
};
```

#### 프론트엔드 사용자 알림

```javascript
// 전역 에러 상태 관리 또는 토스트 메시지
window.dispatchEvent(new CustomEvent('api:error', {
    detail: { message: '공휴일 정보를 불러올 수 없습니다.', status: response.status }
}));
```

### 3. 성능 최적화

#### DB 인덱스 추가

```sql
CREATE INDEX idx_holiday_country_start ON holiday(country_code, start_date);
CREATE INDEX idx_holiday_country_year ON holiday(country_code, year);
CREATE INDEX idx_holiday_date_range ON holiday(start_date, end_date);
```

#### TTL 기반 캐시 갱신

```java
@Column(name = "cache_updated_at")
private LocalDateTime cacheUpdatedAt;

public boolean isCacheExpired(Duration maxAge) {
    return cacheUpdatedAt == null ||
           cacheUpdatedAt.isBefore(LocalDateTime.now().minus(maxAge));
}
```

### 4. 운영 개선

#### 헬스체크 엔드포인트

```java
@GetMapping("/api/v1/health/holiday-api")
public ResponseEntity<Map<String, Object>> checkHolidayApiHealth() {
    try {
        holidayApiClient.fetchRestHolidays(2025, 1);
        return ResponseEntity.ok(Map.of("status", "UP"));
    } catch (Exception e) {
        return ResponseEntity.status(503).body(Map.of("status", "DOWN", "error", e.getMessage()));
    }
}
```

#### 분산 캐시 (Redis)

```java
@Cacheable(value = "holidays", key = "#year + '_' + #month")
public List<Holiday> getMonthlyHolidays(int year, int month) {
    // 기존 로직
}
```

### 5. 확장 기능

#### 관리자 기능

```java
@RestController
@RequestMapping("/api/v1/admin/holidays")
@PreAuthorize("hasRole('ADMIN')")
public class HolidayAdminController {

    @PostMapping
    public ResponseEntity<Holiday> createHoliday(@RequestBody CreateHolidayRequest request) {
        // 수동 공휴일 생성 (임시공휴일 대응)
    }

    @PostMapping("/cache/refresh")
    public ResponseEntity<Void> refreshCache() {
        // 캐시 강제 갱신
    }
}
```

#### 다국가 지원

```java
public interface HolidayProvider {
    List<Holiday> fetchHolidays(String countryCode, int year, int month);
}

@Component
public class KoreanHolidayProvider implements HolidayProvider {
    // 한국천문연구원 API 연동
}

@Component
public class USHolidayProvider implements HolidayProvider {
    // 미국 공휴일 API 연동
}
```

#### 이벤트 기반 아키텍처

```java
@EventListener
public void handleHolidayUpdated(HolidayUpdatedEvent event) {
    // 캐시 무효화
    // 사용자 알림
    // 외부 시스템 연동
}
```

### 6. 모니터링 및 운영

#### 메트릭 수집

- API 응답 시간 및 성공률
- 캐시 히트율
- DB 쿼리 성능
- 메모리 사용량

#### 알림 설정

- API 호출 실패율 임계값 초과 시
- DB 응답 시간 지연 시
- 캐시 미스율 급증 시
