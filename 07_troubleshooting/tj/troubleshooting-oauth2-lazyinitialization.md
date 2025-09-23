# 트러블슈팅: OAuth2 소셜 로그인 LazyInitializationException

본 문서는 **Tropical 백엔드** 개발 과정에서 OAuth2 소셜 로그인 구현 시 발생한 **Hibernate LazyInitializationException** 문제의 원인과 해결 과정을 정리한 기술
문서입니다.

네이버, 카카오, 구글 소셜 로그인에서 발생한 일관성 없는 동작 문제를 다루며, 향후 유사한 JPA/Hibernate 프록시 관련 문제 해결에 도움이 되는 것을 목표로 합니다.

**작성자:** 왕택준  
**작성일:** 2025년 9월 16일  
**문서 버전:** v1.0

---

## 1. 문제 현상

### 1-1. `LazyInitializationException` (네이버/카카오)

* **에러 메시지**
  ```
  org.hibernate.LazyInitializationException: Could not initialize proxy [com.tropical.backend.auth.entity.User#4] - no session
  at com.tropical.backend.auth.entity.User$HibernateProxy.isOnboardingCompleted(Unknown Source)
  ```
* **상황**: 네이버, 카카오 소셜 로그인 시 기존 사용자 재로그인 과정에서 발생
* **특징**: 신규 가입은 성공하지만, 두 번째 로그인부터 실패

### 1-2. 구글 로그인 성공 (일관성 없는 동작)

* **현상**: 동일한 코드임에도 구글은 정상 작동
* **상황**: 신규 가입 후 추가 로그인 테스트를 하지 않아 문제를 발견하지 못함
* **특징**: 실제로는 구글도 재로그인 시 같은 문제 발생 예상

### 1-3. Spring Boot 로그 분석

```
2025-09-16T10:46:31.497+09:00  INFO 15488 --- 기존 소셜 사용자 로그인 - 사용자 ID: 2, 제공자: NAVER
2025-09-16T10:46:31.500+09:00 ERROR 15488 --- OAuth2 로그인 처리 실패 - registrationId: naver, 오류: Could not initialize proxy
```

---

## 2. 원인 분석

### 2-1. Hibernate 프록시와 트랜잭션 경계 문제

OAuth2AuthenticationSuccessHandler에서 발생한 문제:

```java
if(existingSocialAccount.isPresent()){
User user = existingSocialAccount.get().getUser(); // Hibernate 프록시 반환
// 트랜잭션 종료
String redirectUrl = determineRedirectUrl(user, isNewUser);
}

private String determineRedirectUrl(User user, boolean isNewUser) {
    if (isNewUser || !user.isOnboardingCompleted()) { // LazyInitializationException 발생!
        return frontendBaseUrl + "/onboarding";
    }
}
```

### 2-2. 신규 vs 기존 사용자 처리 차이점

**신규 사용자 (구글 성공 케이스)**:

- `userService.createSocialUser()` → 실제 User 엔티티 생성
- 프록시가 아닌 완전한 객체이므로 필드 접근 가능

**기존 사용자 (네이버/카카오 실패 케이스)**:

- `socialAccountRepository.findByProvider...()` → SocialAccount 조회
- `socialAccount.getUser()` → Hibernate 프록시 반환
- 트랜잭션 종료 후 `isOnboardingCompleted()` 접근 시 예외 발생

### 2-3. 트랜잭션 생명주기 분석

```
[SocialAccountService@findActiveSocialAccount] 
    → @Transactional 시작
    → SocialAccount 조회 (User는 LAZY 프록시로 로드)
    → @Transactional 종료
[OAuth2AuthenticationSuccessHandler]
    → user.isOnboardingCompleted() 호출
    → "no session" 에러 발생
```

---

## 3. 해결 과정

### 3-1. 시도했던 접근법들

**A안: Fetch Join 방식 (기각)**

```java

@Query("""
        select sa from SocialAccount sa 
        join fetch sa.user u 
        where sa.provider = :provider and sa.providerUserId = :providerUserId
        """)
Optional<SocialAccount> findActiveWithUser(...);
```

기각 이유: Repository가 특정 사용 사례에 종속되며, 다른 곳에서 불필요한 join 발생

**B안: Service 재조회 방식 (채택)**

```java
if(existingSocialAccount.isPresent()){
Long userId = existingSocialAccount.get().getUser().getId(); // ID 접근은 안전
User user = userService.getById(userId); // 트랜잭션 내에서 완전 조회
}
```

### 3-2. 최종 해결책

UserService에 메서드 추가:

```java

@Transactional(readOnly = true)
public User getById(Long userId) {
    User user = userRepository.findById(userId)
            .orElseThrow(() -> new IllegalArgumentException("사용자를 찾을 수 없습니다: " + userId));

    // Hibernate 프록시 강제 초기화
    boolean initialized = user.isOnboardingCompleted();
    log.debug("User 엔티티 초기화 완료 - 사용자 ID: {}", userId);

    return user;
}
```

OAuth2AuthenticationSuccessHandler 리팩토링:

1. **관심사 분리**: User 엔티티 대신 필요한 값만 전달
2. **메서드 분리**: JWT 발급 로직을 별도 메서드로 추출
3. **프록시 안전성**: Service 레이어에서 완전한 엔티티 조회

---

## 4. 추가 개선사항

### 4-1. 시그니처 개선

```java
// 기존: 엔티티를 UI 결정 로직에 전달
private String determineRedirectUrl(User user, boolean isNewUser)

// 개선: 필요한 값만 전달하여 관심사 분리
private String determineRedirectUrl(boolean onboardingCompleted, boolean isNewUser)
```

### 4-2. 방어적 코딩

- 프록시 초기화 확인 로직 추가
- 명시적인 트랜잭션 경계 설정
- 에러 로깅 강화

### 4-3. 테스트 전략 개선

- 신규 가입뿐만 아니라 **재로그인 시나리오** 필수 테스트
- 각 소셜 제공자별 일관성 검증
- 프록시 관련 에지 케이스 커버

---

## 5. 테스트 코드

### 5-1. 재로그인 시나리오 테스트

```java

@Test
@DisplayName("기존 사용자 재로그인 시 프록시 초기화 정상 처리")
void testExistingUserRelogin() {
    // Given: 기존 소셜 계정 설정
    SocialAccount existingAccount = createSocialAccount(NAVER, "existing-user-id");

    // When: 재로그인 시도
    MockHttpServletRequest request = new MockHttpServletRequest();
    MockHttpServletResponse response = new MockHttpServletResponse();

    // Then: LazyInitializationException 발생하지 않음
    assertDoesNotThrow(() -> {
        successHandler.onAuthenticationSuccess(request, response, authentication);
    });

    // And: 올바른 리다이렉트 URL 반환
    String redirectLocation = response.getHeader("Location");
    assertThat(redirectLocation).contains("/dashboard");
}
```

### 5-2. 프록시 초기화 검증 테스트

```java

@Test
@DisplayName("UserService.getById()가 완전한 엔티티를 반환하는지 검증")
void testUserEntityFullyLoaded() {
    // Given
    Long userId = 1L;

    // When
    User user = userService.getById(userId);

    // Then: 프록시가 아닌 실제 엔티티 확인
    assertThat(user).isNotInstanceOf(HibernateProxy.class);
    assertThat(user.isOnboardingCompleted()).isNotNull();
}
```

---

## 6. 성능 영향 분석

### 6-1. 변경 전후 비교

**변경 전**: 프록시 + 트랜잭션 외부 접근

- 메모리: 낮음 (프록시 객체)
- 쿼리: 1회 (SocialAccount만 조회)
- 결과: LazyInitializationException 발생

**변경 후**: Service 레이어에서 완전 조회

- 메모리: 약간 증가 (완전한 User 엔티티)
- 쿼리: 2회 (SocialAccount + User 별도 조회)
- 결과: 안정적 동작

### 6-2. 최적화 고려사항

- **추가 쿼리 발생**: SocialAccount 조회 후 User 재조회
- **메모리 사용량**: 프록시 대신 완전한 엔티티 로드
- **트레이드오프**: 안정성을 위한 합리적인 성능 비용

---

## 7. 관련 이슈 및 예방책

### 7-1. 프록시 안전성 체크리스트

**위험한 패턴들**:

```java
// 트랜잭션 외부에서 지연 로딩 필드 접근
User user = repository.findById(id).get();
// @Transactional 종료
return user.

getSomeField(); // LazyInitializationException 위험
```

**안전한 패턴들**:

```java
// Service 레이어에서 완전 조회
@Transactional(readOnly = true)
public UserDto getUserInfo(Long userId) {
    User user = userRepository.findById(userId).orElseThrow();
    // 트랜잭션 내에서 필요한 필드 모두 접근
    return new UserDto(user.getName(), user.getEmail(), user.isOnboardingCompleted());
}
```

### 7-2. 코드 리뷰 체크포인트

- 트랜잭션 경계 외부에서 엔티티 필드 접근 여부 확인
- 프록시 객체를 다른 레이어로 전달하는 코드 검토
- ID 접근과 다른 필드 접근의 차이점 인지
- Service 레이어에서 완전한 엔티티 조회 보장

### 7-3. 아키텍처 가이드라인

```java
// 권장 패턴: DTO 변환을 Service 레이어에서 처리
@Service
public class UserService {
    @Transactional(readOnly = true)
    public UserInfoDto getUserInfo(Long userId) {
        User user = userRepository.findById(userId).orElseThrow();
        // 트랜잭션 내에서 DTO 변환
        return UserInfoDto.from(user);
    }
}

// 지양 패턴: 엔티티를 직접 반환
@Service
public class UserService {
    public User getUser(Long userId) {
        return userRepository.findById(userId).orElseThrow(); // 프록시 위험
    }
}
```

---

## 8. 결론 및 배운 점

### 8-1. 주요 성과

1. **근본 원인 해결**: Hibernate 프록시와 트랜잭션 경계 이해
2. **아키텍처 개선**: 레이어 간 책임 분리 강화
3. **테스트 전략 수립**: 재로그인 시나리오 포함한 전체 플로우 검증
4. **방어적 코딩**: 프록시 관련 에지 케이스 대응

### 8-2. 기술적 학습

**JPA/Hibernate 프록시 이해**:

```
인라인 스타일 > CSS !important > ID 선택자 > 클래스 선택자
```

- 지연 로딩은 성능 최적화의 핵심이지만 트랜잭션 경계를 명확히 이해해야 함
- 프록시 객체는 트랜잭션 밖에서 필드 접근 시 `LazyInitializationException` 발생
- ID 접근은 안전하지만, 다른 필드 접근은 세션이 필요

**계층 간 책임 분리**:

- 인증 레이어에서 영속성 엔티티에 직접 의존하지 않도록 설계
- Service 레이어에서 완전한 엔티티 조회 후 전달
- UI 결정 로직에는 원시 값이나 DTO 사용 권장

### 8-3. 프로세스 개선

**체계적 디버깅 방법론**:

1. 에러 로그에서 프록시 패턴 (`$HibernateProxy`) 식별
2. 트랜잭션 경계와 세션 생명주기 분석
3. 표면적 증상이 아닌 아키텍처 레벨에서의 해결책 모색
4. 전체 플로우 테스트로 숨어있는 문제 발견

**테스트 시나리오의 중요성**:

- Happy Path만 테스트하면 숨어있는 문제를 놓칠 수 있음
- 신규 가입 + 재로그인 전체 플로우 검증 필수
- 각 소셜 제공자별 동일한 시나리오 테스트로 일관성 확보

### 8-4. 장기적 개선 방향

**OAuth2 통합 테스트 강화**:

- 모든 소셜 제공자에 대해 동일한 테스트 시나리오 적용
- 신규/기존 사용자 모든 케이스 커버

**아키텍처 가이드라인 수립**:

- 레이어 간 엔티티 전달 규칙 정의
- 프록시 처리 표준 패턴 문서화

이러한 경험을 통해 Hibernate의 지연 로딩 메커니즘을 더 깊이 이해하게 되었으며, 향후 유사한 문제 발생 시 신속한 해결이 가능할 것으로 기대됩니다.