# 인증/인가 전략 문서

본 문서는 **HttpOnly 쿠키 기반 JWT 인증**, **Spring Security Filter 기반 인가**를 중심으로 보안성·확장성·유지보수성을 고려한 설계 원칙과 구현 방식을 상세히 다룹니다. 또한 사용자 플로우, 라우트 구조, 보안 고려사항, 성능 최적화, 향후 개선 로드맵까지 포함하여 **팀 내 개발자와 신규 합류자들이 빠르게 이해하고 유지보수할 수 있도록** 작성되었습니다.

**작성자:** [왕택준](https://github.com/TJK98)

**문서 버전:** v1.0

**대상 독자:**

- **백엔드 개발자**: JWT, Spring Security Filter, 토큰 관리 로직을 구현하고 유지보수하는 개발자
- **프론트엔드 개발자**: React 기반 SPA에서 인증 흐름을 이해하고 API 통신 및 라우팅을 담당하는 개발자
- **DevOps / 인프라 엔지니어**: 환경변수 관리, HTTPS, 쿠키 설정, CORS 및 배포 환경에서의 보안을 관리하는 운영자
- **QA / 테스트 엔지니어**: 다양한 인증 시나리오(회원가입, 소셜 로그인, 토큰 만료/갱신 등)를 검증하는 품질 담당자
- **신규 합류자**: 프로젝트 구조와 인증 시스템의 전체적인 흐름을 빠르게 파악해야 하는 팀 신규 인원
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
- **네트워크 호환성**: IPv4 주소 접속과 localhost 모두 지원

### 채택한 인증 방식

- **JWT (JSON Web Token) 기반 Stateless 인증**
- **HttpOnly 쿠키 + Authorization 헤더 하이브리드 방식**
- **OAuth2 소셜 로그인 통합 (Google, Kakao, Naver)**
- **Refresh Token을 통한 자동 토큰 갱신**
- **동적 URL 시스템으로 다중 개발 환경 지원**

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

---

## JWT 토큰 체계

### 1. 토큰 종류 및 역할

| 토큰 유형 | 유효기간 | 저장 위치 | 용도 | 발급 조건 |
|:---|:---|:---|:---|:---|
| **Access Token** | 15분 | HttpOnly 쿠키 | API 접근 권한 | 로그인/온보딩 완료 시 |
| **Refresh Token** | 14일 | HttpOnly 쿠키 | Access Token 갱신 | 로그인/온보딩 완료 시 |
| **Onboarding Token** | 30분 | HttpOnly 쿠키 | 온보딩 API 전용 | 소셜 로그인 직후 (온보딩 미완료 시만) |
| **Email Verify Token** | 30분 | 이메일 링크 URL | 이메일 인증 | 회원가입/재발송 시 |

### 2. 온보딩 토큰의 필요성과 생명주기

#### 왜 온보딩 토큰이 필요한가?

소셜 로그인 사용자는 두 단계로 나뉩니다:
1. **소셜 인증 완료** (OAuth Provider에서 사용자 확인)
2. **서비스 온보딩 완료** (약관 동의, 추가 정보 입력)

온보딩 토큰은 이 중간 단계에서 **제한된 권한**만 부여합니다:

```java
// OAuth2AuthenticationSuccessHandler.java
if (!user.isOnboardingCompleted()) {
    // 소셜 인증은 성공했지만 온보딩이 필요한 상태
    // 제한된 권한의 ONBOARDING_TOKEN 발급
    String onboardingToken = jwtTokenProvider.createOnboardingToken(user.getId(), user.getEmail());
    response.addCookie(CookieUtil.build("ONBOARDING_TOKEN", onboardingToken, 1800));
    
    // /onboarding 페이지로 리다이렉트
    redirectUrl = frontendBaseUrl + "/onboarding";
} else {
    // 이미 온보딩 완료된 사용자는 바로 정식 토큰 발급
    String accessToken = jwtTokenProvider.createAccessToken(user.getId(), user.getEmail());
    String refreshToken = jwtTokenProvider.createRefreshToken(user.getId());
    
    // 정식 토큰으로 쿠키 설정
    response.addCookie(CookieUtil.build("ACCESS_TOKEN", accessToken, accessMaxAge));
    response.addCookie(CookieUtil.build("REFRESH_TOKEN", refreshToken, refreshMaxAge));
    
    // 바로 대시보드로 리다이렉트
    redirectUrl = frontendBaseUrl + "/dashboard";
}
```

#### 온보딩 토큰의 권한 제한

```java
// SecurityConfig.java
.authorizeHttpRequests(auth -> auth
    // 온보딩 토큰으로만 접근 가능
    .requestMatchers("/api/v1/auth/onboarding").hasRole("ONBOARDING")
    
    // 일반 API는 정식 ACCESS 토큰 필요
    .anyRequest().hasRole("USER")
)
```

#### 온보딩 완료 시 토큰 교체

```java
// AuthController.java - completeSocialOnboarding 메서드
@PostMapping("/onboarding")
public ResponseEntity<?> completeSocialOnboarding(@RequestBody OnboardingRequest request,
                                                 HttpServletResponse response) {
    // 1. 온보딩 처리 (약관 동의 등)
    userConsentService.processOnboardingConsents(userId, request.getAllConsents());
    userService.completeOnboarding(userId);

    // 2. 기존 ONBOARDING_TOKEN 제거 (보안상 중요)
    response.addCookie(CookieUtil.expire("ONBOARDING_TOKEN"));

    // 3. 정식 ACCESS/REFRESH TOKEN 발급
    String accessToken = jwtTokenProvider.createAccessToken(user.getId(), user.getEmail());
    String refreshToken = jwtTokenProvider.createRefreshToken(user.getId());
    
    response.addCookie(CookieUtil.build("ACCESS_TOKEN", accessToken, accessMaxAge));
    response.addCookie(CookieUtil.build("REFRESH_TOKEN", refreshToken, refreshMaxAge));

    return ResponseEntity.ok("온보딩이 완료되었습니다");
}
```

### 3. 이메일 인증 토큰이 URL에 있는 이유

이메일 인증 토큰은 **의도적으로 쿠키가 아닌 URL 파라미터**로 전송됩니다:

#### 설계 근거
1. **크로스 디바이스 지원**: 다른 기기에서 이메일 확인 가능
2. **이메일 클라이언트 독립성**: 웹메일, 모바일 앱 등 어디서든 동작
3. **일회성 사용**: URL 클릭 시 즉시 검증하고 무효화

```java
// AuthController.java - signup 메서드에서 동적 URL 생성
@PostMapping("/signup")
public ResponseEntity<?> signup(@RequestBody SignupRequest request,
                               HttpServletRequest httpRequest) {
    // 동적 백엔드 URL 생성 (IPv4 주소 지원)
    String backendUrl = getBackendBaseUrl(httpRequest);
    String verifyUrl = backendUrl + "/api/v1/auth/verify?token=" + emailVerifyToken;
    
    emailService.sendVerificationMail(user.getEmail(), verifyUrl);
}

private String getBackendBaseUrl(HttpServletRequest request) {
    // 1순위: 고정 도메인 → 2순위: 동적 호스트 → 3순위: localhost
    if (!backendDomain.isEmpty()) return backendDomain;
    if (useDynamicHost) {
        return String.format("%s://%s:%d", 
            request.getScheme(), request.getServerName(), request.getServerPort());
    }
    return backendBaseUrl; // fallback
}
```

### 4. JWT 클레임 구조

```json
{
  "sub": "사용자 ID",
  "email": "사용자 이메일",
  "tokenType": "ACCESS|REFRESH|ONBOARDING|EMAIL_VERIFY",
  "iat": 1695801600,
  "exp": 1695802500
}
```

### 5. 토큰 타입별 권한 매핑

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
[브라우저] ←--HttpOnly Cookie--> [Spring Boot Backend]
    ↓                                ↓
[React SPA]                    [JwtAuthenticationFilter]
    ↓                                ↓
[Axios Interceptor]            [JwtTokenProvider]
    ↓                                ↓
[자동 토큰 갱신]                [SecurityContext 설정]
    ↓                                ↓
[동적 URL 지원]              [환경변수 기반 URL 생성]
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
                .requestMatchers("/", "/api/v1/auth/signup", "/api/v1/auth/login",
                               "/api/v1/auth/verify", "/api/v1/auth/token/refresh",
                               "/login/**", "/oauth2/**").permitAll()

                // 온보딩 전용 엔드포인트 (ONBOARDING_TOKEN 필요)
                .requestMatchers("/api/v1/auth/onboarding").hasRole("ONBOARDING")

                // 일반 사용자 엔드포인트 (ACCESS_TOKEN 필요)
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

### 4. 동적 URL 지원

```java
// OAuth2AuthenticationSuccessHandler.java
private String getFrontendBaseUrl(HttpServletRequest request) {
    // 1순위: 운영환경 고정 도메인
    if (!frontendDomain.isEmpty()) {
        return frontendDomain;
    }
    
    // 2순위: 개발환경 동적 호스트 (IPv4 주소 지원)
    if (useDynamicHost) {
        String scheme = request.getScheme();
        String serverName = request.getServerName();
        return String.format("%s://%s:%s", scheme, serverName, frontendPort);
    }
    
    // 3순위: localhost 폴백
    return String.format("http://localhost:%s", frontendPort);
}
```

---

## 사용자 플로우

### 1. 소셜 신규 회원 (온보딩 토큰 사용)

```
WelcomePage → OAuth 로그인 → OnboardingPage → Dashboard
     ↓              ↓              ↓           ↓
   소셜선택    Provider 인증    약관동의     정식토큰
                    ↓              ↓           ↓
            ONBOARDING_TOKEN   토큰교체    ACCESS_TOKEN
               (제한권한)      (완료시)     (전체권한)
```

**구현 세부사항:**

```java
// OAuth2AuthenticationSuccessHandler.java
private String determineRedirectUrl(User user, boolean isNewUser) {
    if (isNewUser || !user.isOnboardingCompleted()) {
        // ONBOARDING_TOKEN 발급 후 온보딩 페이지로
        String onboardingToken = jwtTokenProvider.createOnboardingToken(user.getId(), user.getEmail());
        response.addCookie(CookieUtil.build("ONBOARDING_TOKEN", onboardingToken, 1800));
        return frontendBaseUrl + "/onboarding";
    } else {
        // 이미 온보딩 완료된 사용자는 바로 정식 토큰
        String accessToken = jwtTokenProvider.createAccessToken(user.getId(), user.getEmail());
        String refreshToken = jwtTokenProvider.createRefreshToken(user.getId());
        response.addCookie(CookieUtil.build("ACCESS_TOKEN", accessToken, accessMaxAge));
        response.addCookie(CookieUtil.build("REFRESH_TOKEN", refreshToken, refreshMaxAge));
        return frontendBaseUrl + "/dashboard";
    }
}
```

### 2. 로컬 신규 회원 (이메일 인증)

```
WelcomePage → SignupPage → 이메일인증 → LoginPage → Dashboard
     ↓            ↓           ↓           ↓         ↓
   회원가입     약관+정보    URL토큰      로그인    ACCESS
   버튼클릭      입력      (이메일)       성공      권한
```

**구현 세부사항:**

```java
// AuthController.java - 회원가입 (동적 URL 적용)
@PostMapping("/signup")
public ResponseEntity<?> signup(@Valid @RequestBody SignupRequest request,
                               HttpServletRequest httpRequest) {
    // 1. 사용자 생성 (emailVerified=false)
    User user = userService.createLocalUser(email, password, nickname);

    // 2. 로컬 계정은 회원가입 시 약관 동의하므로 온보딩 즉시 완료
    userConsentService.processOnboardingConsents(user.getId(), consentData);
    userService.completeOnboarding(user.getId());

    // 3. 동적 백엔드 URL 생성하여 이메일 인증 토큰 발송
    String backendUrl = getBackendBaseUrl(httpRequest);
    String emailVerifyToken = jwtTokenProvider.createEmailVerifyToken(user.getId(), email);
    String verifyUrl = backendUrl + "/api/v1/auth/verify?token=" + emailVerifyToken;
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
| `/onboarding` | OnboardingPage | ONBOARDING | 소셜 로그인 후 약관 동의 (ONBOARDING_TOKEN 필요) |
| `/verify-required` | VerifyRequiredPage | Public | 이메일 인증 안내 |
| `/verified` | VerifiedPage | Public | 이메일 인증 성공 |
| `/verify-failed` | VerifyFailedPage | Public | 이메일 인증 실패 |
| `/dashboard` | CalendarPage | USER | 메인 캘린더 (ACCESS_TOKEN 필요) |
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
- **온보딩 토큰 생명주기 관리**: 온보딩 완료 시 즉시 제거

### 4. 이메일 인증 토큰 보안

```java
// 이메일 토큰은 일회성 사용
@GetMapping("/verify")
public void verifyEmail(@RequestParam("token") String token,
                       HttpServletRequest request,
                       HttpServletResponse response) {
    try {
        Claims claims = jwtTokenProvider.parseClaims(token);
        
        // 토큰 타입 검증
        if (!"EMAIL_VERIFY".equals(String.valueOf(claims.get("tokenType")))) {
            response.sendRedirect(frontendBaseUrl + "/verify-failed");
            return;
        }

        Long userId = Long.valueOf(claims.getSubject());
        String email = claims.get("email", String.class);

        // 이메일 인증 처리 (토큰 무효화)
        boolean wasActuallyVerified = userService.markEmailVerified(userId, email);
        
        // 성공 페이지로 리다이렉트
        String frontendBaseUrl = getFrontendBaseUrl(request);
        response.sendRedirect(frontendBaseUrl + "/verified?status=success");
        
    } catch (Exception e) {
        response.sendRedirect(frontendBaseUrl + "/verify-failed");
    }
}
```

### 5. 네트워크 보안

```yaml
# application.yml - 동적 CORS 설정
app:
  allowed-hosts: ${ALLOWED_HOSTS:localhost}
  frontend:
    port: ${FRONTEND_PORT:5005}

# CorsConfigurationSource에서 동적 origins 생성
cors:
  allowed-origins: # 환경변수 기반으로 동적 생성
    - http://localhost:5005
    - http://IPv4주소:5005
```

### 6. 동시성 제어 (토큰 갱신)

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
            path.startsWith("/api/v1/auth/signup") ||
            path.startsWith("/api/v1/auth/login");
}
```

### 3. 동시성 제어로 중복 요청 방지

```javascript
// api.js - 토큰 갱신 동시성 제어
const RETRY_KEY = 'x-retried';
let isRefreshing = false;
let waiters = [];

async function refreshCookieToken() {
    // 이미 리프레시 중이면 대기열 추가
    if (isRefreshing) {
        return new Promise((resolve) => waiters.push(resolve));
    }

    isRefreshing = true;
    // 첫 번째 요청만 실제 갱신 수행, 나머지는 결과 공유
}
```

### 4. 네트워크 바인딩 최적화

```yaml
# application.yml - 모든 네트워크 인터페이스 바인딩
server:
  port: ${SERVER_PORT:9005}
  address: 0.0.0.0  # localhost와 IPv4 주소 모두 지원
```

---

## 향후 개선사항

### 1. 단기 계획 (3개월)

#### 이메일 인증 보안 강화 (Docker 기반 이메일 토큰)

**현재 문제점**: URL에 토큰 노출로 인한 보안 취약점
```
http://domain.com/api/v1/auth/verify?token=eyJhbGci...
- 서버 로그에 토큰 기록
- Referer 헤더를 통한 토큰 노출
- 이메일 전달 시 중간 서버에서 토큰 확인 가능
```

**개선 방안**: Docker 컨테이너 기반 이메일 임베딩
```java
@Service
public class SecureEmailService {

    private final DockerEmailRenderer dockerRenderer;

    public void sendSecureVerificationMail(String toEmail, String token) {
        // Docker 컨테이너에서 HTML + 토큰 임베딩 처리
        String secureEmailHtml = dockerRenderer.renderEmailWithEmbeddedToken(
            "email-verification-template",
            Map.of(
                "token", token,
                "userEmail", toEmail,
                "expirationMinutes", 30
            )
        );

        // URL에 토큰 없이 이메일 발송
        MimeMessage message = createSecureMessage(toEmail, secureEmailHtml);
        mailSender.send(message);
    }
}

// Docker 컨테이너 기반 이메일 렌더링
@Component
public class DockerEmailRenderer {
    
    public String renderEmailWithEmbeddedToken(String templateName, Map<String, Object> variables) {
        // Docker 컨테이너에서 보안 토큰 처리
        return dockerClient.exec("email-renderer:latest", 
            "render-secure-email", templateName, variables);
    }
}
```

#### 온보딩 토큰 세분화

```java
// 온보딩 단계별 토큰 권한 분리
public enum OnboardingStep {
    TERMS_CONSENT("ROLE_TERMS"),           // 약관 동의만 가능
    PROFILE_SETUP("ROLE_PROFILE"),         // 프로필 설정만 가능
    PREFERENCES("ROLE_PREFERENCES"),       // 환경설정만 가능
    COMPLETED("ROLE_USER");                // 전체 권한

    private final String role;
}

@PostMapping("/onboarding/terms")
@PreAuthorize("hasRole('TERMS')")
public ResponseEntity<?> acceptTerms(@RequestBody TermsRequest request) {
    // 약관 동의 처리 후 다음 단계 토큰 발급
    String profileToken = jwtTokenProvider.createOnboardingToken(
        userId, email, OnboardingStep.PROFILE_SETUP);
    
    response.addCookie(CookieUtil.build("ONBOARDING_TOKEN", profileToken, 1800));
    return ResponseEntity.ok("다음 단계: 프로필 설정");
}
```

#### 설정 파일 환경별 분리

```
# 현재 구조
src/main/resources/
├── application.yml  # 모든 환경 설정 혼재

# 개선 예정 구조  
src/main/resources/
├── application.yml           # 공통 기본 설정
├── application-dev.yml       # 개발환경 (동적 URL, SQL 로깅)
├── application-staging.yml   # 스테이징환경
├── application-prod.yml      # 운영환경 (고정 도메인, 보안 강화)
└── application-test.yml      # 테스트환경
```

### 2. 중기 계획 (6개월)

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

#### 다중 디바이스 로그인 관리

```java
@Entity
public class UserSession {
    private Long id;
    private Long userId;
    private String sessionId;      // 세션 고유 식별자
    private String deviceInfo;     // 디바이스 정보
    private String ipAddress;      // 로그인 IP
    private LocalDateTime loginTime;
    private LocalDateTime lastActivity;
    private Boolean active;
}

@PostMapping("/auth/sessions")
public ResponseEntity<?> getUserSessions() {
    List<UserSession> sessions = sessionService.getActiveSessions(getCurrentUserId());
    return ResponseEntity.ok(sessions);
}

@DeleteMapping("/auth/sessions/{sessionId}")
public ResponseEntity<?> revokeSession(@PathVariable String sessionId) {
    sessionService.revokeSession(getCurrentUserId(), sessionId);
    return ResponseEntity.ok("세션이 종료되었습니다");
}
```

### 3. 장기 계획 (1년)

#### 로컬 로그인과 소셜 로그인 연동 통합

**목표**: 기존 로컬 계정 사용자가 소셜 계정을 연결할 수 있도록 지원

```java
@Entity
public class UserAccountLink {
    private Long id;
    private Long userId;           // 기존 사용자 ID
    private String provider;       // "google", "kakao", "naver"
    private String providerId;     // 소셜 계정 고유 ID
    private LocalDateTime linkedAt;
    private Boolean primary;       // 주 계정 여부
}

@PostMapping("/auth/link/{provider}")
public ResponseEntity<?> linkSocialAccount(@PathVariable String provider) {
    Long currentUserId = getCurrentUserId();
    
    // 현재 로컬 계정에 소셜 계정 연결
    String linkUrl = oauthService.generateLinkUrl(provider, currentUserId);
    
    return ResponseEntity.ok(Map.of(
        "linkUrl", linkUrl,
        "message", provider + " 계정을 연결하려면 링크를 클릭하세요"
    ));
}

@PostMapping("/auth/unlink/{provider}")
public ResponseEntity<?> unlinkSocialAccount(@PathVariable String provider) {
    Long currentUserId = getCurrentUserId();
    
    // 최소 1개의 로그인 방법은 유지해야 함
    if (userService.getLoginMethodCount(currentUserId) <= 1) {
        return ResponseEntity.badRequest().body(
            "최소 1개의 로그인 방법은 유지해야 합니다"
        );
    }
    
    accountLinkService.unlinkProvider(currentUserId, provider);
    return ResponseEntity.ok(provider + " 계정 연결이 해제되었습니다");
}
```

**사용자 시나리오**:
1. 이메일로 가입한 사용자가 나중에 구글 계정 연결
2. 다음부터는 이메일 또는 구글 둘 다로 로그인 가능
3. 계정 설정에서 연결된 소셜 계정들 관리 가능

#### 리스크 기반 인증

**목표**: 의심스러운 로그인 시도를 자동 감지하여 추가 보안 조치 적용

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

        // 로그인 실패 이력 체크
        if (hasRecentFailedAttempts(attempt.getIpAddress())) {
            score.add(RiskFactor.FAILED_ATTEMPTS, 25);
        }

        return score.calculateLevel();
    }

    private boolean isFromSuspiciousLocation(String ipAddress) {
        // IP 지역 정보 조회하여 평소와 다른 국가/지역인지 확인
        String country = geoLocationService.getCountry(ipAddress);
        return !userLocationService.isKnownCountry(getCurrentUserId(), country);
    }

    private boolean isUnusualLoginTime(Long userId, LocalDateTime loginTime) {
        // 평소 로그인 시간 패턴과 비교
        List<LocalDateTime> recentLogins = loginHistoryService.getRecentLogins(userId, 30);
        return timePatternAnalyzer.isOutsideNormalPattern(recentLogins, loginTime);
    }
}

// 위험도에 따른 추가 인증 요구
@PostMapping("/auth/verify-risk")  
public ResponseEntity<?> handleHighRiskLogin(@RequestBody LoginRequest request) {
    RiskLevel risk = riskService.assessLoginRisk(buildLoginAttempt(request));
    
    if (risk == RiskLevel.HIGH) {
        // 이메일로 인증 코드 발송
        String verificationCode = generateVerificationCode();
        emailService.sendRiskVerificationCode(request.getEmail(), verificationCode);
        
        return ResponseEntity.ok(Map.of(
            "requiresVerification", true,
            "message", "보안을 위해 이메일 인증이 필요합니다"
        ));
    }
    
    // 정상 로그인 처리
    return processNormalLogin(request);
}
```

### 4. 모니터링 및 운영 개선

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
    public void handleOnboardingComplete(OnboardingCompleteEvent event) {
        // 온보딩 토큰 -> ACCESS 토큰 교체 모니터링
        meterRegistry.counter("auth.onboarding.completed", 
            "provider", event.getProvider()).increment();
        
        log.info("온보딩 완료 - 사용자: {}, 소요시간: {}초", 
            event.getUserId(), event.getDurationSeconds());
    }
}
```

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

### 5. 성능 및 확장성 개선

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

---

## 결론

TropiCal 프로젝트의 인증/인가 시스템은 **보안성**, **사용자 경험**, **확장성**을 균형있게 고려한 설계를 채택했습니다.

### 핵심 성과

1. **HttpOnly 쿠키 기반 JWT 인증**으로 XSS 공격 방지
2. **온보딩 토큰 시스템**으로 소셜 로그인 사용자의 단계별 권한 관리
3. **동적 URL 시스템**으로 다중 개발 환경 지원 (localhost, IPv4 주소)
4. **Spring Security Filter 방식**으로 서버 사이드 보안 강화
5. **자동 토큰 갱신**으로 끊김 없는 사용자 경험 제공

### 보안 강화 포인트

- **토큰 생명주기 관리**: 온보딩 완료 시 ONBOARDING_TOKEN 즉시 제거
- **이메일 인증 보안**: URL 토큰 방식에서 Docker 기반 임베딩으로 발전 예정
- **네트워크 보안**: 환경변수 기반 동적 CORS 설정
- **동시성 제어**: 중복 토큰 갱신 요청 방지

### 확장성

현재 구조는 향후 **SSO 통합**, **다중 디바이스 관리**, **리스크 기반 인증** 등으로 확장 가능한 기반을 제공합니다. 특히 토큰 타입별 권한 분리와 환경변수 기반 설정 관리는 복잡한 엔터프라이즈 요구사항에도 대응할 수 있는 유연성을 보장합니다.

이 문서는 팀 내 지식 공유와 신규 개발자 온보딩을 위한 참고 자료로서, 지속적으로 업데이트되며 시스템 발전과 함께 진화할 예정입니다.