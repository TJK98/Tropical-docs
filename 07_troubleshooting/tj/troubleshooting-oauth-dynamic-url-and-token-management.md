# 트러블슈팅: OAuth 하드코딩 URL과 토큰 관리 문제 해결

본 문서는 **Tropical 백엔드/프론트엔드** 개발 과정에서 발생한 **OAuth 소셜 로그인의 하드코딩된 URL과 토큰 관리** 문제의 원인과 해결 과정을 정리한 기술 문서입니다.

localhost와 IPv4 주소 접속 환경에서의 OAuth 콜백 실패, 온보딩 토큰 미발급, 이메일 인증 토큰 위치 혼란 등 다양한 인증 관련 이슈를 환경변수 기반 동적 URL 시스템으로 체계적으로 해결하는 것을 목표로 합니다.

**작성자:** [왕택준](https://github.com/TJK98)

**작성일:** 2025년 9월 24일

**문서 버전:** v1.0

---

## 1. 문제 현상

### 1-1. 온보딩 토큰 관리 문제

**증상 A**: 온보딩 미완료 사용자에게 잘못된 토큰 발급
```java
// 문제 코드: 온보딩 토큰 대신 ACCESS_TOKEN으로 발급
String tempToken = jwtTokenProvider.createAccessToken(user.getId(), user.getEmail());
response.addCookie(CookieUtil.build("ACCESS_TOKEN", tempToken, 1800));
```

**증상 B**: 온보딩 토큰이 아예 발급되지 않는 상황
- OAuth 성공 핸들러에서 ACCESS_TOKEN과 REFRESH_TOKEN만 발급
- 온보딩 미완료 상태임에도 불구하고 정식 토큰 발급
- 온보딩 API 접근 시 권한 부족으로 403 에러 발생

**증상 C**: 온보딩 완료 후 ONBOARDING_TOKEN 미제거
- 온보딩 완료 시 ACCESS_TOKEN, REFRESH_TOKEN 발급하지만
- 기존 ONBOARDING_TOKEN이 브라우저에 계속 남아있음
- 온보딩 완료 후에도 `/api/v1/auth/onboarding` API 접근 가능한 보안 위험

**실제 필요**: ONBOARDING_TOKEN 발급/인식 + 완료 시 제거

### 1-2. 이메일 인증 토큰 위치 혼란

**증상**: EMAIL_VERIFY_TOKEN이 개발자 도구에서 보이지 않음
**혼란 지점**: 이메일 인증 대기 화면 vs 이메일 내 링크

### 1-3. IPv4 주소 접속 시 OAuth 실패

**localhost 접속**: 정상 동작
```
OAuth 요청: http://localhost:9005/oauth2/authorization/kakao
콜백 URL: http://localhost:9005/login/oauth2/code/kakao
```

**IPv4 주소 접속**: 실패 (해결 전)
```
OAuth 요청: http://IPv4주소:9005/oauth2/authorization/kakao (하드코딩으로 인한 localhost 요청)
콜백 URL: 도메인 불일치로 실패
```

**추가 문제 발견**: 백엔드 바인딩 문제
- 백엔드가 `127.0.0.1`에만 바인딩되어 외부 IPv4 접근 불가
- 프론트엔드는 IPv4 주소로 접근하지만 백엔드는 localhost에만 응답

### 1-4. 이메일 인증 링크 하드코딩 문제

**증상**: IPv4 주소로 회원가입 시에도 localhost 기반 인증 링크 발송
```java
// 문제 코드
String verifyUrl = backendBaseUrl + "/api/v1/auth/verify?token=" + emailVerifyToken;
// backendBaseUrl = http://localhost:9005 (고정값)
```

**문제 시나리오**:
1. 개발자가 `http://IPv4주소:5005`로 회원가입
2. 백엔드에서 `http://localhost:9005/api/v1/auth/verify?token=...` 링크 생성
3. 이메일로 localhost 링크 발송
4. 개발자가 IPv4 환경에서 localhost 링크 클릭 시 접근 불가

**로그 예시**:
```
2025-09-23T23:41:57.478  INFO : 온보딩용 임시 토큰 발급 - 사용자 ID: 1
2025-09-23T23:41:58.293  WARN : 인증 실패 - Full authentication is required
2025-09-23T23:41:58.295 DEBUG : 에러 응답 작성 - Status: 401, Code: UNAUTHORIZED
```

### 1-5. 환경 정보

- **백엔드**: Spring Boot 3.x, OAuth2, JWT
- **프론트엔드**: React + Vite, Axios
- **인증 방식**: HttpOnly 쿠키 기반 JWT
- **소셜 로그인**: 카카오, 구글, 네이버
- **개발 환경**: 동일 네트워크 내 다중 IPv4 접근 필요

---

## 2. 원인 분석

### 2-1. 1단계: 온보딩 토큰 문제 분석

**발견된 문제**:
```java
// OAuth2AuthenticationSuccessHandler.java
if (!user.isOnboardingCompleted()) {
    // 문제 1: 잘못된 토큰 타입 생성
    String tempToken = jwtTokenProvider.createAccessToken(user.getId(), user.getEmail()); 
    // 문제 2: 잘못된 쿠키명 설정
    response.addCookie(CookieUtil.build("ACCESS_TOKEN", tempToken, 1800)); 
}
```

**JwtAuthenticationFilter에서 ONBOARDING_TOKEN 쿠키를 읽지 못함**:
```java
// 기존: ACCESS_TOKEN만 확인
private static final String ACCESS_TOKEN_COOKIE = "ACCESS_TOKEN";

// 필요: ONBOARDING_TOKEN도 확인
private static final String ONBOARDING_TOKEN_COOKIE = "ONBOARDING_TOKEN";
```

### 2-2. 2단계: 이메일 인증 토큰 위치 분석

**설계상 정상 동작**:
```java
// AuthController.java - 이메일로만 전송
String emailVerifyToken = jwtTokenProvider.createEmailVerifyToken(user.getId(), user.getEmail());
String verifyUrl = backendBaseUrl + "/api/v1/auth/verify?token=" + emailVerifyToken;
emailService.sendVerificationMail(user.getEmail(), verifyUrl);
```

**혼란의 원인**: EMAIL_VERIFY_TOKEN이 쿠키가 아닌 URL 파라미터로 전송되는 보안 설계

### 2-3. 3단계: 하드코딩 URL 문제 분석

**백엔드 하드코딩**:
```yaml
# application.yml (기존)
app:
  frontend:
    base-url: http://localhost:5005  # 고정값 문제
  backend:
    base-url: http://localhost:9005  # 고정값 문제
server:
  port: 9005  # 포트만 설정, 바인딩 주소 누락
```

**프론트엔드 하드코딩**:
```javascript
// .env (기존)
VITE_API_BASE_URL=http://localhost:9005
VITE_KAKAO_LOGIN_URL=http://localhost:9005/oauth2/authorization/kakao
```

**네트워크 바인딩 문제**:
- Spring Boot 기본값: `server.address=127.0.0.1` (localhost만 접근 가능)
- 필요한 설정: `server.address=0.0.0.0` (모든 네트워크 인터페이스 바인딩)

**OAuth 프로세스 실패 시나리오**:
1. 사용자가 `http://IPv4주소:5005`로 접속
2. 카카오 로그인 버튼 클릭 시 `http://localhost:9005/oauth2/authorization/kakao`로 요청 (하드코딩)
3. **Connection refused**: 백엔드가 IPv4주소:9005에서 응답하지 않음
4. 접속 성공해도 카카오에서 `http://localhost:9005/login/oauth2/code/kakao`로 콜백 시도
5. 사용자 브라우저는 IPv4주소이므로 콜백 실패
6. 성공해도 `http://localhost:5005/onboarding`로 리다이렉트 (도메인 불일치)

### 2-4. 4단계: 쿠키 도메인 문제 분석

**CookieUtil 분석 결과**:
```java
public static Cookie build(String name, String value, int maxAgeSeconds) {
    Cookie cookie = new Cookie(name, value);
    cookie.setHttpOnly(true);
    cookie.setPath("/");
    cookie.setMaxAge(maxAgeSeconds);
    // cookie.setDomain() 미설정 → 현재 호스트에만 쿠키 설정
    return cookie;
}
```

**정상 동작 확인**: localhost에서 설정된 쿠키는 IPv4 주소에서 전송되지 않는 것이 브라우저 보안 정책상 정상

---

## 3. 디버깅 과정

### 3-1. 문제 우선순위 분석

**TJ의 문제 정리**:
1. 온보딩 토큰이 제대로 발급이 안 됨
2. 이메일 인증 대기 화면에서 EMAIL_VERIFY_TOKEN이 안 나옴
3. IPv4 주소 접속 시 소셜 로그인 실패
4. 이메일 인증 링크가 localhost로 고정됨

**디버깅 순서 결정**:
1. **네트워크 바인딩**: 백엔드 접근 가능성 확보 (인프라)
2. **온보딩 토큰**: 토큰 타입 및 쿠키명 수정 (인증)
3. **이메일 토큰**: 설계 이해 문제 (설명 필요)
4. **IPv4 주소 접속**: 하드코딩 문제 (구조적 수정)

### 3-2. 온보딩 토큰 디버깅

**JWT 디코딩으로 토큰 내용 확인**:
```json
{
  "tokenType": "ACCESS",     // 문제: ACCESS가 아닌 ONBOARDING이어야 함
  "sub": "1",
  "email": "user@example.com"
}
```

**JwtAuthenticationFilter 로그 분석**:
```
DEBUG : JWT 토큰 추출 완료 (쿠키: ACCESS_TOKEN)
WARN  : 인증에 사용할 수 없는 토큰 타입: ACCESS
```

### 3-3. 이메일 토큰 디버깅

**실제 이메일 링크 분석**:
```
http://localhost:9005/api/v1/auth/verify?token=eyJhbGciOiJIUzI1NiJ9...
```

**JWT 디코딩 결과**:
```json
{
  "sub": "1",
  "email": "wtj1998@naver.com",
  "tokenType": "EMAIL_VERIFY",  // 올바른 타입
  "purpose": "email_verification",
  "iat": 1758683909,
  "exp": 1758685709
}
```

**결론**: 이메일 토큰은 정상 작동, 쿠키가 아닌 URL 파라미터가 설계 의도

### 3-4. IPv4 주소 접속 디버깅

**네트워크 탭 분석**:
```
요청 URL: http://localhost:9005/oauth2/authorization/kakao (잘못됨)
예상 URL: http://IPv4주소:9005/oauth2/authorization/kakao (올바름)
```

**카카오 개발자 콘솔 확인**:
- 등록된 Redirect URI: `http://localhost:9005/login/oauth2/code/kakao`만 존재
- 필요한 추가 URI: `http://IPv4주소:9005/login/oauth2/code/kakao`

---

## 4. 해결 과정

### 4-1. 백엔드 네트워크 바인딩 개선

**application.yml 수정**:
```yaml
server:
  port: ${SERVER_PORT:9005}  # 서버 포트 번호 (환경변수 또는 기본값 9005)
  address: 0.0.0.0          # 모든 네트워크 인터페이스에 바인딩 (핵심 개선사항)

app:
  frontend:
    port: ${FRONTEND_PORT:5005}
    use-dynamic-host: ${USE_DYNAMIC_HOST:true}
    domain: ${FRONTEND_DOMAIN:}
  backend:
    base-url: ${APP_BASE_URL:http://localhost:9005}
    domain: ${BACKEND_DOMAIN:}
  allowed-hosts: ${ALLOWED_HOSTS:localhost}
```

**네트워크 바인딩 변경 효과**:
- **변경 전**: `127.0.0.1:9005`에만 바인딩 → 외부 IPv4 접근 불가
- **변경 후**: `0.0.0.0:9005`로 바인딩 → 모든 IPv4에서 접근 가능 (`localhost:9005`, `IPv4주소:9005` 등)

### 4-2. 온보딩 토큰 수정

**OAuth2AuthenticationSuccessHandler.java 수정**:
```java
if (!user.isOnboardingCompleted()) {
    // 수정 전
    String tempToken = jwtTokenProvider.createAccessToken(user.getId(), user.getEmail());
    response.addCookie(CookieUtil.build("ACCESS_TOKEN", tempToken, 1800));
    
    // 수정 후
    String onboardingToken = jwtTokenProvider.createOnboardingToken(user.getId(), user.getEmail());
    response.addCookie(CookieUtil.build("ONBOARDING_TOKEN", onboardingToken, 1800));
}
```

**JwtAuthenticationFilter.java 수정**:
```java
// 상수 추가
private static final String ONBOARDING_TOKEN_COOKIE = "ONBOARDING_TOKEN";

// 토큰 추출 로직 수정
if (ACCESS_TOKEN_COOKIE.equals(cookie.getName()) || 
    ONBOARDING_TOKEN_COOKIE.equals(cookie.getName())) {
    // 토큰 처리 로직
}
```

**온보딩 완료 시 토큰 정리**:
```java
// AuthController.java - completeSocialOnboarding 메서드
// 기존 ONBOARDING 토큰 제거 (보안상 중요)
response.addCookie(CookieUtil.expire("ONBOARDING_TOKEN"));

// 정식 ACCESS/REFRESH 토큰으로 교체
String accessToken = jwtTokenProvider.createAccessToken(user.getId(), user.getEmail());
String refreshToken = jwtTokenProvider.createRefreshToken(user.getId());
response.addCookie(CookieUtil.build("ACCESS_TOKEN", accessToken, accessMaxAge));
response.addCookie(CookieUtil.build("REFRESH_TOKEN", refreshToken, refreshMaxAge));
```

### 4-3. 동적 URL 생성 시스템 구축

**AuthController에 동적 URL 생성 메서드 추가**:

```java
/**
 * 환경변수 기반 프론트엔드 URL 동적 생성
 *
 * 설정 우선순위:
 * 1순위: 운영환경 고정 도메인 (FRONTEND_DOMAIN)
 * 2순위: 동적 호스트 (현재 요청 호스트 + 프론트엔드 포트)
 * 3순위: localhost 폴백
 */
private String getFrontendBaseUrl(HttpServletRequest request) {
    // 1순위: 운영환경에서 고정 도메인이 설정된 경우 우선 적용
    if (!frontendDomain.isEmpty()) {
        log.debug("고정 프론트엔드 도메인 사용: {}", frontendDomain);
        return frontendDomain;
    }

    // 2순위: 개발환경에서 동적 호스트 사용
    if (useDynamicHost) {
        String scheme = request.getScheme(); // http or https
        String serverName = request.getServerName(); // localhost or IPv4
        String dynamicUrl = String.format("%s://%s:%s", scheme, serverName, frontendPort);
        log.debug("동적 프론트엔드 URL 생성: {} (요청 호스트: {})", dynamicUrl, serverName);
        return dynamicUrl;
    }

    // 3순위: 폴백 - localhost 고정 사용
    String fallbackUrl = String.format("http://localhost:%s", frontendPort);
    log.debug("폴백 프론트엔드 URL 사용: {}", fallbackUrl);
    return fallbackUrl;
}

/**
 * 환경변수 기반 백엔드 URL 동적 생성 (이메일 인증 링크용)
 *
 * 설정 우선순위:
 * 1순위: 운영환경 고정 도메인 (BACKEND_DOMAIN)
 * 2순위: 동적 호스트 (현재 요청 호스트 + 백엔드 포트)
 * 3순위: localhost 폴백
 */
private String getBackendBaseUrl(HttpServletRequest request) {
    // 1순위: 운영환경에서 고정 도메인이 설정된 경우 우선 적용
    if (backendDomain != null && !backendDomain.trim().isEmpty()) {
        log.debug("고정 백엔드 도메인 사용: {}", backendDomain);
        return backendDomain;
    }

    // 2순위: 개발환경에서 동적 호스트 사용
    if (useDynamicHost) {
        String scheme = request.getScheme(); // http or https
        String serverName = request.getServerName(); // localhost or IPv4
        int serverPort = request.getServerPort(); // 실제 요청받은 포트
        String dynamicUrl = String.format("%s://%s:%d", scheme, serverName, serverPort);
        log.debug("동적 백엔드 URL 생성: {} (요청 호스트: {})", dynamicUrl, serverName);
        return dynamicUrl;
    }

    // 3순위: 폴백 - localhost 고정 사용 (기존 설정값 활용)
    log.debug("폴백 백엔드 URL 사용: {}", backendBaseUrl);
    return backendBaseUrl;
}
```

### 4-4. 이메일 인증 링크 동적 URL 적용

**signup과 resendVerificationEmail 메서드 수정**:

```java
@PostMapping("/signup")
@Transactional
public ResponseEntity<?> signup(@Valid @RequestBody SignupRequest signupRequest,
                               HttpServletRequest request) { // 요청 객체 추가
    // ... 기존 로직 ...

    // 4) 이메일 인증 토큰 발급 및 발송 - 동적 URL 적용
    String emailVerifyToken = jwtTokenProvider.createEmailVerifyToken(user.getId(), user.getEmail());
    String backendUrl = getBackendBaseUrl(request); // 동적 URL 생성
    String verifyUrl = backendUrl + "/api/v1/auth/verify?token=" + emailVerifyToken;
    emailService.sendVerificationMail(user.getEmail(), verifyUrl);

    log.info("로컬 계정 회원가입 완료 - 사용자 ID: {}, 동적 인증 URL: {}", user.getId(), backendUrl);
    
    // ... 기존 응답 로직 ...
}

@PostMapping("/verify/resend")
public ResponseEntity<?> resendVerificationEmail(HttpServletRequest request) { // 요청 객체 추가
    // ... 기존 검증 로직 ...

    // 인증 메일 재발송 - 동적 URL 적용
    String emailVerifyToken = jwtTokenProvider.createEmailVerifyToken(user.getId(), user.getEmail());
    String backendUrl = getBackendBaseUrl(request); // 동적 URL 생성
    String verifyUrl = backendUrl + "/api/v1/auth/verify?token=" + emailVerifyToken;
    emailService.sendVerificationMail(user.getEmail(), verifyUrl);

    log.info("이메일 인증 메일 재발송 완료 - 사용자 ID: {}, 동적 인증 URL: {}", userId, backendUrl);
    
    // ... 기존 응답 로직 ...
}
```

### 4-5. 프론트엔드 동적 프록시 라우터 구축

**vite.config.js 고도화**:
```javascript
/**
 * Vite 설정 파일 - 동적 프록시 라우터
 *
 * 목표: 네트워크 내에서 백엔드가 어떤 PC(IPv4)에서 떠도 자동으로 프록시되도록 함
 * 핵심 전략: 요청 시점에 Host 헤더에서 IPv4를 동적으로 추출해 사용
 */
export default ({ mode }) => {
    const env = loadEnv(mode, process.cwd(), "");
    const FRONTEND_PORT = Number(env.VITE_FRONTEND_PORT || "5005");
    const BACKEND_PORT = env.VITE_BACKEND_PORT || "9005";
    const USE_DYNAMIC_HOST = (env.VITE_USE_DYNAMIC_HOST || "false") === "true";
    const BACKEND_DOMAIN = (env.VITE_BACKEND_DOMAIN || "").trim();

    /**
     * DEV 프록시 타깃 동적 계산
     * 
     * 우선순위:
     *  1) VITE_BACKEND_DOMAIN 지정 시: 고정 도메인 사용
     *  2) VITE_USE_DYNAMIC_HOST === true: 현재 요청의 Host 헤더에서 hostname 추출
     *  3) 기본: localhost 사용
     */
    const dynamicRouter = (req) => {
        // (1) 고정 도메인 우선
        if (BACKEND_DOMAIN) {
            const protocol = BACKEND_DOMAIN.startsWith('http') ? '' : 'http://';
            return `${protocol}${BACKEND_DOMAIN}`;
        }

        // (2) 동적 호스트 사용 - 핵심 개선사항
        if (USE_DYNAMIC_HOST) {
            // Host 헤더에서 hostname만 안전하게 추출
            // 예: "IPv4주소:5005" → "IPv4주소"
            const hostHeader = req.headers.host;
            const hostOnly = (hostHeader || 'localhost').split(':')[0] || 'localhost';
            return `http://${hostOnly}:${BACKEND_PORT}`;
        }

        // (3) 기본 폴백
        return `http://localhost:${BACKEND_PORT}`;
    };

    return defineConfig({
        plugins: [react(), svgr()],
        server: {
            host: true,        // 0.0.0.0 바인딩으로 외부 IPv4 접근 허용
            port: FRONTEND_PORT,
            proxy: {
                "/api/v1": {
                    target: `http://localhost:${BACKEND_PORT}`, // 기본값
                    changeOrigin: true,
                    secure: false,
                    router: dynamicRouter,  // 핵심: 요청별 동적 타깃 결정
                },
                "/oauth2": {
                    target: `http://localhost:${BACKEND_PORT}`,
                    changeOrigin: true,
                    secure: false,
                    router: dynamicRouter,
                },
            },
        },
    });
};
```

### 4-6. 환경변수 기반 동적 URL 시스템 구축

**백엔드 .env 파일 수정**:
```bash
# 동적 호스트 지원 환경변수 추가
SERVER_PORT=9005
FRONTEND_PORT=5005
ALLOWED_HOSTS=localhost,IPv4주소1,IPv4주소2
USE_DYNAMIC_HOST=true
FRONTEND_DOMAIN=
BACKEND_DOMAIN=  # 신규 추가: 백엔드 고정 도메인 설정
```

**프론트엔드 .env 파일 수정**:
```bash
# Vite 환경변수 (VITE_ 접두사 필수)
VITE_FRONTEND_PORT=5005
VITE_BACKEND_PORT=9005
VITE_USE_DYNAMIC_HOST=true
VITE_BACKEND_DOMAIN=
```

### 4-7. CORS 설정 동적화

**CORS 설정 개선**:
```java
@Value("${app.allowed-hosts:localhost}")
private String allowedHostsStr;

@Value("${app.frontend.port:5005}")
private String frontendPort;

@Bean
public CorsConfigurationSource corsConfigurationSource() {
    String[] allowedHosts = allowedHostsStr.split(",");
    List<String> allowedOrigins = Arrays.stream(allowedHosts)
            .map(host -> String.format("http://%s:%s", host.trim(), frontendPort))
            .collect(Collectors.toList());
    
    CorsConfiguration conf = new CorsConfiguration();
    conf.setAllowedOrigins(allowedOrigins); // 동적 origins 적용
    conf.setAllowCredentials(true);
    conf.setAllowedHeaders(List.of("*"));
    conf.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", conf);
    return source;
}
```

---

## 5. 테스트 검증

### 5-1. 네트워크 바인딩 테스트

**백엔드 바인딩 확인**:
```bash
# localhost 접근 테스트
curl http://localhost:9005/api/v1/health
# HTTP 200 OK

# IPv4 주소 접근 테스트 (새로운 성공 케이스)
curl http://IPv4주소:9005/api/v1/health  
# HTTP 200 OK - address: 0.0.0.0 설정으로 성공

# 다른 네트워크 PC에서 접근 테스트
curl http://IPv4주소:9005/api/v1/health
# HTTP 200 OK - 네트워크 내 다른 PC에서도 접근 가능
```

### 5-2. 동적 프록시 라우터 테스트

**localhost 접속 테스트**:
```javascript
// 브라우저에서 http://localhost:5005 접속
// Vite 개발서버 로그 확인:
// [vite] proxy /api/v1 -> http://localhost:9005
console.log('프록시 타깃:', dynamicRouter({ headers: { host: 'localhost:5005' } }));
// 출력: http://localhost:9005
```

**IPv4 주소 접속 테스트**:
```javascript
// 브라우저에서 http://IPv4주소:5005 접속
// Vite 개발서버 로그 확인:
// [vite] proxy /api/v1 -> http://IPv4주소:9005 (동적 변경됨)
console.log('프록시 타깃:', dynamicRouter({ headers: { host: 'IPv4주소:5005' } }));
// 출력: http://IPv4주소:9005
```

### 5-3. 온보딩 토큰 발급 및 정리 테스트

**토큰 발급 확인**:
```bash
# 소셜 로그인 후 쿠키 확인
curl -i http://IPv4주소:9005/oauth2/authorization/kakao

# 응답 헤더에서 Set-Cookie 확인 (수정 후)
Set-Cookie: ONBOARDING_TOKEN=eyJhbGci...; HttpOnly; Path=/; Max-Age=1800
```

**토큰 인식 확인**:
```bash
# 온보딩 API 호출
curl -b "ONBOARDING_TOKEN=eyJhbGci..." \
     http://IPv4주소:9005/api/v1/auth/onboarding

# 200 OK 응답 확인
```

**온보딩 완료 후 토큰 정리 확인**:
```bash
# 온보딩 완료 API 호출
curl -b "ONBOARDING_TOKEN=eyJhbGci..." \
     -H "Content-Type: application/json" \
     -d '{"requiredConsents":{"termsOfService":true,"privacyPolicy":true,"calendarPersonalization":true}}' \
     http://IPv4주소:9005/api/v1/auth/onboarding

# 응답에서 토큰 교체 확인
Set-Cookie: ONBOARDING_TOKEN=; Max-Age=0; Expires=Thu, 01 Jan 1970 00:00:00 GMT  # 삭제
Set-Cookie: ACCESS_TOKEN=newAccessToken...; HttpOnly; Path=/; Max-Age=900         # 신규 발급
Set-Cookie: REFRESH_TOKEN=newRefreshToken...; HttpOnly; Path=/; Max-Age=1209600   # 신규 발급
```

### 5-4. 이메일 인증 링크 동적 생성 테스트

**localhost 회원가입 테스트**:
```bash
# localhost:9005에서 회원가입 요청
POST http://localhost:9005/api/v1/auth/signup

# 백엔드 로그 확인
INFO : 로컬 계정 회원가입 완료 - 사용자 ID: 1, 동적 인증 URL: http://localhost:9005
DEBUG: 동적 백엔드 URL 생성: http://localhost:9005 (요청 호스트: localhost)

# 이메일로 발송된 링크
http://localhost:9005/api/v1/auth/verify?token=eyJhbGci...
```

**IPv4 주소 회원가입 테스트 (개선 후)**:
```bash
# IPv4주소:9005에서 회원가입 요청
POST http://IPv4주소:9005/api/v1/auth/signup

# 백엔드 로그 확인
INFO : 로컬 계정 회원가입 완료 - 사용자 ID: 2, 동적 인증 URL: http://IPv4주소:9005
DEBUG: 동적 백엔드 URL 생성: http://IPv4주소:9005 (요청 호스트: IPv4주소)

# 이메일로 발송된 링크 (IPv4 주소 기반)
http://IPv4주소:9005/api/v1/auth/verify?token=eyJhbGci...
```

### 5-5. 통합 OAuth 플로우 테스트

**성공 시나리오 (개선 후)**:
1. `http://IPv4주소:5005` 접속 → Vite 서버 응답
2. 카카오 로그인 버튼 클릭 → `/oauth2/authorization/kakao` 프록시 요청
3. Vite 프록시가 `http://IPv4주소:9005/oauth2/authorization/kakao`로 전달
4. Spring Boot 서버 (`0.0.0.0:9005`)에서 응답
5. 카카오 인증 완료 → `http://IPv4주소:9005/login/oauth2/code/kakao` 콜백
6. 백엔드에서 ONBOARDING_TOKEN 발급 및 동적 프론트엔드 URL 계산
7. `http://IPv4주소:5005/onboarding`로 리다이렉트 성공

### 5-6. 이메일 인증 플로우 종합 테스트

**전체 플로우 검증**:
1. **회원가입**: `IPv4주소:5005`에서 회원가입
2. **동적 링크 생성**: `http://IPv4주소:9005/api/v1/auth/verify?token=...`
3. **이메일 수신**: 올바른 IPv4 기반 링크 확인
4. **링크 클릭**: IPv4 환경에서 정상 인증
5. **리다이렉트**: `http://IPv4주소:5005/verified?status=success`

**재발송 테스트**:
```bash
# 이메일 재발송 요청 (IPv4 환경에서)
POST http://IPv4주소:9005/api/v1/auth/verify/resend

# 백엔드 로그 확인
INFO : 이메일 인증 메일 재발송 완료 - 사용자 ID: 1, 동적 인증 URL: http://IPv4주소:9005

# 재발송된 이메일에도 올바른 IPv4 기반 링크 포함
http://IPv4주소:9005/api/v1/auth/verify?token=newtoken...
```

---

## 6. 성능 및 보안 영향 분석

### 6-1. 성능 영향

**변경 전**:
- 하드코딩으로 인한 URL 불일치
- IPv4 접속 시 Connection refused 에러
- 불필요한 에러 발생과 재시도
- 토큰 타입 오인식으로 인한 추가 요청
- 이메일 인증 링크 접근 불가로 인한 재발송 요청

**변경 후**:
- 환경변수 읽기: 애플리케이션 시작 시 1회
- 동적 URL 생성: 요청당 String.format() 및 Host 헤더 파싱
- 동적 프록시 라우터: 요청당 Host 헤더 split 연산
- 전체적으로 무시할 수 있는 오버헤드 (마이크로초 단위)

**네트워크 바인딩 개선**:
- `0.0.0.0` 바인딩으로 추가적인 네트워크 인터페이스 리스닝
- 메모리 사용량 무시할 정도 증가
- CPU 오버헤드 없음

### 6-2. 보안 강화

**CORS 정책 개선**:
```java
// 기존: 하드코딩된 origin
conf.setAllowedOrigins(List.of("http://localhost:5005"));

// 개선: 환경변수 기반 동적 origins
conf.setAllowedOrigins(allowedOrigins); // localhost,IPv4주소 모두 지원
```

**토큰 타입 명확화**:
- ONBOARDING_TOKEN: 온보딩 API만 접근 가능
- ACCESS_TOKEN: 모든 API 접근 가능
- 권한별 명확한 토큰 구분으로 보안 강화
- 온보딩 완료 시 ONBOARDING_TOKEN 자동 제거

**네트워크 보안 고려사항**:
- `0.0.0.0` 바인딩으로 외부 접근 허용
- 개발환경에서는 적절하지만 운영환경에서는 방화벽/로드밸런서와 함께 사용 권장
- ALLOWED_HOSTS 환경변수로 허용 호스트 제한

---

## 7. 운영 가이드라인

### 7-1. 새로운 개발 환경 추가 방법

**새로운 IPv4 주소 추가**:
```bash
# 백엔드 .env 파일 수정
ALLOWED_HOSTS=localhost,기존IPv4,새IPv4주소  # 새 IPv4 추가

# 프론트엔드 .env 파일 (동일한 호스트 목록)
VITE_ALLOWED_HOSTS=localhost,기존IPv4,새IPv4주소
```

**소셜 로그인 콜백 URL 등록**:
```
카카오 개발자 콘솔 → Redirect URI 추가:
http://새IPv4주소:9005/login/oauth2/code/kakao
```

**자동 동작 확인**:
- 환경변수 설정 후 서버 재시작만으로 새 IPv4에서 접근 가능
- 프론트엔드는 코드 변경 없이 자동으로 새 IPv4 인식

### 7-2. 환경별 배포 가이드

**개발환경 (.env.development)**:
```bash
# 동적 호스트 사용으로 팀원별 IPv4 자동 지원
SERVER_PORT=9005
USE_DYNAMIC_HOST=true
FRONTEND_DOMAIN=
BACKEND_DOMAIN=
ALLOWED_HOSTS=localhost,IPv4주소1,IPv4주소2
```

**스테이징 환경 (.env.staging)**:
```bash
# 고정 도메인 사용
USE_DYNAMIC_HOST=false
FRONTEND_DOMAIN=https://staging-tropical.com
BACKEND_DOMAIN=https://staging-api-tropical.com
```

**운영환경 (.env.production)**:
```bash
# 보안 강화를 위한 고정 도메인 + 특정 바인딩
SERVER_PORT=8080
USE_DYNAMIC_HOST=false
FRONTEND_DOMAIN=https://tropical.com
BACKEND_DOMAIN=https://api.tropical.com
# server.address는 운영환경에서 로드밸런서 IPv4로 제한
```

### 7-3. 모니터링 포인트

**애플리케이션 시작 시 로그 확인**:
```
INFO : 서버 바인딩 주소: 0.0.0.0:9005
INFO : CORS 허용 Origins: [http://localhost:5005, http://IPv4주소:5005]
INFO : 동적 호스트 모드: true
INFO : 허용된 호스트 목록: [localhost, IPv4주소]
```

**런타임 로그 확인**:
```
INFO : 온보딩용 토큰 발급 - 사용자 ID: 1
INFO : 동적 프론트엔드 URL 생성: http://IPv4주소:5005
INFO : 동적 백엔드 URL 생성: http://IPv4주소:9005
DEBUG: OAuth 리다이렉트 URL: http://IPv4주소:5005/onboarding
DEBUG: 이메일 인증 URL: http://IPv4주소:9005/api/v1/auth/verify?token=...
```

**에러 패턴 및 해결**:
- `CORS policy blocked`: ALLOWED_HOSTS 설정 확인
- `Connection refused`: 백엔드 `server.address` 설정 확인
- `토큰 타입 오류`: ONBOARDING_TOKEN vs ACCESS_TOKEN 발급 확인
- `OAuth callback failed`: 소셜 로그인 콘솔 Redirect URI 등록 확인
- `이메일 링크 접근 불가`: 백엔드 동적 URL 생성 확인

**Vite 개발서버 프록시 로그**:
```
[vite] proxy /api/v1 -> http://IPv4주소:9005 (동적 변경됨)
[vite] proxy /oauth2 -> http://IPv4주소:9005
```

---

## 8. 관련 이슈 및 예방책

### 8-1. 네트워크 바인딩 안티패턴

**위험한 패턴들**:
```yaml
# 하드코딩된 바인딩 (외부 접근 불가)
server:
  port: 9005
  # address 미설정 시 기본값 127.0.0.1 사용 - IPv4 접근 불가

# 하드코딩된 CORS 설정
cors:
  allowed-origins: http://localhost:5005  # IPv4 접근 시 CORS 에러
```

**안전한 패턴들**:
```yaml
# 환경변수 기반 동적 바인딩
server:
  port: ${SERVER_PORT:9005}
  address: 0.0.0.0  # 모든 네트워크 인터페이스 바인딩

# 환경변수 기반 CORS 설정
app:
  allowed-hosts: ${ALLOWED_HOSTS:localhost}
  frontend:
    port: ${FRONTEND_PORT:5005}
```

### 8-2. Vite 프록시 설정 안티패턴

**위험한 패턴들**:
```javascript
// 정적 프록시 설정 (IPv4 접근 불가)
proxy: {
  "/api/v1": {
    target: "http://localhost:9005",  // 하드코딩
    changeOrigin: true,
  }
}

// 환경변수 무시
export default defineConfig({
  server: {
    host: false,  // localhost에만 바인딩
    port: 5005,   // 하드코딩
  }
});
```

**안전한 패턴들**:
```javascript
// 동적 프록시 라우터
const dynamicRouter = (req) => {
  if (USE_DYNAMIC_HOST) {
    const hostOnly = (req.headers.host || 'localhost').split(':')[0];
    return `http://${hostOnly}:${BACKEND_PORT}`;
  }
  return `http://localhost:${BACKEND_PORT}`;
};

export default defineConfig({
  server: {
    host: true,    // 0.0.0.0 바인딩
    port: FRONTEND_PORT,  // 환경변수 사용
    proxy: {
      "/api/v1": {
        target: `http://localhost:${BACKEND_PORT}`,
        changeOrigin: true,
        router: dynamicRouter,  // 동적 라우팅
      }
    }
  }
});
```

### 8-3. 토큰 관리 체크리스트

**토큰 발급 시 확인사항**:
- 올바른 토큰 타입 사용 (ONBOARDING vs ACCESS)
- 적절한 쿠키명 설정 (토큰 타입과 일치)
- 만료 시간 설정 (ONBOARDING: 30분, ACCESS: 15분)
- HttpOnly 플래그 확인
- 온보딩 완료 시 ONBOARDING_TOKEN 제거

**토큰 검증 시 확인사항**:
- 모든 토큰 타입의 쿠키 읽기 지원
- 토큰 타입별 권한 매핑 정확성
- 만료 토큰 처리 로직
- CORS와 쿠키 도메인 정책 호환성

### 8-4. URL 생성 메서드 구조 가이드라인

**메서드 역할 구분**:
```java
/**
 * 프론트엔드 리다이렉트용 URL 생성
 * - OAuth 성공 후 리다이렉트
 * - 이메일 인증 완료 후 리다이렉트
 * - 포트: 5005 (프론트엔드)
 */
private String getFrontendBaseUrl(HttpServletRequest request) {
    // 1순위: 고정 도메인 → 2순위: 동적 호스트 → 3순위: localhost 폴백
}

/**
 * 이메일 인증 링크용 URL 생성
 * - 회원가입 시 이메일 인증 링크
 * - 이메일 재발송 시 인증 링크
 * - 포트: 9005 (백엔드)
 */
private String getBackendBaseUrl(HttpServletRequest request) {
    // 1순위: 고정 도메인 → 2순위: 동적 호스트 → 3순위: localhost 폴백
}
```

**사용 위치별 구분**:
- **getFrontendBaseUrl**: `verifyEmail` 메서드에서 인증 완료 후 리다이렉트
- **getBackendBaseUrl**: `signup`, `resendVerificationEmail` 메서드에서 이메일 링크 생성

### 8-5. 환경 설정 관리 가이드라인

**환경변수 네이밍 규칙**:
```bash
# 백엔드 환경변수
SERVER_PORT=9005
FRONTEND_PORT=5005
USE_DYNAMIC_HOST=true
ALLOWED_HOSTS=localhost,IPv4주소1,IPv4주소2
BACKEND_DOMAIN=  # 운영환경용 고정 도메인

# 프론트엔드 환경변수 (VITE_ 접두사 필수)
VITE_FRONTEND_PORT=5005
VITE_BACKEND_PORT=9005  
VITE_USE_DYNAMIC_HOST=true
VITE_BACKEND_DOMAIN=
```

**설정 검증 방법**:
- 애플리케이션 시작 시 환경변수 로깅
- 개발/운영 환경별 설정값 문서화
- 설정 변경 시 영향도 체크리스트 활용
- 네트워크 바인딩 테스트 (localhost + IPv4 접근)
- 이메일 인증 링크 URL 동적 생성 확인

### 8-6. 네트워크 보안 고려사항

**개발환경 보안**:
```yaml
# 적절한 바인딩: 팀 네트워크 내에서만 접근
server:
  address: 0.0.0.0  # 모든 인터페이스, 방화벽으로 제한

# CORS 제한: 알려진 호스트만 허용
app:
  allowed-hosts: localhost,IPv4주소1,IPv4주소2
```

**운영환경 보안**:
```yaml
# 제한적 바인딩: 로드밸런서 IPv4만
server:
  address: 10.0.1.100  # 특정 내부 IPv4

# 프록시/게이트웨이 뒤에서 동작
app:
  frontend:
    domain: https://tropical.com
  backend:
    domain: https://api.tropical.com
```

---

## 9. 결론 및 배운 점

### 9-1. 주요 성과

1. **온보딩 토큰 정상화**: ONBOARDING_TOKEN 발급 및 인식으로 권한 체계 명확화
2. **토큰 생명주기 관리**: 온보딩 완료 시 ONBOARDING_TOKEN 제거 및 정식 토큰 발급
3. **이메일 인증 이해**: URL 파라미터 방식의 보안 설계 원리 파악
4. **네트워크 바인딩 개선**: `server.address: 0.0.0.0`으로 IPv4 접근 문제 근본 해결
5. **동적 프록시 시스템**: Vite router 기능으로 요청별 백엔드 IPv4 자동 인식
6. **이메일 인증 링크 동적화**: 현재 요청 호스트 기반 인증 링크 생성으로 IPv4 접속 지원
7. **확장 가능한 아키텍처**: 새로운 개발 환경 추가 시 환경변수만 수정하면 되는 구조

### 9-2. 기술적 학습

**네트워크 바인딩의 중요성**:
- Spring Boot 기본 바인딩 (`127.0.0.1`)의 제한사항 이해
- `0.0.0.0` 바인딩으로 모든 네트워크 인터페이스에서 접근 가능
- 개발환경 vs 운영환경에서의 바인딩 전략 차이

**OAuth 인증 플로우의 복잡성**:
- 프론트엔드 URL → 백엔드 OAuth 엔드포인트 → 소셜 로그인 서버 → 백엔드 콜백 → 프론트엔드 리다이렉트
- 각 단계에서 도메인 일치 및 네트워크 접근 가능성 필요
- 하드코딩된 URL 하나가 전체 플로우를 망가뜨릴 수 있음

**이메일 인증 시스템의 도메인 종속성**:
- 이메일 인증 링크는 백엔드 API 엔드포인트를 직접 참조
- 하드코딩된 백엔드 URL로 인한 IPv4 접속 환경에서의 인증 실패
- `getBackendBaseUrl(HttpServletRequest)` 메서드를 통한 동적 URL 생성의 필요성

**Vite 개발서버의 고급 기능**:
- `proxy.router` 함수로 요청별 동적 타깃 결정 가능
- `req.headers.host`에서 클라이언트 접속 정보 추출
- 개발환경에서만 동작하는 프록시의 한계와 클라이언트 사이드 URL 처리 필요성

**JWT 토큰 타입의 중요성**:
- 토큰 내용(payload)보다 타입(tokenType)이 권한 결정에 핵심
- 쿠키명과 토큰 타입의 일관성 유지 필요
- 토큰별 명확한 용도 구분으로 보안 강화
- 토큰 생명주기 관리의 중요성 (발급 → 사용 → 제거)

### 9-3. 문제 해결 방법론

**단계적 문제 분석**:
1. **현상 파악**: 각 문제를 독립적으로 분석 (토큰, 네트워크, OAuth, 이메일 분리)
2. **우선순위 결정**: 수정 난이도와 영향도 고려 (네트워크 > 토큰 > 프록시 > 이메일)
3. **근본 원인 탐구**: 표면적 증상이 아닌 구조적 문제 식별
4. **체계적 해결**: 인프라 → 백엔드 → 프론트엔드 → 테스트 순서

**효과적인 디버깅 기법**:
- **네트워크 레벨**: `curl`로 직접 백엔드 IPv4 접근 테스트
- **JWT 디코딩**: 토큰 내용 직접 확인으로 타입 문제 식별
- **브라우저 네트워크 탭**: 실제 요청 URL과 프록시 동작 추적
- **로그 패턴 분석**: Vite와 Spring Boot 로그를 통한 에러 발생 지점 특정
- **이메일 링크 추적**: 실제 발송된 이메일의 링크 URL 확인
- **단계별 검증**: 각 수정사항의 효과를 독립적으로 확인

### 9-4. 아키텍처 개선 원칙

**설정 관리 원칙**:
- **하드코딩 금지**: 모든 URL과 환경 정보는 환경변수로 관리
- **계층화된 우선순위**: 고정 도메인 > 동적 호스트 > 폴백값
- **환경별 분리**: 개발/스테이징/운영 환경의 명확한 구분
- **네트워크 레벨 고려**: 바인딩 주소와 CORS 정책의 일관성

**토큰 관리 원칙**:
- **타입별 명확한 역할**: ONBOARDING, ACCESS, EMAIL_VERIFY 등
- **일관된 네이밍**: 토큰 타입과 쿠키명의 일치
- **최소 권한 원칙**: 각 토큰은 필요한 최소한의 권한만 부여
- **생명주기 관리**: 토큰 발급, 갱신, 만료, 제거의 명확한 정책

**URL 생성 원칙**:
- **용도별 메서드 분리**: 프론트엔드 리다이렉트 vs 이메일 링크 생성
- **동적 생성 우선**: 현재 요청 기반 호스트 정보 활용
- **일관된 우선순위**: 고정 도메인 → 동적 호스트 → 폴백 순서

### 9-5. 장기적 개선 방향

**개발 환경 표준화**:
```bash
# 팀 공통 .env.template 파일
SERVER_PORT=9005
FRONTEND_PORT=5005
USE_DYNAMIC_HOST=true
ALLOWED_HOSTS=localhost,$(hostname -I | awk '{print $1}')
BACKEND_DOMAIN=
```

**설정 파일 구조 개선**:
```
# 현재 구조
src/main/resources/
├── application.yml  # 단일 파일로 모든 환경 설정

# 개선 예정 구조
src/main/resources/
├── application.yml           # 공통 기본 설정
├── application-dev.yml       # 개발환경 전용 설정
├── application-staging.yml   # 스테이징환경 전용 설정  
├── application-prod.yml      # 운영환경 전용 설정
└── application-test.yml      # 테스트환경 전용 설정
```

**환경별 YAML 분리 계획**:
```yaml
# application.yml (공통 기본 설정)
spring:
  application:
    name: tropical-app
  jpa:
    open-in-view: false
    show-sql: false  # 기본값, 개발환경에서 override

# application-dev.yml (개발환경)
spring:
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: create-drop
server:
  address: 0.0.0.0
app:
  frontend:
    use-dynamic-host: true

# application-prod.yml (운영환경)  
spring:
  jpa:
    show-sql: false
    hibernate:
      ddl-auto: validate
server:
  address: ${PROD_BIND_ADDRESS:127.0.0.1}
app:
  frontend:
    use-dynamic-host: false
    domain: ${FRONTEND_DOMAIN}
  backend:
    domain: ${BACKEND_DOMAIN}
```

**인증 시스템 고도화**:
- Refresh Token 자동 갱신 로직 개선
- 토큰별 세분화된 권한 체계 구축 (RBAC 도입)
- OAuth 상태값(state) 활용한 CSRF 방어 강화
- 멀티 디바이스 로그인 관리
- 이메일 인증 토큰 재사용 방지 메커니즘
- 온보딩 토큰의 단계별 권한 세분화

**CI/CD 파이프라인 개선**:
```yaml
# GitHub Actions - 환경별 자동 배포
- name: Deploy to Development
  env:
    SERVER_ADDRESS: "0.0.0.0"
    USE_DYNAMIC_HOST: "true"
  
- name: Deploy to Production  
  env:
    SERVER_ADDRESS: "${{ secrets.PROD_INTERNAL_IP }}"
    USE_DYNAMIC_HOST: "false"
    FRONTEND_DOMAIN: "https://tropical.com"
    BACKEND_DOMAIN: "https://api.tropical.com"
```

**모니터링 및 알림 시스템**:
- OAuth 실패율 및 토큰 만료율 모니터링
- 이메일 인증 링크 클릭률 및 성공률 추적
- 비정상적인 IPv4에서의 접근 시도 알림
- 환경별 설정 불일치 자동 감지
- 성능 메트릭스 (토큰 발급 시간, 프록시 응답 시간, 이메일 발송 지연)
- 온보딩 프로세스 완료율 및 중단 지점 분석

### 9-6. 팀 협업 개선사항

**문서화 체계**:
- 환경 설정 가이드 표준화
- 트러블슈팅 케이스 스터디 축적
- 신규 팀원 온보딩을 위한 환경 설정 자동화 스크립트
- URL 생성 로직 아키텍처 문서화
- 토큰 생명주기 관리 가이드

**개발 워크플로우 개선**:
- 환경변수 변경 시 팀 공유 프로세스
- IPv4 주소 변경 시 소셜 로그인 콜백 URL 업데이트 체크리스트
- 로컬 개발환경 문제 발생 시 빠른 진단을 위한 체크리스트
- 이메일 인증 시스템 변경 시 테스트 시나리오
- 온보딩 토큰 관련 변경 시 보안 검토 프로세스

이러한 해결 과정을 통해 **OAuth 인증 시스템의 복잡성과 네트워크 환경 설정의 중요성**, 그리고 **이메일 기반 인증 시스템의 도메인 종속성**을 깊이 이해하게 되었으며, 향후 유사한 인증 관련 문제 발생 시 체계적이고 신속한 해결이 가능한 방법론과 확장 가능한 아키텍처를 확립했습니다.

특히 **온보딩 토큰의 올바른 발급과 생명주기 관리**, **`server.address: 0.0.0.0` 설정의 중요성**, **Vite의 동적 프록시 라우터 활용**, 그리고 **이메일 인증 링크의 동적 URL 생성**이 핵심 해결책이었으며, 이는 향후 마이크로서비스나 컨테이너 환경에서도 응용 가능한 패턴입니다.

### 핵심 교훈

1. **토큰 관리의 정확성**: 토큰 타입과 쿠키명의 일치성, 권한별 명확한 구분이 인증 시스템의 안정성을 결정
2. **네트워크 설정의 기본**: 개발환경에서도 네트워크 바인딩과 CORS 설정이 팀 협업의 핵심 요소
3. **하드코딩의 위험성**: 단일 하드코딩된 URL이 전체 인증 플로우를 무력화시킬 수 있음
4. **동적 설정의 필요성**: 환경변수 기반 동적 URL 생성으로 개발 환경의 유연성 확보
5. **단계적 해결 접근**: 문제를 계층별로 나누어 해결함으로써 복잡한 시스템 이슈의 체계적 해결 가능

이 문서는 향후 유사한 인증 시스템 구축이나 트러블슈팅 상황에서 참고 자료로 활용할 수 있으며, 특히 다중 개발 환경과 동적 네트워크 환경에서의 인증 시스템 설계에 대한 실무적 가이드라인을 제공합니다.