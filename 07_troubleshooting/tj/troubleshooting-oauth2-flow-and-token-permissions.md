# 트러블슈팅: 소셜 로그인 시스템 OAuth2 플로우 및 토큰 인증 문제

본 문서는 **Spring Boot 기반 소셜 로그인 시스템** 개발 과정에서 발생한 **OAuth2 콜백 처리 및 온보딩 토큰 권한 관리** 문제의 원인과 해결 과정을 정리한 기술 문서입니다.

소셜 로그인(구글/카카오/네이버) 연동과 사용자 온보딩 프로세스에서 발생하는 인증 플로우 복잡성을 해결하여 안전하고 사용자 친화적인 인증 시스템을 구축하는 것을 목표로 합니다.

**작성자:** [왕택준](https://github.com/TJK98)

**작성일:** 2025년 9월 15일

**문서 버전:** v1.0

---

## 1. 문제 현상

### 1-1. OAuth2 콜백 실패

* **증상**: 소셜 로그인 버튼 클릭 시 404 에러 또는 "OAuth2 provider not found" 에러
* **추가 증상**: redirect_uri_mismatch로 인한 무한 로딩
* **영향 범위**: 모든 소셜 로그인 제공자(구글/카카오/네이버)

### 1-2. 온보딩 프로세스 권한 문제

* **증상**: 신규 사용자가 온보딩 페이지에서 401/403 에러 발생
* **문제**: 임시 토큰 발급 실패로 온보딩 진행 불가
* **보안 이슈**: 임시 토큰으로 보호된 API 접근 가능한 권한 누출

### 1-3. 토큰 관리 혼재

```java
// 예상 동작: 온보딩 완료 → 정식 ACCESS 토큰 발급 → 대시보드 접근
// 실제 동작: 온보딩 완료 → 여전히 임시 토큰 → 대시보드 접근 불가
```

### 1-4. 환경 정보

- **백엔드**: Spring Boot 3.5.5, Java 17
- **데이터베이스**: MariaDB
- **추가 라이브러리**: Spring Security, Spring OAuth2 Client, Spring Web, Spring Validation, Jackson, springdoc-openapi, JJWT, Logback
- **인증/인가**: HttpOnly 쿠키 기반 JWT (ACCESS/REFRESH, Onboarding/EmailVerify 토큰 분리)
- **OAuth2 클라이언트**: Google, Kakao, Naver
- **브라우저**: Chrome 140+
- **운영체제**: Windows 11, macOS Sequoia

---

## 2. 원인 분석

### 2-1. 1단계: OAuth2 설정 문제 의심

**가설**: application.yml의 OAuth2 클라이언트 설정 누락

```yaml
# 문제가 된 설정
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}  # 환경변수 미설정
            client-secret: ${GOOGLE_CLIENT_SECRET}
```

**결과**: 환경변수는 정상 설정되어 있음

### 2-2. 2단계: Redirect URI 불일치 의심

**가설**: 개발자 콘솔 설정과 실제 콜백 URL 미스매치

```bash
# Google Console 등록 URL
http://localhost:9005/login/oauth2/code/google

# 실제 Spring Security 기본 URL
http://localhost:9005/oauth2/code/google  # /login 누락
```

**결과**: URI 패턴 불일치 확인됨

### 2-3. 3단계: CORS 및 쿠키 설정 문제 의심

**가설**: 프론트엔드-백엔드 간 도메인 차이로 인한 쿠키 전달 실패

```yaml
# 잘못된 CORS 설정
cors:
  allowed-origins:
    - http://localhost:5005  # 프론트엔드 포트
    
# 실제 요청 도메인    
http://127.0.0.1:5005  # localhost vs 127.0.0.1 차이
```

**결과**: 도메인 불일치로 쿠키 전달 실패 확인

### 2-4. 4단계: JWT 토큰 타입 구분 부재 의심

**가설**: 온보딩 토큰과 정식 토큰의 권한 구분 실패

```java
// 문제가 된 토큰 생성
public String createOnboardingToken(Long userId, String email) {
    return Jwts.builder()
        .setSubject(String.valueOf(userId))
        .claim("email", email)
        // .claim("tokenType", "ONBOARDING") 누락
        .signWith(key, SignatureAlgorithm.HS512)
        .compact();
}
```

**결과**: 토큰 타입 구분 로직 누락 확인

### 2-5. 5단계: 근본 원인 발견

**핵심 발견**: SecurityConfig에서 권한별 엔드포인트 보호 설정 누락

```java
// 문제가 된 SecurityConfig
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
        .requestMatchers("/login/**", "/oauth2/**").permitAll()
        .anyRequest().authenticated();  // 모든 인증된 사용자 허용
        // → 임시 토큰으로도 모든 API 접근 가능
}
```

**원인**:
- 토큰 타입에 따른 세분화된 권한 제어 부재
- 온보딩 미완료 사용자의 API 접근 제한 로직 누락
- OAuth2 성공 핸들러에서 사용자 상태별 토큰 발급 로직 미흡

---

## 3. 디버깅 과정

### 3-1. 체계적 디버깅 방법론

**OAuth2 플로우 추적**: 각 단계별 로그 추가하여 흐름 파악

```java
@Component
public class OAuth2AuthenticationSuccessHandler 
    extends SimpleUrlAuthenticationSuccessHandler {
    
    @Override
    public void onAuthenticationSuccess(HttpServletRequest request,
                                      HttpServletResponse response,
                                      Authentication authentication) {
        log.info("=== OAuth2 인증 성공 ===");
        log.info("제공자: {}", getProvider(authentication));
        log.info("사용자 이메일: {}", getEmail(authentication));
        
        // 기존 사용자 확인
        User user = findOrCreateUser(authentication);
        log.info("사용자 온보딩 완료 여부: {}", user.isOnboardingCompleted());
        
        if (user.isOnboardingCompleted()) {
            log.info("기존 사용자 → 정식 토큰 발급");
        } else {
            log.info("신규 사용자 → 임시 토큰 발급");
        }
    }
}
```

**JWT 토큰 검증**: 발급된 토큰의 권한과 클레임 확인

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                  HttpServletResponse response,
                                  FilterChain filterChain) {
        String token = extractTokenFromCookie(request);
        
        if (token != null && jwtTokenProvider.validateToken(token)) {
            Claims claims = jwtTokenProvider.getClaimsFromToken(token);
            
            log.debug("=== JWT 토큰 정보 ===");
            log.debug("사용자 ID: {}", claims.getSubject());
            log.debug("토큰 타입: {}", claims.get("tokenType"));
            log.debug("권한: {}", getAuthorities(claims));
        }
    }
}
```

**권한 검증 로직**: SecurityContext의 권한 정보 확인

```java
@RestController
public class AuthController {
    
    @GetMapping("/api/me")
    public ResponseEntity<UserInfo> getCurrentUser() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        
        log.debug("=== 현재 인증 정보 ===");
        log.debug("인증 여부: {}", auth.isAuthenticated());
        log.debug("권한 목록: {}", auth.getAuthorities());
        log.debug("Principal: {}", auth.getPrincipal());
        
        // 온보딩 토큰으로 접근 시 차단
        if (hasOnlyOnboardingRole(auth)) {
            return ResponseEntity.status(HttpStatus.FORBIDDEN).build();
        }
        
        return ResponseEntity.ok(getCurrentUserInfo());
    }
}
```

---

## 4. 해결 과정

### 4-1. 실패한 해결책들

**A. 단순 Redirect URI 수정**

```yaml
# Google Console에서만 URL 변경
http://localhost:9005/oauth2/code/google
```

**실패 이유**: 프론트엔드 URL 설정도 함께 수정해야 했음

**B. 환경변수만 재설정**

```bash
# .env 파일만 수정
GOOGLE_CLIENT_ID=new_client_id
FRONTEND_BASE_URL=http://localhost:5005
```

**실패 이유**: application.yml의 CORS 설정이 하드코딩되어 있었음

**C. SecurityConfig 단순 수정**

```java
// 모든 요청에 ROLE_USER 권한 요구
.anyRequest().hasRole("USER")
```

**실패 이유**: 온보딩 토큰도 ROLE_USER를 가져서 보호 API 접근 가능

### 4-2. 최종 해결책: 토큰 타입별 세분화된 권한 제어

**1. JWT 토큰에 타입 정보 추가**

```java
@Component
public class JwtTokenProvider {
    
    // 온보딩용 임시 토큰
    public String createOnboardingToken(Long userId, String email) {
        return Jwts.builder()
            .setSubject(String.valueOf(userId))
            .claim("email", email)
            .claim("tokenType", "ONBOARDING")  // 토큰 타입 명시
            .setExpiration(new Date(System.currentTimeMillis() + ONBOARDING_EXPIRATION))
            .signWith(key, SignatureAlgorithm.HS512)
            .compact();
    }
    
    // 정식 액세스 토큰
    public String createAccessToken(Long userId, String email) {
        return Jwts.builder()
            .setSubject(String.valueOf(userId))
            .claim("email", email)
            .claim("tokenType", "ACCESS")  // 토큰 타입 명시
            .setExpiration(new Date(System.currentTimeMillis() + ACCESS_EXPIRATION))
            .signWith(key, SignatureAlgorithm.HS512)
            .compact();
    }
}
```

**2. 권한별 엔드포인트 보호 설정**

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/login/**", "/oauth2/**").permitAll()
                .requestMatchers("/api/auth/onboarding/**").hasRole("ONBOARDING")  // 온보딩 전용
                .requestMatchers("/api/**").hasRole("USER")  // 정식 사용자 전용
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .successHandler(oAuth2AuthenticationSuccessHandler)
            )
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }
}
```

**3. JwtAuthenticationFilter에서 권한 부여 로직 개선**

```java
@Override
protected void doFilterInternal(HttpServletRequest request,
                              HttpServletResponse response, 
                              FilterChain filterChain) throws ServletException, IOException {
    String token = extractTokenFromCookie(request);
    
    if (token != null && jwtTokenProvider.validateToken(token)) {
        Claims claims = jwtTokenProvider.getClaimsFromToken(token);
        String tokenType = claims.get("tokenType", String.class);
        
        Collection<? extends GrantedAuthority> authorities;
        
        if ("ONBOARDING".equals(tokenType)) {
            // 온보딩 토큰 → 제한된 권한
            authorities = List.of(new SimpleGrantedAuthority("ROLE_ONBOARDING"));
        } else if ("ACCESS".equals(tokenType)) {
            // 정식 토큰 → 전체 권한  
            authorities = List.of(new SimpleGrantedAuthority("ROLE_USER"));
        } else {
            // 알 수 없는 토큰 → 권한 없음
            authorities = Collections.emptyList();
        }
        
        UsernamePasswordAuthenticationToken authentication = 
            new UsernamePasswordAuthenticationToken(claims.getSubject(), null, authorities);
        SecurityContextHolder.getContext().setAuthentication(authentication);
    }
    
    filterChain.doFilter(request, response);
}
```

**4. OAuth2 성공 핸들러에서 상태별 토큰 발급**

```java
@Override
public void onAuthenticationSuccess(HttpServletRequest request,
                                  HttpServletResponse response,
                                  Authentication authentication) throws IOException {
    User user = findOrCreateUser(authentication);
    
    if (user.isOnboardingCompleted()) {
        // 기존 사용자 → 정식 토큰 발급
        String accessToken = jwtTokenProvider.createAccessToken(user.getId(), user.getEmail());
        String refreshToken = jwtTokenProvider.createRefreshToken(user.getId());
        
        response.addCookie(CookieUtil.build("ACCESS_TOKEN", accessToken, accessMaxAge));
        response.addCookie(CookieUtil.build("REFRESH_TOKEN", refreshToken, refreshMaxAge));
        
        // 대시보드로 리다이렉트
        response.sendRedirect(frontendBaseUrl + "/dashboard");
    } else {
        // 신규 사용자 → 온보딩 토큰 발급
        String tempToken = jwtTokenProvider.createOnboardingToken(user.getId(), user.getEmail());
        response.addCookie(CookieUtil.build("ACCESS_TOKEN", tempToken, 1800));
        
        // 온보딩 페이지로 리다이렉트  
        response.sendRedirect(frontendBaseUrl + "/onboarding");
    }
}
```

**5. 온보딩 완료 API에서 토큰 교체**

```java
@PostMapping("/api/auth/onboarding/complete")
@PreAuthorize("hasRole('ONBOARDING')")  // 온보딩 토큰만 접근 가능
public ResponseEntity<?> completeOnboarding(@RequestBody OnboardingRequest request,
                                          HttpServletResponse response) {
    // 필수 동의 검증
    if (!isRequiredConsentsProvided(request.getConsents())) {
        return ResponseEntity.badRequest()
            .body(Map.of("error", "CONSENT_REQUIRED", "message", "필수 동의가 필요합니다."));
    }
    
    // 사용자 정보 업데이트
    Long userId = getCurrentUserId();
    userService.completeOnboarding(userId, request);
    
    // 임시 토큰을 정식 토큰으로 교체
    User user = userService.findById(userId);
    String accessToken = jwtTokenProvider.createAccessToken(user.getId(), user.getEmail());
    String refreshToken = jwtTokenProvider.createRefreshToken(user.getId());
    
    response.addCookie(CookieUtil.build("ACCESS_TOKEN", accessToken, accessMaxAge));
    response.addCookie(CookieUtil.build("REFRESH_TOKEN", refreshToken, refreshMaxAge));
    
    return ResponseEntity.ok(Map.of("message", "온보딩이 완료되었습니다."));
}
```

**성공 이유**:
- 토큰 타입별로 명확한 권한 분리
- SecurityConfig에서 엔드포인트별 세분화된 접근 제어
- OAuth2 플로우와 온보딩 프로세스의 명확한 분리
- 상태 전이에 따른 토큰 교체 로직 구현

---

5. 테스트 검증

5-1. Postman을 통한 API 테스트

OAuth2 플로우 검증:

# 1. 소셜 로그인 URL 테스트
GET http://localhost:9005/oauth2/authorization/google
→ Google 로그인 페이지로 리다이렉트 확인

# 2. 콜백 URL 테스트 (브라우저에서 확인)
http://localhost:9005/oauth2/code/google?code=...
→ 프론트엔드 온보딩/대시보드 페이지로 리다이렉트 확인


권한별 API 접근 테스트:

# 온보딩 토큰으로 보호된 API 접근 시도
GET http://localhost:9005/api/me
Cookie: ACCESS_TOKEN=온보딩토큰값
→ 403 Forbidden 확인

# 온보딩 토큰으로 온보딩 API 접근
POST http://localhost:9005/api/auth/onboarding/complete
Cookie: ACCESS_TOKEN=온보딩토큰값
→ 200 OK 확인

# 정식 토큰으로 모든 API 접근
GET http://localhost:9005/api/me
Cookie: ACCESS_TOKEN=정식토큰값
→ 200 OK 및 사용자 정보 반환 확인


5-2. Swagger UI를 통한 API 문서 검증

온보딩 API 테스트:

/api/auth/onboarding/complete 엔드포인트에서 필수 동의 누락 시 CONSENT_REQUIRED 에러 응답 확인

정상 온보딩 완료 시 새로운 ACCESS_TOKEN과 REFRESH_TOKEN 쿠키 설정 확인

5-3. 서버 로그를 통한 플로우 검증

OAuth2 성공 핸들러 로그:

2025-09-15 14:30:15 INFO  OAuth2AuthenticationSuccessHandler - === OAuth2 인증 성공 ===
2025-09-15 14:30:15 INFO  OAuth2AuthenticationSuccessHandler - 제공자: google
2025-09-15 14:30:15 INFO  OAuth2AuthenticationSuccessHandler - 사용자 이메일: test@gmail.com
2025-09-15 14:30:15 INFO  OAuth2AuthenticationSuccessHandler - 사용자 온보딩 완료 여부: false
2025-09-15 14:30:15 INFO  OAuth2AuthenticationSuccessHandler - 신규 사용자 → 임시 토큰 발급


JWT 필터 로그:

2025-09-15 14:31:20 DEBUG JwtAuthenticationFilter - === JWT 토큰 정보 ===
2025-09-15 14:31:20 DEBUG JwtAuthenticationFilter - 사용자 ID: 1
2025-09-15 14:31:20 DEBUG JwtAuthenticationFilter - 토큰 타입: ONBOARDING
2025-09-15 14:31:20 DEBUG JwtAuthenticationFilter - 권한: [ROLE_ONBOARDING]

---

## 6. 관련 이슈 및 예방책

### 6-1. OAuth2 설정 안티패턴

**위험한 패턴들**:

```yaml
# 하드코딩된 URL
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            redirect-uri: http://localhost:9005/oauth2/code/google  # 환경별 다른 포트

# CORS 설정 누락
cors:
  allowed-origins: "*"  # 보안 위험
```

**안전한 패턴들**:

```yaml
# 환경변수 활용
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            redirect-uri: ${APP_BASE_URL}/oauth2/code/google
            
# 명시적 CORS 설정            
cors:
  allowed-origins:
    - ${FRONTEND_BASE_URL}
    - http://localhost:${FRONTEND_PORT:5005}
```

### 6-2. JWT 토큰 보안 체크리스트

**토큰 설계 시 고려사항**:
- 토큰 타입별 명확한 권한 분리
- 만료 시간 차등 설정 (온보딩: 30분, 액세스: 1시간)
- 민감 정보는 클레임에 포함하지 않기
- 토큰 갱신 전략 수립

**권한 제어 패턴**:
```java
// 좋은 예: 명시적 권한 검증
@PreAuthorize("hasRole('USER') and !hasRole('ONBOARDING')")
public ResponseEntity<?> sensitiveApi() { }

// 나쁜 예: 모호한 권한 설정
@PreAuthorize("isAuthenticated()")  // 온보딩 토큰도 통과
public ResponseEntity<?> sensitiveApi() { }
```

---

## 7. 결론 및 배운 점

### 7-1. 주요 성과

1. **OAuth2 플로우 안정화**: Redirect URI 불일치 문제 해결로 모든 소셜 로그인 제공자 정상 동작
2. **세분화된 권한 제어**: 토큰 타입별 명확한 API 접근 권한 분리 구현
3. **보안 강화**: 온보딩 미완료 사용자의 보호된 API 접근 차단
4. **사용자 경험 개선**: 사용자 상태에 따른 적절한 페이지 리다이렉트 구현

### 7-2. 기술적 학습

**OAuth2와 Spring Security 통합**:
- OAuth2AuthenticationSuccessHandler에서 사용자 상태별 토큰 발급의 중요성
- SecurityConfig의 엔드포인트별 권한 설정이 전체 보안에 미치는 영향
- JWT 클레임에 `tokenType` 추가로 권한 제어 로직 구현 가능

**토큰 기반 인증의 복잡성**:
- 임시 토큰(온보딩)과 정식 토큰(액세스)의 명확한 분리 필요
- JwtAuthenticationFilter에서 토큰 타입에 따른 권한 부여 로직의 중요성
- 쿠키 기반 토큰 전달에서 도메인 일치(localhost vs 127.0.0.1) 문제

### 7-3. 디버깅 방법론 확립

**체계적 문제 해결 과정**:
1. **로그 기반 추적**: OAuth2 핸들러와 JWT 필터에 상세 로그 추가하여 플로우 파악
2. **브라우저 도구 활용**: 네트워크 탭으로 실제 요청 URL과 쿠키 전달 상황 확인
3. **Postman 검증**: API별 권한 테스트로 예상 동작과 실제 동작 비교
4. **토큰 내용 확인**: JWT 디코더로 클레임 정보 직접 분석

### 7-4. 실무 적용 포인트

**설정 관리**:
- 하드코딩된 URL 대신 환경변수 활용으로 환경별 유연성 확보
- 개발자 콘솔 Redirect URI와 실제 Spring Security URL 패턴 일치 필수

**보안 설계**:
- `@PreAuthorize`보다 `HttpSecurity` 설정에서 엔드포인트별 권한 제어
- 토큰 타입별 차등 만료 시간 설정 (온보딩: 30분, 액세스: 1시간)
- CORS 설정에서 와일드카드(*) 사용 지양, 명시적 도메인 지정

이번 트러블슈팅을 통해 OAuth2 기반 소셜 로그인 시스템에서 **토큰 타입별 권한 분리 패턴**의 중요성을 깊이 이해했으며, 향후 유사한 인증 시스템 구축 시 체계적인 접근이 가능한 경험을 쌓았습니다.