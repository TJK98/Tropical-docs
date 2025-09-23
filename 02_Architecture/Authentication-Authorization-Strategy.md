# 인증/인가 전략 문서

본 문서는 **HttpOnly 쿠키 기반 JWT 인증**, **Spring Security Filter 기반 인가**를 중심으로 보안성·확장성·유지보수성을 고려한 설계 원칙과 구현 방식을 상세히 다룹니다. 또한 사용자 플로우, 라우트 구조, 보안 고려사항, 성능 최적화, 향후 개선 로드맵까지 포함하여 **팀 내 개발자와 신규 합류자들이 빠르게 이해하고 유지보수할 수 있도록** 작성되었습니다.

**작성자:** [왕택준](https://github.com/TJK98)

**문서 버전:** v1.0

**대상 독자:**

- **백엔드 개발자**: JWT, Spring Security Filter, 토큰 관리 로직을 구현하고 유지보수하는 개발자
- **프론트엔드 개발자**: React 기반 SPA에서 인증 흐름을 이해하고 API 통신 및 라우팅을 담당하는 개발자
- **DevOps / 인프라 엔지니어**: 환경변수 관리, HTTPS, 쿠키 설정, CORS 및 배포 환경에서의 보안을 관리하는 운영자
- **QA / 테스트 엔지니어**: 다양한 인증 시나리오(회원가입, 소셜 로그인, 토큰 만료/갱신 등)를 검증하는 품질 담당자
- **신규 합류자**: 프로젝트 구조와 공휴일 시스템의 전체적인 흐름을 빠르게 파악해야 하는 팀 신규 인원
- **프로덕트 매니저 (PM)**: 인증/인가 설계가 사용자 경험과 보안에 어떤 영향을 주는지 파악하고, 기획 방향을 설정하는 담당자

---

## 목차

1. [개요](#개요)
2. [토큰 저장 방식 비교 및 선택 근거](#토큰-저장-방식-비교-및-선택-근거)
3. [인증 검증 방식 비교 및 선택 근거](#인증-검증-방식-비교-및-선택-근거)
4. [JWT 토큰 체계](#jwt-토큰-체계)
5. [인증/인가 아키텍처](#인증인가-아키텍처)
6. [사용자 플로우](#사용자-플로우)
7. [라우트 구조](#라우트-구조)
8. [보안 고려사항](#보안-고려사항)
9. [성능 최적화](#성능-최적화)
10. [향후 개선사항](#향후-개선사항)

---

## 개요

TropiCal 프로젝트는 **HttpOnly 쿠키 기반 JWT 인증**과 **Spring Security Filter 방식**을 채택하여 웹 보안 베스트 프랙티스를 구현했습니다.

### 핵심 설계 원칙

- **보안 우선**: XSS 공격으로부터 토큰 보호
- **사용자 경험**: 자동 토큰 갱신으로 끊김 없는 서비스
- **확장성**: 소셜 로그인과 로컬 계정 통합 지원
- **유지보수성**: 명확한 책임 분리와 일관된 에러 처리

### 채택한 인증 방식

- **JWT (JSON Web Token) 기반 Stateless 인증**
- **HttpOnly 쿠키 + Authorization 헤더 하이브리드 방식**
- **OAuth2 소셜 로그인 통합 (Google, Kakao, Naver)**
- **Refresh Token을 통한 자동 토큰 갱신**

---

## 토큰 저장 방식 비교 및 선택 근거

### 1. 로컬 스토리지 vs HttpOnly 쿠키

| 구분 | 로컬 스토리지 | HttpOnly 쿠키 (채택) |
|------|---------------|---------------------|
| **XSS 취약성** | JavaScript로 접근 가능 | JavaScript 접근 차단 |
| **CSRF 취약성** | 자동 전송 안됨 | SameSite 설정으로 완화 |
| **구현 복잡도** | 간단 | 서버-클라이언트 협조 필요 |
| **토큰 관리** | 수동 관리 필요 | 브라우저 자동 관리 |
| **개발자 도구** | 쉽게 노출 | 개발자도구에서 보이지 않음 |
| **모바일 호환성** | 문제없음 | 문제없음 |

### 2. 우리가 HttpOnly 쿠키를 선택한 이유

#### 보안성 우선

```javascript
// 로컬 스토리지 - XSS 공격에 취약
localStorage.setItem('token', accessToken);
// 악의적 스크립트: localStorage.getItem('token') 으로 탈취 가능

// HttpOnly 쿠키 - JavaScript 접근 불가
// 서버에서 Set-Cookie: ACCESS_TOKEN=...; HttpOnly; Secure 설정
// 클라이언트에서는 document.cookie로도 접근 불가
```

#### 자동 토큰 관리

```java
// 서버에서 쿠키 설정 (CookieUtil.java)
public static Cookie build(String name, String value, int maxAgeSeconds) {
    Cookie cookie = new Cookie(name, value);
    cookie.setHttpOnly(true);    // JavaScript 접근 차단
    cookie.setPath("/");         // 전역 경로
    cookie.setMaxAge(maxAgeSeconds);
    return cookie;
}
```

#### 프론트엔드 코드 단순화

```javascript
// 쿠키 방식 - 토큰 관리 불필요
const apiClient = axios.create({
    withCredentials: true,  // 쿠키 자동 전송
});

// 로컬 스토리지 방식 - 매번 헤더 설정 필요
const token = localStorage.getItem('token');
const apiClient = axios.create({
    headers: { Authorization: `Bearer ${token}` }
});
```

---

## 인증 검증 방식 비교 및 선택 근거

### 1. Spring Security Filter vs Axios Interceptor

| 구분 | Axios Interceptor | Spring Security Filter (채택) |
|------|-------------------|------------------------------|
| **보안 레벨** | 클라이언트 사이드만 | 서버 사이드 강제 |
| **우회 가능성** | 개발자도구로 우회 가능 | 서버에서 강제 검증 |
| **토큰 갱신** | 클라이언트 주도 | 서버 주도 |
| **API 보호** | 직접 API 호출 시 무력화 | 모든 요청 강제 검증 |
| **에러 처리** | 네트워크 의존적 | 서버에서 일관된 처리 |
| **Spring 통합** | 별도 구현 필요 | 네이티브 통합 |

### 2. 우리가 Spring Security Filter를 선택한 이유

#### 서버 사이드 보안 강화

```java
// Filter 방식 - 모든 요청을 서버에서 강제 검증
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                   HttpServletResponse response,
                                   FilterChain filterChain) {
        // 쿠키에서 토큰 추출
        String token = extractTokenFromRequest(request);

        // 서버에서 강제 검증
        if (StringUtils.hasText(token) && jwtTokenProvider.validateToken(token)) {
            processAuthentication(request, token);
        }
    }
}
```

#### 포괄적 보안 적용

- **정적 리소스까지 보호**: `/css`, `/js` 등 정적 파일에도 인증 적용 가능
- **에러 페이지 보호**: 404, 500 에러 페이지도 인증 검사 통과
- **API 외부 요청 차단**: Controller에 매핑되지 않은 경로도 보안 적용

#### 이른 단계에서의 인증 실패 처리

```java
// Filter에서 인증 실패 시 즉시 401 응답
if (!isValidToken(token)) {
    response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
    response.getWriter().write("{\"error\":\"Unauthorized\"}");
    return; // Controller까지 가지 않고 즉시 차단
}
```

#### 토큰 자동 갱신 (서버 주도)

```java
// AuthController.java - 서버에서 쿠키 갱신
@PostMapping("/token/refresh")
public ResponseEntity<?> refreshToken(@CookieValue("REFRESH_TOKEN") String refreshToken,
                                     HttpServletResponse response) {
    String newAccessToken = jwtTokenProvider.createAccessToken(userId, email);

    // 새로운 쿠키 자동 설정
    response.addCookie(CookieUtil.build("ACCESS_TOKEN", newAccessToken, accessMaxAge));
    return ResponseEntity.ok(tokenResponse);
}
```

#### 클라이언트는 단순히 쿠키 리프레시만 호출

```javascript
// api.js - 클라이언트는 빈 객체만 전송
async function refreshCookieToken() {
    try {
        // HttpOnly 쿠키가 자동으로 전송되어 서버에서 처리
        await axios.post(`${API_ENDPOINT}/auth/token/refresh`, {}, {
            withCredentials: true
        });
        return true;
    } catch (error) {
        return false;
    }
}
```

---

## JWT 토큰 체계

### 1. 토큰 종류 및 역할

| 토큰 유형 | 유효기간 | 저장 위치 | 용도 | 보안 속성 |
|:---|:---|:---|:---|:---|
| **Access Token** | 15분 | HttpOnly 쿠키 | API 접근 권한 | HttpOnly, Secure, SameSite |
| **Refresh Token** | 14일 | HttpOnly 쿠키 | Access Token 갱신 | HttpOnly, Secure, SameSite |
| **Onboarding Token** | 30분 | HttpOnly 쿠키 | 소셜 로그인 후 온보딩 | HttpOnly, Secure, SameSite |
| **Email Verify Token** | 30분 | 이메일 링크 | 이메일 인증 | 일회성 사용 |

### 2. JWT 클레임 구조

```json
{
  "sub": "사용자 ID",
  "email": "사용자 이메일",
  "tokenType": "ACCESS|REFRESH|ONBOARDING|EMAIL_VERIFY",
  "iat": 1695801600,
  "exp": 1695802500
}
```

### 3. 토큰 타입별 권한 매핑

```java
// JwtAuthenticationFilter.java
private Collection<? extends GrantedAuthority> mapAuthorities(String tokenType) {
    return switch (tokenType) {
        case "ACCESS" -> List.of(new SimpleGrantedAuthority("ROLE_USER"));
        case "ONBOARDING" -> List.of(new SimpleGrantedAuthority("ROLE_ONBOARDING"));
        default -> List.of(); // 알 수 없는 토큰은 권한 없음
    };
}
```

---

## 인증/인가 아키텍처

### 1. 전체 아키텍처 다이어그램

```
[브라우저] ←--HTTP Cookie--> [Spring Boot Backend]
    ↓                              ↓
[React SPA]                [JwtAuthenticationFilter]
    ↓                              ↓
[Axios Interceptor]        [JwtTokenProvider]
    ↓                              ↓
[자동 토큰 갱신]              [SecurityContext 설정]
```

### 2. Spring Security 설정

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                // Public 엔드포인트
                .requestMatchers("/", "/api/auth/signup", "/api/auth/login",
                               "/api/auth/verify", "/api/auth/token/refresh",
                               "/login/**", "/oauth2/**").permitAll()

                // 온보딩 전용 엔드포인트
                .requestMatchers("/api/auth/onboarding").hasRole("ONBOARDING")

                // 일반 사용자 엔드포인트
                .anyRequest().hasRole("USER")
            )
            .oauth2Login(oauth2 -> oauth2
                .successHandler(oAuth2AuthenticationSuccessHandler)
            )
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }
}
```

### 3. 쿠키 보안 설정

```java
// CookieUtil.java
public static Cookie build(String name, String value, int maxAgeSeconds) {
    Cookie cookie = new Cookie(name, value);
    cookie.setHttpOnly(true);    // XSS 보호
    cookie.setPath("/");         // 전역 접근
    cookie.setMaxAge(maxAgeSeconds);
    // cookie.setSecure(true);   // HTTPS 전용 (운영 환경)
    // cookie.setSameSite("Strict"); // CSRF 보호
    return cookie;
}
```

---

## 사용자 플로우

### 1. 소셜 신규 회원

```
WelcomePage → OAuth 로그인 → OnboardingPage → Dashboard
     ↓              ↓              ↓           ↓
   소셜선택    Provider 인증    약관동의     정식토큰
                    ↓              ↓           ↓
              임시토큰발급     ONBOARDING    ACCESS
                             권한          권한
```

**구현 세부사항:**

```java
// OAuth2AuthenticationSuccessHandler.java
private String determineRedirectUrl(User user, boolean isNewUser) {
    if (isNewUser || !user.isOnboardingCompleted()) {
        return frontendBaseUrl + "/onboarding";  // 온보딩 필요
    } else {
        return frontendBaseUrl + "/dashboard";   // 바로 대시보드
    }
}
```

### 2. 로컬 신규 회원

```
WelcomePage → SignupPage → 이메일인증 → LoginPage → Dashboard
     ↓            ↓           ↓           ↓         ↓
   회원가입     약관+정보    인증메일     로그인    ACCESS
   버튼클릭      입력        발송        성공      권한
```

**구현 세부사항:**

```java
// AuthController.java - 회원가입
@PostMapping("/signup")
public ResponseEntity<?> signup(@Valid @RequestBody SignupRequest request) {
    // 1. 사용자 생성 (emailVerified=false)
    User user = userService.createLocalUser(email, password, nickname);

    // 2. 온보딩 완료 처리 (로컬은 회원가입 시 약관 동의)
    userConsentService.processOnboardingConsents(user.getId(), consentData);
    userService.completeOnboarding(user.getId());

    // 3. 이메일 인증 토큰 발송
    String emailVerifyToken = jwtTokenProvider.createEmailVerifyToken(user.getId(), email);
    emailService.sendVerificationMail(email, verifyUrl);

    // 로그인 토큰은 발급하지 않음 - 이메일 인증 후 로그인 필요
    return ResponseEntity.ok("이메일 인증을 완료해 주세요");
}
```

### 3. 기존 회원 (소셜/로컬)

```
WelcomePage → 로그인 → Dashboard
     ↓          ↓        ↓
   로그인     인증성공   ACCESS
   버튼                권한
```

### 4. API 요청 및 토큰 갱신 플로우

```
Frontend → API 요청 (Access Token 쿠키 포함) → Backend

Access Token 유효 → 정상 응답
Access Token 만료 → 401 Unauthorized → Interceptor
                                    ↓
                   POST /auth/token/refresh (Refresh Token 쿠키)
                                    ↓
                   Refresh Token 검증 → 새 Access Token 쿠키 설정
                                    ↓
                   원래 API 재요청 → 정상 응답
```

---

## 라우트 구조

### 1. 프론트엔드 라우트 매핑

| 경로 | 컴포넌트 | 권한 | 설명 |
|------|----------|------|------|
| `/` | WelcomePage | Public | 서비스 소개 및 로그인/회원가입 |
| `/login` | LoginPage | Public | 로컬 계정 로그인 |
| `/signup` | SignupPage | Public | 로컬 계정 회원가입 |
| `/onboarding` | OnboardingPage | ONBOARDING | 소셜 로그인 후 약관 동의 |
| `/verify-required` | VerifyRequiredPage | Public | 이메일 인증 안내 |
| `/verified` | VerifiedPage | Public | 이메일 인증 성공 |
| `/verify-failed` | VerifyFailedPage | Public | 이메일 인증 실패 |
| `/dashboard` | CalendarPage | USER | 메인 캘린더 (기본) |
| `/dashboard/todo` | TodoPage | USER | 투두 관리 |
| `/dashboard/bucket` | BucketPage | USER | 버킷리스트 |

### 2. 프론트엔드 라우트 가드

```javascript
// ProtectedRoute.jsx
function ProtectedRoute({ children, requiredRole = 'USER' }) {
    const { user, isLoading } = useAuth();

    if (isLoading) return <LoadingSpinner />;

    if (!user) {
        return <Navigate to="/" replace />;
    }

    // 온보딩 미완료 시 온보딩으로 리다이렉트
    if (requiredRole === 'USER' && !user.onboardingCompleted) {
        return <Navigate to="/onboarding" replace />;
    }

    return children;
}
```

---

## 보안 고려사항

### 1. XSS 공격 방지

- **HttpOnly 쿠키**: JavaScript에서 토큰 접근 불가
- **CSP 헤더**: 인라인 스크립트 실행 차단
- **입력값 검증**: 모든 사용자 입력 서버에서 검증

### 2. CSRF 공격 방지

- **SameSite 쿠키**: 크로스 사이트 요청 제한
- **CORS 설정**: 허용된 도메인만 API 접근
- **Referer 검증**: 요청 출처 확인

### 3. 토큰 보안

- **짧은 만료시간**: Access Token 15분
- **자동 갱신**: Refresh Token 14일
- **토큰 타입 구분**: 용도별 권한 분리

### 4. 네트워크 보안

```yaml
# application.yml - CORS 설정
cors:
  allowed-origins:
    - http://localhost:5005
  allowed-methods:
    - GET, POST, PUT, DELETE, OPTIONS
  allow-credentials: true
```

### 5. 동시성 제어 (토큰 갱신)

```javascript
// 중복 리프레시 요청 방지
let isRefreshing = false;
let waiters = [];

async function refreshCookieToken() {
    if (isRefreshing) {
        return new Promise(resolve => waiters.push(resolve));
    }

    isRefreshing = true;
    try {
        await performRefresh();
        waiters.forEach(resolve => resolve(true));
    } finally {
        waiters = [];
        isRefreshing = false;
    }
}
```

---

## 성능 최적화

### 1. JWT 서명 검증 최적화

```java
// JwtAuthenticationFilter.java - 중복 파싱 방지
private void processAuthentication(HttpServletRequest request, String token) {
    // 이미 인증된 컨텍스트면 재설정 금지 (중복 필터 진입 방지, 성능 향상)
    if (SecurityContextHolder.getContext().getAuthentication() != null) {
        return;
    }

    // 중복 파싱 방지: 한 번만 Claims 추출하고 캐시해서 사용
    Claims claims = jwtTokenProvider.parseClaims(token);
    String tokenType = String.valueOf(claims.get("tokenType"));
    Long userId = Long.valueOf(claims.getSubject());
    String email = claims.get("email", String.class);
}
```

### 2. 조기 인증 실패 처리

```java
// Filter에서 잘못된 토큰을 조기에 차단하여
// 불필요한 Controller 로직 실행을 방지
protected boolean shouldNotFilter(HttpServletRequest request) {
    String path = request.getRequestURI();

    // JWT 필터를 적용하지 않을 경로들 (Public 엔드포인트)
    return path.equals("/") ||
            path.startsWith("/api/auth/signup") ||
            path.startsWith("/api/auth/login");
}
```

### 3. 동시성 제어로 중복 요청 방지

```javascript
// api.js - 토큰 갱신 동시성 제어
const RETRY_KEY = 'x-retried';
let isRefreshing = false;
let waiters = [];

async function refreshCookieToken() {
    // 리팩토링 포인트: 이미 리프레시 중이면 대기열 추가
    if (isRefreshing) {
        return new Promise((resolve) => waiters.push(resolve));
    }

    isRefreshing = true;
    // 첫 번째 요청만 실제 갱신 수행, 나머지는 결과 공유
}
```

---

## 향후 개선사항

### 1. 단기 계획 (3개월)

#### 보안 강화

```java
// 토큰 무효화 API 구현
@PostMapping("/auth/invalidate")
public ResponseEntity<?> invalidateUserTokens(@RequestParam Long userId) {
    // Redis 블랙리스트에 사용자의 모든 토큰 추가
    tokenBlacklistService.invalidateAllUserTokens(userId);
    return ResponseEntity.ok("모든 토큰이 무효화되었습니다");
}

// 로그인 히스토리 및 의심 활동 탐지
@Entity
public class LoginHistory {
    private Long userId;
    private String ipAddress;
    private String userAgent;
    private LocalDateTime loginTime;
    private Boolean suspicious; // 이상 패턴 감지
}
```

#### 2FA 인증 추가

```java
// TOTP 기반 이중 인증
@PostMapping("/auth/2fa/enable")
public ResponseEntity<?> enable2FA() {
    String secret = totpService.generateSecret();
    String qrCodeUrl = totpService.generateQRCode(secret);

    return ResponseEntity.ok(Map.of(
        "secret", secret,
        "qrCode", qrCodeUrl
    ));
}

@PostMapping("/auth/2fa/verify")
public ResponseEntity<?> verify2FA(@RequestParam String code) {
    boolean isValid = totpService.verifyCode(getCurrentUserId(), code);

    if (isValid) {
        userService.enable2FA(getCurrentUserId());
        return ResponseEntity.ok("2FA가 활성화되었습니다");
    }

    return ResponseEntity.badRequest().body("인증 코드가 올바르지 않습니다");
}
```

### 2. 중기 계획 (6개월)

#### SSO 통합 (SAML/OIDC)

```java
// 기업용 SSO 지원
@Configuration
public class SamlConfig {

    @Bean
    public RelyingPartyRegistrationRepository relyingPartyRegistrations() {
        return InMemoryRelyingPartyRegistrationRepository.builder()
                .relyingPartyRegistration(
                    RelyingPartyRegistration.withRegistrationId("corporate-sso")
                        .assertingPartyDetails(party -> party
                            .entityId("https://corporate.example.com/saml/metadata")
                            .singleSignOnServiceLocation("https://corporate.example.com/saml/sso")
                            .wantAuthnRequestsSigned(false)
                        )
                        .build()
                )
                .build();
    }
}
```

#### 디바이스 관리 시스템

```java
@Entity
public class UserDevice {
    private Long id;
    private Long userId;
    private String deviceId;       // 디바이스 고유 식별자
    private String deviceName;     // 사용자 지정 이름
    private String fingerprint;    // 브라우저 fingerprint
    private Boolean trusted;       // 신뢰할 수 있는 디바이스 여부
    private LocalDateTime lastUsed;
    private LocalDateTime registeredAt;
}

// 디바이스 기반 인증
@PostMapping("/auth/device/register")
public ResponseEntity<?> registerDevice(@RequestBody DeviceRegistrationRequest request) {
    String deviceFingerprint = deviceService.generateFingerprint(
        request.getUserAgent(),
        request.getScreenResolution(),
        request.getTimezone()
    );

    UserDevice device = userDeviceService.registerDevice(
        getCurrentUserId(),
        deviceFingerprint,
        request.getDeviceName()
    );

    return ResponseEntity.ok("디바이스가 등록되었습니다");
}
```

#### 리스크 기반 인증

```java
@Service
public class RiskAssessmentService {

    public RiskLevel assessLoginRisk(LoginAttempt attempt) {
        RiskScore score = new RiskScore();

        // IP 주소 기반 위험도 평가
        if (isFromSuspiciousLocation(attempt.getIpAddress())) {
            score.add(RiskFactor.SUSPICIOUS_LOCATION, 30);
        }

        // 시간대 기반 위험도 평가
        if (isUnusualLoginTime(attempt.getUserId(), attempt.getTimestamp())) {
            score.add(RiskFactor.UNUSUAL_TIME, 20);
        }

        // 디바이스 기반 위험도 평가
        if (!isKnownDevice(attempt.getUserId(), attempt.getDeviceFingerprint())) {
            score.add(RiskFactor.UNKNOWN_DEVICE, 40);
        }

        return score.calculateLevel();
    }
}
```

### 3. 장기 계획 (1년)

#### Zero Trust 아키텍처

```java
@Component
public class ContinuousAuthenticationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                   HttpServletResponse response,
                                   FilterChain filterChain) {

        // 모든 요청에 대한 지속적 검증
        AuthenticationContext context = buildAuthenticationContext(request);

        // 행동 패턴 분석
        BehaviorAnalysis behavior = behaviorAnalyzer.analyze(context);

        // 위험도 재평가
        RiskLevel currentRisk = riskService.reassessRisk(context, behavior);

        if (currentRisk.isHigh()) {
            // 추가 인증 요구
            requireStepUpAuthentication(response);
            return;
        }

        filterChain.doFilter(request, response);
    }
}
```

#### 생체 인증 (WebAuthn FIDO2)

```java
@RestController
@RequestMapping("/api/webauthn")
public class WebAuthnController {

    @PostMapping("/register/begin")
    public ResponseEntity<?> beginRegistration() {
        PublicKeyCredentialCreationOptions options =
            webAuthnService.startRegistration(getCurrentUserId());

        return ResponseEntity.ok(options);
    }

    @PostMapping("/register/finish")
    public ResponseEntity<?> finishRegistration(@RequestBody AuthenticatorAttestationResponse response) {
        boolean success = webAuthnService.finishRegistration(
            getCurrentUserId(),
            response
        );

        if (success) {
            return ResponseEntity.ok("생체 인증이 등록되었습니다");
        }

        return ResponseEntity.badRequest().body("등록에 실패했습니다");
    }

    @PostMapping("/authenticate")
    public ResponseEntity<?> authenticate(@RequestBody AuthenticatorAssertionResponse response) {
        boolean isValid = webAuthnService.verifyAssertion(response);

        if (isValid) {
            String accessToken = generateBiometricToken(getCurrentUserId());
            return ResponseEntity.ok(Map.of("token", accessToken));
        }

        return ResponseEntity.badRequest().body("인증에 실패했습니다");
    }
}
```

#### AI 기반 이상 탐지

```java
@Service
public class AnomalyDetectionService {

    private final MachineLearningModel behaviorModel;

    public AnomalyScore detectAnomaly(UserSession session) {
        // 사용자 행동 패턴 벡터화
        BehaviorVector vector = vectorizer.vectorize(
            session.getClickPatterns(),
            session.getNavigationFlow(),
            session.getTypingSpeed(),
            session.getMouseMovements()
        );

        // ML 모델로 이상 점수 계산
        double anomalyScore = behaviorModel.predict(vector);

        // 임계값 초과 시 알림
        if (anomalyScore > ANOMALY_THRESHOLD) {
            securityEventService.raiseAlert(
                SecurityEvent.BEHAVIOR_ANOMALY,
                session.getUserId(),
                anomalyScore
            );
        }

        return new AnomalyScore(anomalyScore, ANOMALY_THRESHOLD);
    }
}
```

### 4. 기술적 부채 해결

#### JWT 서명 검증 캐싱

```java
@Service
public class CachedJwtTokenProvider {

    private final RedisTemplate<String, Boolean> redisTemplate;
    private static final String TOKEN_CACHE_PREFIX = "jwt:valid:";
    private static final Duration CACHE_TTL = Duration.ofMinutes(5);

    public boolean validateTokenWithCache(String token) {
        String cacheKey = TOKEN_CACHE_PREFIX + DigestUtils.sha256Hex(token);

        // Redis 캐시에서 검증 결과 조회
        Boolean cachedResult = redisTemplate.opsForValue().get(cacheKey);
        if (cachedResult != null) {
            return cachedResult;
        }

        // 캐시 미스 시 실제 검증 수행
        boolean isValid = validateToken(token);

        // 검증 결과를 캐시에 저장 (5분 TTL)
        redisTemplate.opsForValue().set(cacheKey, isValid, CACHE_TTL);

        return isValid;
    }
}
```

#### Refresh Token Rotation

```java
@Service
public class RotatingRefreshTokenService {

    @Transactional
    public TokenPair refreshTokens(String oldRefreshToken) {
        // 기존 Refresh Token 검증
        if (!jwtTokenProvider.validateToken(oldRefreshToken)) {
            throw new InvalidTokenException("유효하지 않은 Refresh Token");
        }

        Long userId = jwtTokenProvider.getUserIdFromToken(oldRefreshToken);
        User user = userService.findById(userId);

        // 새로운 토큰 쌍 생성
        String newAccessToken = jwtTokenProvider.createAccessToken(user.getId(), user.getEmail());
        String newRefreshToken = jwtTokenProvider.createRefreshToken(user.getId());

        // 기존 Refresh Token 블랙리스트 추가
        tokenBlacklistService.addToBlacklist(oldRefreshToken);

        return new TokenPair(newAccessToken, newRefreshToken);
    }
}
```

#### 토큰 블랙리스트 시스템

```java
@Service
public class TokenBlacklistService {

    private final RedisTemplate<String, String> redisTemplate;
    private static final String BLACKLIST_PREFIX = "blacklist:";

    public void addToBlacklist(String token) {
        try {
            Claims claims = jwtTokenProvider.parseClaims(token);
            long remainingTtl = claims.getExpiration().getTime() - System.currentTimeMillis();

            if (remainingTtl > 0) {
                String key = BLACKLIST_PREFIX + DigestUtils.sha256Hex(token);
                redisTemplate.opsForValue().set(key, "blacklisted",
                    Duration.ofMillis(remainingTtl));
            }
        } catch (Exception e) {
            log.warn("블랙리스트 추가 실패: {}", e.getMessage());
        }
    }

    public boolean isBlacklisted(String token) {
        String key = BLACKLIST_PREFIX + DigestUtils.sha256Hex(token);
        return redisTemplate.hasKey(key);
    }
}
```

### 5. 모니터링 및 운영 개선

#### 인증 이벤트 모니터링

```java
@Component
public class AuthenticationMetrics {

    private final MeterRegistry meterRegistry;
    private final Counter loginSuccessCounter;
    private final Counter loginFailureCounter;
    private final Timer tokenRefreshTimer;

    @EventListener
    public void handleAuthenticationSuccess(AuthenticationSuccessEvent event) {
        loginSuccessCounter.increment(
            Tags.of(
                "provider", getAuthProvider(event),
                "user_type", getUserType(event)
            )
        );

        // 로그인 성공률 대시보드에 표시
        Gauge.builder("auth.success_rate")
            .register(meterRegistry, this, this::calculateSuccessRate);
    }

    @EventListener
    public void handleTokenRefresh(TokenRefreshEvent event) {
        Timer.Sample sample = Timer.start(meterRegistry);
        sample.stop(tokenRefreshTimer);

        // 토큰 갱신 지연시간 모니터링
        if (event.getDuration().toMillis() > 1000) {
            log.warn("토큰 갱신 지연 감지 - 사용자: {}, 지연시간: {}ms",
                    event.getUserId(), event.getDuration().toMillis());
        }
    }
}
```

#### 보안 대시보드

```java
@RestController
@RequestMapping("/api/admin/security")
@PreAuthorize("hasRole('ADMIN')")
public class SecurityDashboardController {

    @GetMapping("/stats")
    public ResponseEntity<SecurityStats> getSecurityStats() {
        SecurityStats stats = SecurityStats.builder()
            .activeUsers(userService.getActiveUserCount())
            .failedLoginAttempts(authService.getFailedLoginCount24h())
            .suspiciousActivities(securityService.getSuspiciousActivityCount())
            .tokenRefreshRate(tokenService.getRefreshRate())
            .build();

        return ResponseEntity.ok(stats);
    }

    @GetMapping("/threats")
    public ResponseEntity<List<SecurityThreat>> getSecurityThreats() {
        List<SecurityThreat> threats = securityService.getRecentThreats();
        return ResponseEntity.ok(threats);
    }
}
```

### 6. 성능 및 확장성 개선

#### Connection Pool 최적화

```java
@Configuration
public class HttpClientConfig {

    @Bean
    public RestTemplate oAuthRestTemplate() {
        HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory();

        // OAuth Provider와의 연결 최적화
        factory.setConnectTimeout(3000);
        factory.setReadTimeout(10000);
        factory.setConnectionRequestTimeout(1000);

        // Connection Pool 설정
        PoolingHttpClientConnectionManager connectionManager =
            new PoolingHttpClientConnectionManager();
        connectionManager.setMaxTotal(100);
        connectionManager.setDefaultMaxPerRoute(20);

        CloseableHttpClient httpClient = HttpClientBuilder.create()
            .setConnectionManager(connectionManager)
            .build();

        factory.setHttpClient(httpClient);

        return new RestTemplate(factory);
    }
}
```

#### 장애 대응 개선

```java
@Component
public class AuthenticationCircuitBreaker {

    private final CircuitBreaker oauthCircuitBreaker;

    @CircuitBreaker(name = "oauth-provider", fallbackMethod = "fallbackAuthentication")
    public AuthenticationResult authenticateWithProvider(String provider, String code) {
        // OAuth Provider와 통신
        return oauthService.authenticate(provider, code);
    }

    public AuthenticationResult fallbackAuthentication(String provider, String code, Exception ex) {
        log.warn("OAuth Provider 장애 감지 - provider: {}, 오류: {}", provider, ex.getMessage());

        // Graceful Degradation: 읽기 전용 모드 제공
        return AuthenticationResult.builder()
            .success(false)
            .fallbackMode(true)
            .message("현재 " + provider + " 로그인에 일시적 문제가 있습니다. 이메일 로그인을 이용해주세요.")
            .build();
    }
}
```

### 7. 면접 질문 대응 가이드

#### Q1: "현재 구현한 인증 시스템의 한계점은 무엇인가요?"

**A**: "현재 시스템의 주요 한계점은 세 가지입니다. 첫째, 토큰 탈취 시 즉시 무효화 기능이 없어 Access Token 만료까지 기다려야 합니다. 둘째, 단일 디바이스 인증만 지원하여 멀티 디바이스 환경에서 제약이 있습니다. 셋째, 정적인 보안 정책으로 사용자 행동 패턴에 따른 동적 보안 조치가 부족합니다. 이를 해결하기 위해 토큰 블랙리스트, 디바이스 관리, AI 기반 이상 탐지 시스템을 단계적으로 도입할 계획입니다."

#### Q2: "확장성 측면에서 어떤 개선이 필요한가요?"

**A**: "현재는 단일 서버 환경에 최적화되어 있어 MSA 환경으로 확장 시 몇 가지 개선이 필요합니다. JWT 서명 검증을 Redis로 캐싱하여 성능을 향상시키고, 토큰 블랙리스트를 중앙화하여 서비스 간 일관성을 보장해야 합니다. 또한 OAuth Provider와의 Connection Pool을 최적화하고 Circuit Breaker 패턴을 적용하여 외부 의존성 장애에 대한 복원력을 높일 계획입니다."

#### Q3: "보안 측면에서 추가로 고려할 점은?"

**A**: "현재 기본적인 XSS, CSRF 방어는 구현되어 있지만, 더 고도화된 보안이 필요합니다. Refresh Token Rotation으로 토큰 탈취 위험을 줄이고, 디바이스 기반 인증으로 알려지지 않은 디바이스 접근을 제한할 예정입니다. 장기적으로는 Zero Trust 아키텍처를 도입하여 모든 요청을 지속적으로 검증하고, WebAuthn을 통한 생체 인증으로 패스워드 의존성을 줄여나갈 계획입니다."

---

이러한 향후 개선사항들은 TropiCal 프로젝트의 보안성, 사용자 경험, 확장성을 체계적으로 발전시키기 위한 로드맵입니다. 단기적으로는 기본적인 보안 강화에 집중하고, 중장기적으로는 AI와 생체 인증 등 차세대 기술을 도입하여 미래지향적인 인증 시스템을 구축할 예정입니다.
