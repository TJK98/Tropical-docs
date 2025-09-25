# 트러블슈팅: 계정 타입별 이메일 중복 검증 이슈 및 인증 시스템 설계 개선

본 문서는 **Tropical 백엔드** 개발 과정에서 발생한 **이메일 중복 검증 로직의 비일관성** 문제의 원인과 해결 과정을 정리한 기술 문서입니다.

소셜 로그인과 로컬 계정 간의 이메일 중복 처리 정책 차이로 인한 트랜잭션 롤백 및 회원가입 실패 문제를 해결하여 일관성 있고 사용자 친화적인 인증 시스템을 구축하는 것을 목표로 합니다.

**작성자:** [왕택준](https://github.com/TJK98)

**작성일:** 2025년 9월 16일

**문서 버전:** v1.0

---

## 1. 문제 현상

### 1-1. 트랜잭션 롤백으로 인한 UnexpectedRollbackException 발생

* **에러**: `UnexpectedRollbackException: Transaction silently rolled back because it has been marked as rollback-only`
* **발생 시나리오**: 소셜 계정이 이미 존재하는 이메일로 로컬 회원가입 시도
* **근본 원인**: 이메일 중복 검증 실패 후 트랜잭션이 롤백 전용으로 마킹되어 후속 작업 불가

### 1-2. 계정 타입별 비대칭적 이메일 중복 검증

```java
// 로컬 계정 생성 시 - 엄격한 검증
if (userRepository.existsByEmailAndActive(email)) {
    throw new IllegalArgumentException("이미 사용 중인 이메일입니다: " + email);
    // 모든 계정 타입(로컬+소셜)에서 중복 체크
}

// 소셜 계정 생성 시 - 검증 없음
User user = User.createSocialUser(email, nickname);
User savedUser = userRepository.save(user);
// 이메일 중복 체크 로직 완전 누락
```

### 1-3. 순서 의존적인 회원가입 결과

* **소셜 → 로컬 순서**: wtj199814@gmail.com로 구글 가입 후 로컬 가입 시도 → 실패
* **로컬 → 소셜 순서**: wtj199814@gmail.com로 로컬 가입 후 구글 가입 시도 → 성공
* **논리적 모순**: 동일한 최종 상태를 원하는데 순서에 따라 결과가 달라짐

### 1-4. 사용자 혼란 야기

```
실제 사용자 시나리오:
1단계: 구글 소셜 로그인으로 서비스 가입 (wtj199814@gmail.com)
2단계: 나중에 비밀번호 로그인도 사용하고 싶어서 로컬 계정 추가 생성 시도
3단계: "이미 사용 중인 이메일입니다" 에러로 인한 사용자 좌절
```

### 1-5. 환경 정보

- **백엔드**: Spring Boot 3.5.5, Java 17
- **데이터베이스**: MariaDB
- **추가 라이브러리**: Spring Data JPA, Spring Security, Spring Transaction, Spring Validation, Jackson, JUnit 5, Mockito
- **인증 방식**: JWT(ACCESS/REFRESH, HttpOnly 쿠키)
- **외부 OAuth**: Google, Kakao, Naver
- **테스트 환경**: Postman
- **브라우저**: Chrome 140+
- **운영체제**: Windows 11, macOS Sequoia

---

## 2. 원인 분석

### 2-1. 1단계: 설계 철학의 모호성 의심

**가설**: 계정 타입별 이메일 중복 정책이 명확하게 정의되지 않음

```
현재 암묵적 정책 (일관성 없음):
- 소셜 계정끼리: 같은 이메일 허용
- 소셜 → 로컬: 이메일 중복으로 차단  
- 로컬 → 소셜: 허용
- 로컬끼리: 중복 차단
```

**결과**: 사용자와 개발자 모두 예측할 수 없는 동작 확인

### 2-2. 2단계: Repository 메서드 설계 문제 의심

**가설**: 전체 계정을 대상으로 하는 `existsByEmailAndActive()` 사용의 부적절성

```java
// 문제가 된 메서드
@Query("SELECT CASE WHEN COUNT(u) > 0 THEN true ELSE false END " +
       "FROM User u WHERE u.email = :email AND u.status = 'ACTIVE'")
boolean existsByEmailAndActive(@Param("email") String email);
```

**결과**: 계정 타입을 구분하지 않는 단일 검증 로직으로 인한 비일관성 확인

### 2-3. 3단계: 비즈니스 로직과 기술적 제약 충돌 의심

**가설**: 동일 이메일 다중 계정 허용 vs 중복 방지 간 정책 충돌

```java
// 비즈니스 요구사항: 같은 이메일로 여러 인증 방식 사용 허용
사용자: "구글, 네이버, 로컬 계정 모두 wtj199814@gmail.com로 사용하고 싶어요"

// 기존 기술적 제약: 이메일 유일성 강제
시스템: "이미 사용 중인 이메일입니다"
```

**결과**: 사용자 요구사항과 시스템 제약 간 근본적 충돌 확인

### 2-4. 4단계: 트랜잭션 예외 처리 문제 의심

**가설**: 이메일 중복 검증 실패 시 트랜잭션 상태 관리 부적절

```java
// 예외 발생 플로우
@Transactional
public User createLocalUser(String email, String password) {
    if (userRepository.existsByEmailAndActive(email)) {
        throw new IllegalArgumentException("이미 사용 중인 이메일입니다");
        // 트랜잭션이 rollback-only로 마킹됨
    }
    // 이후 로직 실행 시 UnexpectedRollbackException 발생
}
```

**결과**: 예외 발생 후 트랜잭션 롤백 처리 미흡으로 인한 부차적 에러 발생 확인

### 2-5. 5단계: 근본 원인 발견

**핵심 발견**: 인증 수단별 독립적 계정 관리 vs 이메일 기반 통합 관리 간 설계 충돌

**구조적 문제들**:
- 소셜 계정과 로컬 계정의 이메일 중복 정책 비대칭성
- 계정 타입을 고려하지 않는 단일 검증 로직
- 사용자 실제 사용 패턴과 시스템 제약 간 불일치
- 예외 상황에서 트랜잭션 상태 관리 부실

---

## 3. 디버깅 과정

### 3-1. 체계적 디버깅 방법론

**회원가입 플로우 상세 추적**: 각 계정 타입별 가입 과정 분석

```java
@Transactional
public User createLocalUser(String email, String password, String nickname) {
    log.debug("=== 로컬 계정 생성 시작 ===");
    log.debug("이메일: {}, 닉네임: {}", email, nickname);
    
    // 중복 검증 수행
    boolean emailExists = userRepository.existsByEmailAndActive(email);
    log.debug("이메일 중복 검사 결과: {}", emailExists);
    
    if (emailExists) {
        // 기존 계정 정보 상세 확인
        List<User> existingUsers = userRepository.findAllByEmailAndActive(email);
        for (User user : existingUsers) {
            log.debug("기존 계정: ID={}, 타입={}, 제공자={}", 
                     user.getId(), user.getAccountType(), user.getProvider());
        }
        
        log.warn("로컬 계정 생성 실패 - 이메일 중복: {}", email);
        throw new IllegalArgumentException("이미 사용 중인 이메일입니다: " + email);
    }
    
    // 계정 생성 로직
    User user = User.createLocalUser(email, passwordEncoder.encode(password), nickname);
    User savedUser = userRepository.save(user);
    
    log.info("로컬 계정 생성 완료: ID={}, 이메일={}", savedUser.getId(), savedUser.getEmail());
    return savedUser;
}
```

**소셜 계정 생성 플로우 분석**: 중복 검증 누락 확인

```java
@Transactional
public User createSocialUser(String email, String nickname, String provider) {
    log.debug("=== 소셜 계정 생성 시작 ===");
    log.debug("이메일: {}, 닉네임: {}, 제공자: {}", email, nickname, provider);
    
    // 중복 검증 없음 - 이것이 문제
    log.debug("소셜 계정은 이메일 중복 검사를 수행하지 않음");
    
    User user = User.createSocialUser(email, nickname, provider);
    User savedUser = userRepository.save(user);
    
    log.info("소셜 계정 생성 완료: ID={}, 이메일={}, 제공자={}", 
             savedUser.getId(), savedUser.getEmail(), savedUser.getProvider());
    return savedUser;
}
```

**데이터베이스 상태 분석**: 실제 계정 구성 확인

```sql
-- 동일 이메일로 생성된 모든 계정 조회
SELECT id, email, account_type, provider, status, created_at 
FROM users 
WHERE email = 'wtj199814@gmail.com' AND status = 'ACTIVE'
ORDER BY created_at;

-- 결과 예시:
-- ID | EMAIL              | ACCOUNT_TYPE | PROVIDER | STATUS | CREATED_AT
-- 1  | wtj199814@gmail.com | SOCIAL       | GOOGLE   | ACTIVE | 2025-09-16 10:00:00
-- 2  | wtj199814@gmail.com | SOCIAL       | NAVER    | ACTIVE | 2025-09-16 10:05:00
-- (로컬 계정 생성 시도 시 위 데이터로 인해 실패)
```

**트랜잭션 상태 추적**: 롤백 발생 지점 파악

```java
@Transactional
public User createLocalUser(String email, String password, String nickname) {
    try {
        log.debug("트랜잭션 시작 - 현재 상태: {}", 
                 TransactionSynchronizationManager.getCurrentTransactionName());
        
        if (userRepository.existsByEmailAndActive(email)) {
            log.warn("트랜잭션을 rollback-only로 마킹");
            throw new IllegalArgumentException("이미 사용 중인 이메일입니다: " + email);
        }
        
        // 이 지점은 실행되지 않음
        User user = User.createLocalUser(email, passwordEncoder.encode(password), nickname);
        return userRepository.save(user);
        
    } catch (Exception e) {
        log.error("로컬 계정 생성 중 예외 발생", e);
        log.debug("트랜잭션 상태: rollback-only={}", 
                 TransactionSynchronizationManager.getCurrentTransactionStatus().isRollbackOnly());
        throw e; // 예외 재던짐으로 트랜잭션 롤백 확정
    }
}
```

---

## 4. 해결 과정

### 4-1. 실패한 해결책들

**A. 소셜 계정에도 동일한 중복 검증 추가**

```java
// 시도했던 방법: 모든 계정 타입에 동일한 검증 적용
@Transactional
public User createSocialUser(String email, String nickname, String provider) {
    if (userRepository.existsByEmailAndActive(email)) {
        throw new IllegalArgumentException("이미 사용 중인 이메일입니다: " + email);
    }
    // 계정 생성 로직
}
```

**실패 이유**: 소셜 계정끼리도 이메일 중복 허용 불가로 사용자 경험 급격히 악화

**B. 예외 대신 기존 계정 반환**

```java
// 중복 시 예외 대신 기존 계정 조회 후 반환
@Transactional
public User createLocalUser(String email, String password, String nickname) {
    User existingUser = userRepository.findByEmailAndActive(email);
    if (existingUser != null) {
        log.info("기존 계정 반환: {}", existingUser.getId());
        return existingUser; // 소셜 계정을 로컬 계정으로 잘못 반환
    }
    // 신규 생성 로직
}
```

**실패 이유**: 계정 타입이 다른 기존 계정 반환으로 인한 인증 메커니즘 오동작

**C. 이메일 변경 강제**

```java
// 소셜 계정이 있으면 로컬 계정 이메일에 접미사 추가
public User createLocalUser(String email, String password, String nickname) {
    if (userRepository.existsByEmailAndActive(email)) {
        email = email + ".local"; // wtj199814@gmail.com.local
    }
    // 수정된 이메일로 계정 생성
}
```

**실패 이유**: 사용자가 원하는 이메일과 다른 주소로 강제 변경되어 혼란 야기

### 4-2. 최종 해결책: 계정 타입별 분리된 이메일 중복 검증

**1. Repository에 계정 타입별 중복 검증 메서드 추가**

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // 기존 메서드 유지 (하위 호환성)
    @Query("SELECT CASE WHEN COUNT(u) > 0 THEN true ELSE false END " +
           "FROM User u WHERE u.email = :email AND u.status = 'ACTIVE'")
    boolean existsByEmailAndActive(@Param("email") String email);
    
    /**
     * 로컬 계정에서만 이메일 중복 확인
     * 소셜 계정과 로컬 계정을 분리하여 관리하기 위한 메서드
     * 
     * @param email 확인할 이메일 주소
     * @return 로컬 계정 중 해당 이메일이 존재하면 true
     */
    @Query("SELECT CASE WHEN COUNT(u) > 0 THEN true ELSE false END " +
           "FROM User u WHERE u.email = :email AND u.status = 'ACTIVE' AND u.accountType = 'LOCAL'")
    boolean existsByEmailAndActiveAndLocal(@Param("email") String email);
    
    /**
     * 소셜 계정에서 특정 제공자별 이메일 중복 확인 (향후 확장용)
     */
    @Query("SELECT CASE WHEN COUNT(u) > 0 THEN true ELSE false END " +
           "FROM User u WHERE u.email = :email AND u.status = 'ACTIVE' " +
           "AND u.accountType = 'SOCIAL' AND u.provider = :provider")
    boolean existsByEmailAndActiveAndSocialProvider(@Param("email") String email, 
                                                   @Param("provider") String provider);
}
```

**2. UserService 로컬 계정 생성 로직 개선**

```java
@Service
public class UserService {
    
    /**
     * 로컬 계정 생성 (이메일/비밀번호 인증)
     * 로컬 계정끼리만 이메일 중복을 체크하여 소셜 계정과 별도 관리
     */
    @Transactional
    public User createLocalUser(String email, String password, String nickname) {
        log.info("로컬 계정 생성 시작 - 이메일: {}, 닉네임: {}", email, nickname);
        
        // 계정 타입별 분리된 중복 검증
        if (userRepository.existsByEmailAndActiveAndLocal(email)) {
            log.warn("로컬 계정 이메일 중복 - 이미 존재하는 로컬 이메일: {}", email);
            throw new IllegalArgumentException("이미 사용 중인 로컬 계정 이메일입니다: " + email);
        }
        
        // 참고: 소셜 계정 존재 여부는 체크하지 않음 (정책적 허용)
        List<User> socialAccounts = userRepository.findByEmailAndActiveAndSocial(email);
        if (!socialAccounts.isEmpty()) {
            log.info("동일 이메일의 소셜 계정 {}개 존재, 로컬 계정 생성 허용", socialAccounts.size());
            for (User socialAccount : socialAccounts) {
                log.debug("기존 소셜 계정: ID={}, 제공자={}", 
                         socialAccount.getId(), socialAccount.getProvider());
            }
        }
        
        // 로컬 계정 생성
        User user = User.createLocalUser(email, passwordEncoder.encode(password), nickname);
        User savedUser = userRepository.save(user);
        
        log.info("로컬 계정 생성 완료 - 사용자 ID: {}, 이메일: {}", savedUser.getId(), savedUser.getEmail());
        return savedUser;
    }
    
    /**
     * 소셜 계정 생성 (OAuth 인증)
     * 현재 정책: 소셜 계정끼리는 이메일 중복 허용 (제공자별 독립)
     */
    @Transactional
    public User createSocialUser(String email, String nickname, String provider) {
        log.info("소셜 계정 생성 시작 - 이메일: {}, 제공자: {}", email, provider);
        
        // 현재 정책: 소셜 계정은 이메일 중복 검증 없음
        // 향후 정책 변경 시 여기에 검증 로직 추가 가능
        log.debug("소셜 계정 이메일 중복 검증 생략 (제공자별 독립 계정 정책)");
        
        User user = User.createSocialUser(email, nickname, provider);
        User savedUser = userRepository.save(user);
        
        log.info("소셜 계정 생성 완료 - 사용자 ID: {}, 이메일: {}, 제공자: {}", 
                savedUser.getId(), savedUser.getEmail(), savedUser.getProvider());
        return savedUser;
    }
}
```

**3. 명확한 정책 문서화**

```java
/**
 * 계정 타입별 이메일 중복 정책 (v1.0)
 * 
 * 현재 정책:
 * 1. 로컬 계정: 로컬 계정끼리만 이메일 중복 방지
 * 2. 소셜 계정: 이메일 중복 허용 (제공자별 독립 계정)
 * 3. 교차 허용: 소셜-로컬 간 동일 이메일 사용 가능
 * 
 * 허용되는 계정 구성 (같은 이메일 기준):
 * 로컬 계정 1개 (이메일/비밀번호)
 * 구글 소셜 계정 1개 (Google OAuth)
 * 네이버 소셜 계정 1개 (Naver OAuth)  
 * 카카오 소셜 계정 1개 (Kakao OAuth)
 * 
 * 금지되는 계정 구성:
 * 로컬 계정 2개 (동일 이메일)
 * 동일 제공자 소셜 계정 2개 (향후 정책에 따라 변경 가능)
 * 
 * 향후 발전 방향:
 * - 1단계: 계정 연동 기능 (사용자 주도적 통합)
 * - 2단계: 스마트 로그인 (이메일 기반 인증 방식 추천)
 * - 3단계: 완전 통합 (이메일 기준 단일 계정, 다중 인증 방식)
 */
```

**4. 단위 테스트 및 통합 테스트 추가**

```java
@SpringBootTest
@Transactional
class UserServiceAccountTypeTest {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void 소셜_계정_존재시_로컬_계정_생성_성공() {
        // Given: 구글 소셜 계정 생성
        String email = "test@example.com";
        User socialUser = userService.createSocialUser(email, "소셜사용자", "GOOGLE");
        assertThat(socialUser.getAccountType()).isEqualTo(AccountType.SOCIAL);
        
        // When: 동일 이메일로 로컬 계정 생성
        User localUser = userService.createLocalUser(email, "password123", "로컬사용자");
        
        // Then: 로컬 계정 생성 성공
        assertThat(localUser.getAccountType()).isEqualTo(AccountType.LOCAL);
        assertThat(localUser.getEmail()).isEqualTo(email);
        
        // 동일 이메일로 2개 계정 공존 확인
        List<User> allUsers = userRepository.findByEmailAndActive(email);
        assertThat(allUsers).hasSize(2);
        assertThat(allUsers).extracting("accountType")
                           .containsExactlyInAnyOrder(AccountType.SOCIAL, AccountType.LOCAL);
    }
    
    @Test
    void 로컬_계정_존재시_동일_이메일_로컬_계정_생성_실패() {
        // Given: 로컬 계정 생성
        String email = "test@example.com";
        userService.createLocalUser(email, "password123", "기존사용자");
        
        // When & Then: 동일 이메일로 로컬 계정 재생성 시도 시 예외 발생
        assertThatThrownBy(() -> userService.createLocalUser(email, "password456", "신규사용자"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessage("이미 사용 중인 로컬 계정 이메일입니다: " + email);
    }
    
    @Test
    void 소셜_계정_다중_제공자_생성_성공() {
        // Given: 동일 이메일, 다른 제공자
        String email = "test@example.com";
        
        // When: 여러 소셜 제공자로 계정 생성
        User googleUser = userService.createSocialUser(email, "구글사용자", "GOOGLE");
        User naverUser = userService.createSocialUser(email, "네이버사용자", "NAVER");
        User kakaoUser = userService.createSocialUser(email, "카카오사용자", "KAKAO");
        
        // Then: 모든 계정 생성 성공
        List<User> socialUsers = userRepository.findByEmailAndActive(email);
        assertThat(socialUsers).hasSize(3);
        assertThat(socialUsers).extracting("provider")
                              .containsExactlyInAnyOrder("GOOGLE", "NAVER", "KAKAO");
    }
}
```

**성공 이유**:
- 명확한 정책 정의: 계정 타입별 분리된 이메일 관리 정책 수립
- 기술적 구현: Repository 계층에서 계정 타입별 검증 메서드 제공
- 사용자 경험 우선: 실제 사용 패턴에 맞는 유연한 계정 관리
- 확장 가능성: 향후 계정 통합 기능을 위한 기반 마련
- 트랜잭션 안정성: 명확한 예외 처리로 롤백 상황 개선

---

## 5. 테스트 검증

### 5-1. 계정 타입별 이메일 중복 허용 테스트

```java
@SpringBootTest
@AutoConfigureTestDatabase
class EmailDuplicationPolicyIntegrationTest {
    
    @Autowired
    private UserService userService;
    
    @Test
    @Transactional
    void 동일_이메일_다중_계정_타입_생성_성공() {
        String email = "wtj199814@gmail.com";
        String nickname = "테스트사용자";
        String password = "password123";
        
        // Given & When: 동일 이메일로 여러 계정 타입 생성
        User googleUser = userService.createSocialUser(email, nickname + "구글", "GOOGLE");
        User naverUser = userService.createSocialUser(email, nickname + "네이버", "NAVER");  
        User kakaoUser = userService.createSocialUser(email, nickname + "카카오", "KAKAO");
        User localUser = userService.createLocalUser(email, password, nickname + "로컬");
        
        // Then: 모든 계정 생성 성공
        assertThat(googleUser.getId()).isNotNull();
        assertThat(naverUser.getId()).isNotNull();
        assertThat(kakaoUser.getId()).isNotNull();
        assertThat(localUser.getId()).isNotNull();
        
        // 계정 타입별 구분 확인
        assertThat(googleUser.getAccountType()).isEqualTo(AccountType.SOCIAL);
        assertThat(googleUser.getProvider()).isEqualTo("GOOGLE");
        assertThat(localUser.getAccountType()).isEqualTo(AccountType.LOCAL);
        assertThat(localUser.getProvider()).isNull();
        
        // 데이터베이스에서 실제 저장 확인
        List<User> allUsers = userRepository.findByEmail(email);
        assertThat(allUsers).hasSize(4);
        assertThat(allUsers.stream().map(User::getProvider))
                          .containsExactlyInAnyOrder("GOOGLE", "NAVER", "KAKAO", null);
    }
    
    @Test
    void 로컬_계정_중복_생성_실패() {
        // Given: 기존 로컬 계정
        String email = "duplicate@test.com";
        userService.createLocalUser(email, "password1", "기존사용자");
        
        // When & Then: 동일 이메일 로컬 계정 재생성 시도
        assertThatThrownBy(() -> userService.createLocalUser(email, "password2", "중복사용자"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("이미 사용 중인 로컬 계정 이메일입니다");
    }
}
```

**트랜잭션 안정성 테스트**

```java
@Test
void 중복_검증_실패시_트랜잭션_롤백_정상_처리() {
    // Given: 기존 로컬 계정 생성
    String email = "transaction@test.com";  
    userService.createLocalUser(email, "password1", "기존사용자");
    
    long userCountBefore = userRepository.count();
    
    // When: 트랜잭션 내에서 중복 생성 시도 (예외 발생 예상)
    assertThatThrownBy(() -> {
        userService.createLocalUser(email, "password2", "중복사용자");
    }).isInstanceOf(IllegalArgumentException.class);
    
    // Then: 트랜잭션 롤백으로 사용자 수 변경 없음
    long userCountAfter = userRepository.count();
    assertThat(userCountAfter).isEqualTo(userCountBefore);
    
    // UnexpectedRollbackException 발생하지 않음 확인
    assertThatCode(() -> {
        userRepository.findByEmail(email); // 정상 동작 확인
    }).doesNotThrowAnyException();
}
```

### 5-3. API 레벨 통합 테스트

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class AuthApiIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    void 소셜_계정_후_로컬_계정_가입_성공() {
        // Given: 소셜 계정 OAuth 인증 시뮬레이션 (실제로는 OAuth 플로우)
        String email = "api@test.com";
        simulateOAuthLogin(email, "GOOGLE");
        
        // When: 동일 이메일로 로컬 계정 회원가입 API 호출
        SignUpRequest request = SignUpRequest.builder()
                .email(email)
                .password("password123")
                .nickname("로컬사용자")
                .build();
        
        ResponseEntity<String> response = restTemplate.postForEntity(
            "/api/auth/signup", request, String.class);
        
        // Then: 성공 응답
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).contains("회원가입이 완료되었습니다");
        
        // 데이터베이스에 두 계정 모두 존재 확인
        List<User> users = userRepository.findByEmail(email);
        assertThat(users).hasSize(2);
        assertThat(users).extracting("accountType")
                         .containsExactlyInAnyOrder(AccountType.SOCIAL, AccountType.LOCAL);
    }
    
    private void simulateOAuthLogin(String email, String provider) {
        // OAuth 인증 시뮬레이션 - 실제로는 소셜 제공자를 통한 인증 플로우
        userService.createSocialUser(email, "소셜사용자", provider);
    }
}
```

---

## 6. 성능 영향 분석

### 6-1. Repository 쿼리 최적화

**변경 전**:
```sql
-- existsByEmailAndActive(): 모든 활성 계정 검색
SELECT CASE WHEN COUNT(u) > 0 THEN true ELSE false END 
FROM User u WHERE u.email = ? AND u.status = 'ACTIVE'
```

**변경 후**:
```sql  
-- existsByEmailAndActiveAndLocal(): 로컬 계정만 검색
SELECT CASE WHEN COUNT(u) > 0 THEN true ELSE false END 
FROM User u WHERE u.email = ? AND u.status = 'ACTIVE' AND u.accountType = 'LOCAL'
```

**성능 개선 효과**:
- 스캔 범위 축소: 전체 계정 → 로컬 계정만
- 인덱스 활용도 향상: `(email, status, account_type)` 복합 인덱스 권장
- 동시성 개선: 계정 타입별 분리로 락 경합 감소

### 6-2. 메모리 및 처리 시간

**트랜잭션 처리 시간**:
- 중복 검증 실패 시 조기 반환으로 불필요한 DB 작업 방지
- 명확한 예외 처리로 롤백 시간 최소화

**메모리 사용량**:
- 검색 결과 집합 크기 감소로 메모리 효율성 향상
- 계정 타입별 캐시 분리 시 캐시 효율성 증대

### 6-3. 동시성 및 확장성

**동시 가입 처리**:
```java
// 계정 타입별 분리로 동시성 향상
Thread1: 로컬 계정 생성 (email@test.com) 
Thread2: 소셜 계정 생성 (email@test.com) 
// 서로 다른 검증 로직으로 락 경합 최소화
```

**향후 샤딩 준비**:
- 계정 타입별 테이블 분리 시 자연스러운 마이그레이션 경로 제공
- 소셜 제공자별 독립적인 확장 가능

---

## 7. 관련 이슈 및 예방책

### 7-1. 계정 관리 설계 안티패턴

**위험한 패턴들**:

```java
// 단일 검증 로직으로 모든 계정 타입 처리
public void createAccount(String email, AccountType type) {
    if (existsAnyAccount(email)) { // 타입 무시한 일괄 검증
        throw new IllegalArgumentException("중복 이메일");
    }
    // 계정 타입별 다른 요구사항 무시
}

// 계정 타입별 정책 하드코딩
public boolean canCreateAccount(String email, AccountType type) {
    if (type == AccountType.LOCAL) {
        return !existsLocalAccount(email); // 로컬만 체크
    } else {
        return true; // 소셜은 무조건 허용
    }
    // 정책이 코드에 산재되어 일관성 관리 어려움
}
```

**안전한 패턴들**:

```java
// 정책 중앙화 및 명시적 정의
@Component
public class AccountCreationPolicy {
    
    public boolean canCreateLocalAccount(String email) {
        // 로컬 계정 생성 정책을 한 곳에서 관리
        return !userRepository.existsByEmailAndActiveAndLocal(email);
    }
    
    public boolean canCreateSocialAccount(String email, String provider) {
        // 소셜 계정 생성 정책 (현재: 허용, 향후 변경 가능)
        return true; // 또는 제공자별 세분화된 정책
    }
}

// 계층별 책임 분리
@Service
public class UserService {
    
    private final AccountCreationPolicy policy;
    
    public User createLocalUser(String email, String password, String nickname) {
        if (!policy.canCreateLocalAccount(email)) {
            throw new BusinessException("로컬 계정 생성 불가");
        }
        // 실제 생성 로직
    }
}
```

### 7-2. 사용자 경험 최적화 가이드라인

**혼란 방지 전략**:

```java
// 사용자에게 명확한 안내 제공
@RestController
public class AuthController {
    
    @PostMapping("/signup")
    public ResponseEntity<?> signup(@RequestBody SignUpRequest request) {
        try {
            User user = userService.createLocalUser(
                request.getEmail(), request.getPassword(), request.getNickname());
            
            return ResponseEntity.ok(SignUpResponse.builder()
                .message("회원가입이 완료되었습니다.")
                .accountType("로컬")
                .availableLoginMethods(getAvailableLoginMethods(request.getEmail()))
                .build());
                
        } catch (IllegalArgumentException e) {
            return ResponseEntity.badRequest().body(ErrorResponse.builder()
                .message("이미 해당 이메일로 로컬 계정이 존재합니다.")
                .suggestion("소셜 로그인을 시도하거나 다른 이메일을 사용해주세요.")
                .availableAlternatives(getSocialLoginOptions(request.getEmail()))
                .build());
        }
    }
    
    private List<String> getAvailableLoginMethods(String email) {
        List<String> methods = new ArrayList<>();
        
        if (userRepository.existsByEmailAndActiveAndLocal(email)) {
            methods.add("이메일/비밀번호");
        }
        
        List<String> socialProviders = userRepository
            .findSocialProvidersByEmail(email);
        socialProviders.forEach(provider -> 
            methods.add(provider.toLowerCase() + " 소셜 로그인"));
        
        return methods;
    }
}
```

**프론트엔드 가이드라인**:

```javascript
// 회원가입 전 사용자 안내
function showSignupGuidance(email) {
    const existingMethods = checkExistingLoginMethods(email);
    
    if (existingMethods.length > 0) {
        showMessage({
            type: 'info',
            title: '기존 계정이 있습니다',
            message: `${email}로 다음 방법으로 로그인할 수 있습니다:`,
            methods: existingMethods,
            actions: [
                { text: '기존 계정으로 로그인', action: 'redirectToLogin' },
                { text: '새로운 인증 방식 추가', action: 'continueSignup' }
            ]
        });
    }
}

// 명확한 에러 메시지와 대안 제시
function handleSignupError(error) {
    if (error.code === 'DUPLICATE_LOCAL_EMAIL') {
        showError({
            message: '이미 해당 이메일로 가입된 계정이 있습니다.',
            alternatives: [
                '비밀번호를 잊으셨다면 비밀번호 재설정을 이용해주세요.',
                '소셜 로그인으로 가입하신 경우 해당 방법으로 로그인해주세요.',
                '다른 이메일 주소로 가입해보세요.'
            ]
        });
    }
}
```

### 7-3. 데이터 일관성 유지 전략

**감시 및 알림 시스템**:

```java
@Component
public class AccountConsistencyMonitor {
    
    @EventListener
    public void validateAccountCreation(UserCreatedEvent event) {
        User user = event.getUser();
        
        // 정책 위반 사항 검사
        if (user.getAccountType() == AccountType.LOCAL) {
            long duplicateLocalCount = userRepository
                .countByEmailAndActiveAndLocal(user.getEmail());
            
            if (duplicateLocalCount > 1) {
                alertService.sendAlert("로컬 계정 이메일 중복 발생: " + user.getEmail());
            }
        }
        
        // 비즈니스 규칙 위반 검사
        validateBusinessRules(user);
    }
    
    @Scheduled(cron = "0 0 2 * * *") // 매일 새벽 2시
    public void performDailyConsistencyCheck() {
        // 일관성 위반 사항 일괄 검사 및 보고
        List<String> violations = findConsistencyViolations();
        if (!violations.isEmpty()) {
            alertService.sendDailyReport("계정 일관성 위반 사항", violations);
        }
    }
}
```

**마이그레이션 안전장치**:

```java
@Component
public class AccountMigrationGuard {
    
    /**
     * 계정 정책 변경 시 기존 데이터 영향도 분석
     */
    public MigrationImpactReport analyzePolicyChange(AccountPolicy newPolicy) {
        MigrationImpactReport report = new MigrationImpactReport();
        
        // 영향받는 계정 수 계산
        long affectedAccounts = calculateAffectedAccounts(newPolicy);
        report.setAffectedAccountCount(affectedAccounts);
        
        // 데이터 마이그레이션 필요 여부
        boolean needsMigration = needsDataMigration(newPolicy);
        report.setNeedsMigration(needsMigration);
        
        // 사용자 커뮤니케이션 필요 계정
        List<User> needsNotification = findUsersNeedingNotification(newPolicy);
        report.setUsersNeedingNotification(needsNotification);
        
        return report;
    }
}
```

### 7-4. 테스트 전략 고도화

**경계 조건 테스트**:

```java
@ParameterizedTest
@ValueSource(strings = {"", " ", "invalid-email", "test@"})
void 잘못된_이메일_형식_처리(String invalidEmail) {
    assertThatThrownBy(() -> userService.createLocalUser(invalidEmail, "password", "nickname"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("유효하지 않은 이메일 형식");
}

@Test
void 동시_계정_생성_경쟁_조건_테스트() throws InterruptedException {
    String email = "concurrent@test.com";
    CountDownLatch latch = new CountDownLatch(2);
    AtomicInteger successCount = new AtomicInteger(0);
    AtomicInteger failCount = new AtomicInteger(0);
    
    // 동시에 같은 이메일로 로컬 계정 생성 시도
    ExecutorService executor = Executors.newFixedThreadPool(2);
    
    executor.submit(() -> {
        try {
            userService.createLocalUser(email, "password1", "user1");
            successCount.incrementAndGet();
        } catch (Exception e) {
            failCount.incrementAndGet();
        } finally {
            latch.countDown();
        }
    });
    
    executor.submit(() -> {
        try {
            userService.createLocalUser(email, "password2", "user2");
            successCount.incrementAndGet();
        } catch (Exception e) {
            failCount.incrementAndGet();
        } finally {
            latch.countDown();
        }
    });
    
    latch.await(5, TimeUnit.SECONDS);
    
    // 하나만 성공해야 함
    assertThat(successCount.get()).isEqualTo(1);
    assertThat(failCount.get()).isEqualTo(1);
}
```

**통합 시나리오 테스트**:

```java
@Test
void 실제_사용자_여정_시뮬레이션() {
    String email = "realuser@gmail.com";
    
    // 1단계: 구글 소셜 로그인으로 첫 가입
    User googleUser = userService.createSocialUser(email, "구글사용자", "GOOGLE");
    assertThat(googleUser).isNotNull();
    
    // 2단계: 네이버로도 가입 (같은 이메일)
    User naverUser = userService.createSocialUser(email, "네이버사용자", "NAVER");
    assertThat(naverUser).isNotNull();
    
    // 3단계: 로컬 계정도 생성 (이전에는 실패했던 시나리오)
    User localUser = userService.createLocalUser(email, "password123", "로컬사용자");
    assertThat(localUser).isNotNull();
    
    // 4단계: 사용자가 모든 인증 방식으로 로그인 가능한지 확인
    List<User> allAccounts = userRepository.findByEmail(email);
    assertThat(allAccounts).hasSize(3);
    
    Map<String, User> accountsByType = allAccounts.stream()
        .collect(Collectors.toMap(
            u -> u.getAccountType() == AccountType.LOCAL ? "LOCAL" : u.getProvider(),
            Function.identity()
        ));
    
    assertThat(accountsByType.keySet())
        .containsExactlyInAnyOrder("GOOGLE", "NAVER", "LOCAL");
}
```

---

## 8. 결론 및 배운 점

### 8-1. 주요 성과

1. **트랜잭션 안정성 확보**: UnexpectedRollbackException 해결로 시스템 안정성 향상
2. **일관된 계정 관리 정책 수립**: 계정 타입별 명확한 이메일 중복 검증 정책 확립
3. **사용자 경험 개선**: 실제 사용 패턴에 맞는 유연한 계정 생성 허용
4. **확장 가능한 아키텍처**: 향후 계정 통합 기능을 위한 기술적 기반 마련

### 8-2. 기술적 학습

**계정 관리 시스템의 복잡성**:
- 단순해 보이는 이메일 중복 검증도 계정 타입별로 다른 정책 적용 필요
- 소셜 로그인과 로컬 계정의 인증 메커니즘 차이를 고려한 설계 중요
- 사용자의 실제 사용 패턴과 기술적 제약 간 균형점 찾기

**트랜잭션 관리의 중요성**:
- 비즈니스 예외 발생 시 트랜잭션 상태 관리의 중요성 체감
- 롤백 전용 마킹과 UnexpectedRollbackException의 관계 이해
- 예외 처리 전략이 전체 시스템 안정성에 미치는 영향

**점진적 설계 개선**:
- 완벽한 초기 설계보다 실제 문제 해결 후 점진적 개선의 효과성
- 하위 호환성 유지하면서 정책 변경하는 방법론 습득
- 사용자 피드백을 통한 설계 가정 검증의 중요성

### 8-3. 프로세스 개선

**문제 해결 방법론**:

1. **현상 정확한 파악**: UnexpectedRollbackException이라는 표면적 증상에서 시작
2. **근본 원인 분석**: 계정 타입별 비대칭적 정책이라는 구조적 문제 발견
3. **사용자 관점 고려**: 기술적 제약보다 실제 사용 패턴 우선 고려
4. **점진적 해결**: 기존 기능 유지하면서 단계적 개선
5. **정책 명문화**: 암묵적 규칙을 명시적 정책으로 문서화

**설계 검증 체크리스트**:
- [ ] 계정 타입별로 일관된 정책이 적용되는가?
- [ ] 사용자의 실제 사용 패턴을 고려했는가?
- [ ] 예외 상황에서 시스템이 안정적으로 동작하는가?
- [ ] 향후 정책 변경에 유연하게 대응할 수 있는가?

### 8-4. 장기적 개선 방향

**계정 통합 시스템 구축**:

```java
// 1단계: 계정 연동 기능 (사용자 주도)
@Service
public class AccountLinkingService {
    
    public void linkAccounts(Long primaryUserId, Long secondaryUserId) {
        // 사용자가 자발적으로 계정 통합 요청
        User primary = userRepository.findById(primaryUserId).orElseThrow();
        User secondary = userRepository.findById(secondaryUserId).orElseThrow();
        
        if (!primary.getEmail().equals(secondary.getEmail())) {
            throw new IllegalArgumentException("다른 이메일의 계정은 연동할 수 없습니다.");
        }
        
        // 보조 계정의 데이터를 주 계정으로 이관
        mergeAccountData(primary, secondary);
        
        // 보조 계정을 연동됨 상태로 변경 (삭제하지 않음)
        secondary.linkTo(primary);
        userRepository.save(secondary);
    }
}

// 2단계: 스마트 로그인 (시스템 지원)
@Service
public class SmartLoginService {
    
    public LoginOptionsResponse getLoginOptions(String email) {
        List<User> accounts = userRepository.findByEmailAndActive(email);
        
        return LoginOptionsResponse.builder()
            .email(email)
            .hasLocalAccount(hasLocalAccount(accounts))
            .availableSocialProviders(getSocialProviders(accounts))
            .suggestedMethod(suggestOptimalLoginMethod(accounts))
            .build();
    }
}

// 3단계: 완전 통합 (정책 전환)
@Configuration
public class AccountPolicyV2 {
    
    /**
     * v2.0 정책: 이메일 기준 단일 계정, 다중 인증 방식
     * - 하나의 이메일당 하나의 사용자 계정
     * - 여러 인증 방식(로컬, 소셜) 연결 가능
     * - 기존 v1.0 계정은 점진적 마이그레이션
     */
    public static final boolean ENABLE_UNIFIED_ACCOUNT_POLICY = false; // 향후 활성화
}
```

**운영 모니터링 및 개선**:

```java
@Component
public class AccountManagementMetrics {
    
    @EventListener
    public void trackAccountCreationPattern(UserCreatedEvent event) {
        User user = event.getUser();
        
        // 계정 생성 패턴 분석
        meterRegistry.counter("account.creation", 
                             "type", user.getAccountType().name(),
                             "provider", Optional.ofNullable(user.getProvider()).orElse("none"))
                    .increment();
        
        // 동일 이메일 계정 수 추적
        long sameEmailCount = userRepository.countByEmailAndActive(user.getEmail());
        meterRegistry.gauge("account.same_email_count", sameEmailCount);
        
        if (sameEmailCount > 3) {
            alertService.sendAlert("동일 이메일 다중 계정 임계치 초과: " + user.getEmail());
        }
    }
}
```

### 8-5. 실무 적용 가이드

**인증 시스템 설계 시 고려사항**:

**정책 설계 단계**:
- [ ] 계정 타입별 이메일 중복 정책을 명확히 정의
- [ ] 사용자 실제 사용 패턴 조사 및 반영
- [ ] 향후 통합 시나리오를 고려한 확장 가능한 구조 설계
- [ ] 정책 변경 시 기존 사용자 영향도 분석 방안 수립

**기술 구현 단계**:
- [ ] Repository 레이어에서 계정 타입별 검증 메서드 분리
- [ ] Service 레이어에서 명확한 정책 적용
- [ ] 트랜잭션 예외 처리 시 롤백 상태 고려
- [ ] 동시성 문제 해결을 위한 적절한 격리 수준 설정

**운영 관리 단계**:
- [ ] 계정 생성 패턴 모니터링 시스템 구축
- [ ] 정책 위반 사항 자동 감지 및 알림
- [ ] 사용자 문의 대응을 위한 계정 조회 도구
- [ ] 정기적인 데이터 일관성 검증 및 보고

이러한 체계적인 접근을 통해 **계정 타입별 이메일 중복 검증 문제**를 근본적으로 해결하고, **사용자 친화적이면서도 기술적으로 안정적인** 인증 시스템을 구축할 수 있습니다. 특히 실제 사용자 패턴을 고려한 정책 설계는 다양한 인증 방식이 공존하는 현대적 웹 서비스에서 필수적인 접근 방식입니다.