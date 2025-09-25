# 소셜 로그인 구현 문서

본 문서는 TropiCal 프로젝트의 소셜 로그인 구현 방식을 설명한 기술 문서입니다. Google, Kakao, Naver 소셜 로그인 연동 과정과 Spring Security OAuth2 기반의 통합 인증 처리 방식을 다룹니다.

**작성자:** [왕택준](https://github.com/TJK98)

**문서 버전:** v1.0

**대상 독자:**

- **백엔드 개발자**: Spring Security OAuth2, Success/Failure Handler, SocialAccount 매핑 로직을 이해하고 유지보수해야 하는 개발자
- **프론트엔드 개발자**: React 컴포넌트(`SocialLoginButtons`, `useAuth`)와 백엔드 OAuth 엔드포인트(`/oauth2/authorization/{provider}`) 연동을 담당하는 개발자
- **DevOps / 인프라 엔지니어**: 각 소셜 제공자의 Client ID/Secret 관리, Redirect URI 환경 변수 설정, HTTPS 및 쿠키 보안 설정을 책임지는 운영 담당자
- **QA / 테스트 엔지니어**: 소셜 로그인 성공/실패, 동의 거부, 이메일 미제공 등 다양한 시나리오를 테스트하는 품질 보증 담당자
- **신규 합류자**: 프로젝트의 소셜 로그인 플로우를 빠르게 이해해야 하는 팀 신규 멤버
- **프로덕트 매니저 (PM)**: 인증/인가 설계가 사용자 경험과 보안에 어떤 영향을 주는지 파악하고, 기획 방향을 설정하는 담당자

## 목차

1. [개요](#개요)
2. [동적 URL 시스템 구성](#동적-url-시스템-구성)
3. [OAuth2 플로우 이해](#oauth2-플로우-이해)
4. [Google 로그인 구현](#google-로그인-구현)
5. [Kakao 로그인 구현](#kakao-로그인-구현)
6. [Naver 로그인 구현](#naver-로그인-구현)
7. [통합 인증 처리](#통합-인증-처리)
8. [프론트엔드 연동](#프론트엔드-연동)
9. [에러 처리 및 예외 상황](#에러-처리-및-예외-상황)
10. [보안 고려사항](#보안-고려사항)
11. [트러블슈팅](#트러블슈팅)

---

## 개요

TropiCal 프로젝트는 Spring Security OAuth2를 기반으로 Google, Kakao, Naver의 소셜 로그인을 통합 구현했습니다. 각 제공자별 특성을 고려한 맞춤형 설정과 일관된 사용자 경험을 제공하며, 환경변수 기반 동적 URL 시스템으로 개발 환경의 유연성을 확보했습니다.

### 지원하는 소셜 제공자

- **Google**: OpenID Connect 표준 준수
- **Kakao**: REST API 기반 커스텀 구현
- **Naver**: REST API 기반 커스텀 구현

### 핵심 설계 원칙

- **표준 준수**: OAuth2/OIDC 표준을 최대한 활용
- **확장성**: 새로운 소셜 제공자 추가 용이
- **보안성**: PKCE, State 파라미터 등 보안 강화
- **사용자 경험**: 실패 시 명확한 안내 및 대안 제공
- **환경 적응성**: localhost와 IPv4 주소 접속 모두 지원하는 동적 URL 시스템

---

## 동적 URL 시스템 구성

### 1. 핵심 아키텍처

TropiCal의 소셜 로그인은 **환경변수 기반 동적 URL 생성 시스템**을 사용합니다. 이를 통해 개발 환경에서 IPv4 주소 변경에 자동으로 대응할 수 있습니다.

**우선순위 체계:**
1. 운영환경 고정 도메인 (FRONTEND_DOMAIN, BACKEND_DOMAIN)
2. 동적 호스트 (현재 요청 호스트 기반)
3. localhost 폴백

### 2. 환경변수 설정

**백엔드 .env 설정:**
```bash
# 서버 네트워크 설정
SERVER_PORT=9005
server.address=0.0.0.0  # 모든 네트워크 인터페이스 바인딩

# 동적 URL 설정
USE_DYNAMIC_HOST=true
FRONTEND_PORT=5005
FRONTEND_DOMAIN=  # 운영환경용 (빈 값이면 동적 생성)
BACKEND_DOMAIN=   # 운영환경용 (빈 값이면 동적 생성)

# CORS 허용 호스트
ALLOWED_HOSTS=localhost,IPv4주소1,IPv4주소2

# OAuth2 소셜 로그인 설정
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret
KAKAO_CLIENT_ID=your_kakao_client_id
KAKAO_CLIENT_SECRET=your_kakao_client_secret
NAVER_CLIENT_ID=your_naver_client_id
NAVER_CLIENT_SECRET=your_naver_client_secret
```

**프론트엔드 .env 설정:**
```bash
# Vite 환경변수 (VITE_ 접두사 필수)
VITE_USE_DYNAMIC_HOST=true
VITE_BACKEND_PORT=9005
VITE_BACKEND_DOMAIN=  # 운영환경용 고정 도메인
```

### 3. application.yml 설정

```yaml
server:
  port: ${SERVER_PORT:9005}
  address: 0.0.0.0  # 모든 네트워크 인터페이스 바인딩

app:
  frontend:
    port: ${FRONTEND_PORT:5005}
    use-dynamic-host: ${USE_DYNAMIC_HOST:true}
    domain: ${FRONTEND_DOMAIN:}
  backend:
    domain: ${BACKEND_DOMAIN:}
  allowed-hosts: ${ALLOWED_HOSTS:localhost}

spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, profile, email
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            authorization-grant-type: authorization_code

          kakao:
            client-id: ${KAKAO_CLIENT_ID}
            client-secret: ${KAKAO_CLIENT_SECRET}
            client-authentication-method: client_secret_post
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            scope: account_email, profile_nickname

          naver:
            client-id: ${NAVER_CLIENT_ID}
            client-secret: ${NAVER_CLIENT_SECRET}
            client-authentication-method: client_secret_post
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            scope: name, email

        provider:
          kakao:
            authorization-uri: https://kauth.kakao.com/oauth/authorize
            token-uri: https://kauth.kakao.com/oauth/token
            user-info-uri: https://kapi.kakao.com/v2/user/me
            user-name-attribute: id

          naver:
            authorization-uri: https://nid.naver.com/oauth2.0/authorize
            token-uri: https://nid.naver.com/oauth2.0/token
            user-info-uri: https://openapi.naver.com/v1/nid/me
            user-name-attribute: response
```

---

## OAuth2 플로우 이해

### 1. Authorization Code Grant 플로우

```
1. [사용자] → [프론트엔드]: 소셜 로그인 버튼 클릭
2. [프론트엔드] → [백엔드]: GET http://현재호스트:9005/oauth2/authorization/{provider}
3. [백엔드] → [소셜 제공자]: 인증 요청 (client_id, redirect_uri, scope)
4. [소셜 제공자] → [사용자]: 로그인 화면 표시
5. [사용자] → [소셜 제공자]: 로그인 및 권한 승인
6. [소셜 제공자] → [백엔드]: 인가 코드 전달 (GET http://현재호스트:9005/login/oauth2/code/{provider})
7. [백엔드] → [소셜 제공자]: Access Token 요청 (인가 코드 교환)
8. [소셜 제공자] → [백엔드]: Access Token 응답
9. [백엔드] → [소셜 제공자]: 사용자 정보 요청 (Access Token 사용)
10. [소셜 제공자] → [백엔드]: 사용자 정보 응답
11. [백엔드]: 사용자 생성/조회 및 JWT 토큰 발급
12. [백엔드] → [프론트엔드]: http://현재호스트:5005로 리다이렉트 (JWT 쿠키 설정)
```

### 1-1. 동적 URL 생성 과정

```
요청: http://IPv4주소:5005 에서 Google 로그인 버튼 클릭
↓
프론트엔드: 동적 백엔드 URL 생성 → http://IPv4주소:9005
↓
OAuth 요청: http://IPv4주소:9005/oauth2/authorization/google
↓
Google 콜백: http://IPv4주소:9005/login/oauth2/code/google
↓
백엔드: 동적 프론트엔드 URL 생성 → http://IPv4주소:5005
↓
리다이렉트: http://IPv4주소:5005/onboarding 또는 /dashboard
```

---

### 2. 다른 OAuth2 Grant Type과의 비교

#### Implicit Grant (암묵적 방식)

- **특징**: Access Token을 프론트엔드가 직접 전달받음
- **장점**: 서버 단계를 거치지 않아 구현이 단순함
- **단점**: 브라우저 환경에서 토큰이 그대로 노출 → XSS, 토큰 탈취 위험 큼
- **현황**: SPA 초창기에 많이 쓰였지만, 현재는 보안 취약점 때문에 **거의 사용되지 않음**

---

#### Client Credentials Grant (클라이언트 자격 증명 방식)

- **특징**: 사용자 개입 없이 서버-서버 간 인증에서 사용
- **장점**: 마이크로서비스, 외부 API 호출 등에는 적합
- **단점**: 사용자 로그인과는 무관, 사용자 식별 불가
- **현황**: TropiCal과 같은 **사용자 로그인 시나리오에는 적합하지 않음**

---

#### Resource Owner Password Credentials Grant (ROPC, 패스워드 방식)

- **특징**: 사용자의 ID/PW를 애플리케이션이 직접 받아서 Access Token 발급
- **장점**: 구조가 단순하고 빠름
- **단점**: 사용자가 서비스에 비밀번호를 직접 제공해야 하므로 보안 위험(중간 탈취, 저장 위험) 큼
- **현황**: 과거 레거시 앱에서 사용되었으나, 현재는 **완전히 권장되지 않음**

---

### 3. Authorization Code Grant 선택

- **선택 이유**

  * 프론트엔드에서 토큰이 노출되지 않음 (HttpOnly 쿠키 활용)
  * Google, Kakao, Naver 등 주요 소셜 제공자들이 권장하는 표준 방식
  * Refresh Token과 함께 사용 가능 → 세션 만료 시 끊김 없는 사용자 경험
  * PKCE 적용 시 추가적인 보안 강화 (코드 탈취 방지)

- **장점**

  * 토큰이 서버에서만 관리 → 보안성 높음
  * 소셜 제공자의 최신 표준(OIDC)과 완벽 호환
  * 다국적 서비스 확장 시에도 재사용 가능

- **단점**

  * 구현 복잡도가 상대적으로 높음
  * 서버에서 추가적인 토큰 교환 로직 필요

### 4. Spring Security OAuth2 설정

```java
// SecurityConfig.java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")
                .successHandler(oAuth2AuthenticationSuccessHandler)
                .failureHandler(oAuth2AuthenticationFailureHandler)
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(customOAuth2UserService)
                )
            )
            .build();
    }
}
```

---

## Google 로그인 구현

### 1. Google Cloud Console 설정 (동적 환경 지원)

```bash
# Google Cloud Console (console.cloud.google.com)에서 설정
1. 프로젝트 생성 또는 선택
2. API 및 서비스 → 사용자 인증 정보
3. OAuth 2.0 클라이언트 ID 생성
   - 애플리케이션 유형: 웹 애플리케이션
   - 승인된 자바스크립트 원본: 
     * http://localhost:5005
     * http://IPv4주소1:5005
     * http://IPv4주소2:5005
   - 승인된 리디렉션 URI:
     * http://localhost:9005/login/oauth2/code/google
     * http://IPv4주소1:9005/login/oauth2/code/google
     * http://IPv4주소2:9005/login/oauth2/code/google
```

### 2. Google 사용자 정보 처리

```java
// OAuth2AuthenticationSuccessHandler.java
private SocialUserInfo extractSocialUserInfo(SocialAccount.SocialProvider provider, OAuth2User oauth2User) {
    Map<String, Object> attributes = oauth2User.getAttributes();

    return switch (provider) {
        case GOOGLE -> {
            yield new SocialUserInfo(
                (String) attributes.get("sub"),           // Google 사용자 ID
                (String) attributes.get("email"),        // 이메일
                (String) attributes.get("name")          // 이름
            );
        }
        // ... 다른 제공자들
    };
}
```

---

## Kakao 로그인 구현

### 1. Kakao Developers 설정 (동적 환경 지원)

```bash
# Kakao Developers (developers.kakao.com)에서 설정
1. 애플리케이션 추가
2. 플랫폼 설정 → Web 추가
   - 사이트 도메인: 
     * http://localhost:5005
     * http://IPv4주소1:5005
     * http://IPv4주소2:5005
3. 카카오 로그인 → 활성화 설정
   - Redirect URI: 
     * http://localhost:9005/login/oauth2/code/kakao
     * http://IPv4주소1:9005/login/oauth2/code/kakao
     * http://IPv4주소2:9005/login/oauth2/code/kakao
4. 동의항목 → 개인정보 항목 설정
   - 닉네임, 프로필 사진, 이메일 (선택 동의)
```

### 2. Kakao 사용자 정보 처리

```java
case KAKAO -> {
    Map<String, Object> kakaoAccount = (Map<String, Object>) attributes.get("kakao_account");
    Map<String, Object> profile = (Map<String, Object>) kakaoAccount.get("profile");

    yield new SocialUserInfo(
        String.valueOf(attributes.get("id")),    // 카카오 사용자 ID
        (String) kakaoAccount.get("email"),      // 이메일
        (String) profile.get("nickname")         // 닉네임
    );
}
```

---

## Naver 로그인 구현

### 1. Naver Developers 설정 (동적 환경 지원)

```bash
# Naver Developers (developers.naver.com)에서 설정
1. 애플리케이션 등록
2. API 설정
   - 서비스 URL: 
     * http://localhost:5005
     * http://IPv4주소1:5005
     * http://IPv4주소2:5005
   - Callback URL:
     * http://localhost:9005/login/oauth2/code/naver
     * http://IPv4주소1:9005/login/oauth2/code/naver
     * http://IPv4주소2:9005/login/oauth2/code/naver
3. 제공 정보 선택
   - 이메일 주소, 이름, 연락처 전화번호 등
```

### 2. Naver 사용자 정보 처리

```java
case NAVER -> {
    Map<String, Object> naverResponse = (Map<String, Object>) attributes.get("response");

    yield new SocialUserInfo(
        (String) naverResponse.get("id"),        // 네이버 사용자 ID
        (String) naverResponse.get("email"),     // 이메일
        (String) naverResponse.get("name")       // 이름
    );
}
```

---

## 통합 인증 처리

### 1. 동적 리다이렉트 URL 생성

```java
// OAuth2AuthenticationSuccessHandler.java
@Value("${app.frontend.use-dynamic-host:true}")
private boolean useDynamicHost;

@Value("${app.frontend.domain:}")
private String frontendDomain;

@Value("${app.frontend.port:5005}")
private String frontendPort;

/**
 * 환경변수 기반 프론트엔드 URL 동적 생성
 */
private String getFrontendBaseUrl(HttpServletRequest request) {
    // 1순위: 운영환경 고정 도메인
    if (!frontendDomain.isEmpty()) {
        log.debug("고정 프론트엔드 도메인 사용: {}", frontendDomain);
        return frontendDomain;
    }

    // 2순위: 개발환경 동적 호스트
    if (useDynamicHost) {
        String scheme = request.getScheme();
        String serverName = request.getServerName();
        String dynamicUrl = String.format("%s://%s:%s", scheme, serverName, frontendPort);
        log.debug("동적 프론트엔드 URL 생성: {} (요청 호스트: {})", dynamicUrl, serverName);
        return dynamicUrl;
    }

    // 3순위: localhost 폴백
    String fallbackUrl = String.format("http://localhost:%s", frontendPort);
    log.debug("폴백 프론트엔드 URL 사용: {}", fallbackUrl);
    return fallbackUrl;
}
```

### 2. 통합 OAuth 성공 처리

```java
@Override
public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,
                                    Authentication authentication) throws IOException {
    OAuth2AuthenticationToken oauth2Token = (OAuth2AuthenticationToken) authentication;
    OAuth2User oauth2User = oauth2Token.getPrincipal();
    String registrationId = oauth2Token.getAuthorizedClientRegistrationId();

    try {
        // 소셜 제공자 변환
        SocialAccount.SocialProvider provider = socialAccountService.getSocialProviderFromRegistrationId(registrationId);

        // 소셜 사용자 정보 추출
        SocialUserInfo socialUserInfo = extractSocialUserInfo(provider, oauth2User);

        // 기존 소셜 계정 조회
        Optional<SocialAccount> existingSocialAccount = socialAccountService
                .findActiveSocialAccount(provider, socialUserInfo.providerId());

        User user;
        boolean isNewUser = false;

        if (existingSocialAccount.isPresent()) {
            // 기존 사용자 로그인
            Long userId = existingSocialAccount.get().getUser().getId();
            user = userService.getById(userId);
            userService.updateLastLoginTime(user.getId());
        } else {
            // 신규 사용자 생성
            user = userService.createSocialUser(socialUserInfo.email(), socialUserInfo.nickname());
            socialAccountService.createOrGetSocialAccount(user, provider, socialUserInfo.providerId());
            isNewUser = true;
        }

        // 동적 리다이렉트 URL 결정 및 토큰 발급
        String redirectUrl = determineRedirectUrl(request, user, isNewUser);
        issueTokens(user, response);

        log.info("소셜 로그인 성공 - 제공자: {}, 사용자 ID: {}, 리다이렉트: {}", 
                 provider, user.getId(), redirectUrl);

        getRedirectStrategy().sendRedirect(request, response, redirectUrl);

    } catch (Exception e) {
        handleAuthenticationFailure(request, response, registrationId, e);
    }
}

private String determineRedirectUrl(HttpServletRequest request, User user, boolean isNewUser) {
    String baseUrl = getFrontendBaseUrl(request);  // 동적 URL 생성

    // 화이트리스트 검증
    if (!isValidRedirectUrl(baseUrl)) {
        throw new SecurityException("유효하지 않은 리다이렉트 URL입니다");
    }

    if (isNewUser || !user.isOnboardingCompleted()) {
        return baseUrl + "/onboarding";
    } else {
        return baseUrl + "/dashboard";
    }
}
```

### 3. 토큰 발급 전략

```java
private void issueTokens(User user, HttpServletResponse response) {
    if (!user.isOnboardingCompleted()) {
        // 온보딩용 임시 토큰 발급 (30분)
        String onboardingToken = jwtTokenProvider.createOnboardingToken(user.getId(), user.getEmail());
        response.addCookie(CookieUtil.build("ONBOARDING_TOKEN", onboardingToken, 1800));
        log.info("온보딩용 임시 토큰 발급 - 사용자 ID: {}", user.getId());
    } else {
        // 정식 JWT 토큰 발급
        String accessToken = jwtTokenProvider.createAccessToken(user.getId(), user.getEmail());
        String refreshToken = jwtTokenProvider.createRefreshToken(user.getId());

        int accessMaxAge = jwtTokenProvider.getAccessMaxAge();
        int refreshMaxAge = jwtTokenProvider.getRefreshMaxAge();

        response.addCookie(CookieUtil.build("ACCESS_TOKEN", accessToken, accessMaxAge));
        response.addCookie(CookieUtil.build("REFRESH_TOKEN", refreshToken, refreshMaxAge));
        log.info("정식 JWT 토큰 발급 - 사용자 ID: {}", user.getId());
    }
}
```

---

## 프론트엔드 연동

### 1. 동적 백엔드 URL 생성

```javascript
// api.js
/**
 * 동적 백엔드 URL 생성 함수
 * 
 * 우선순위:
 * 1. VITE_BACKEND_DOMAIN (운영환경 고정 도메인)
 * 2. 동적 호스트 (현재 브라우저 호스트 + 백엔드 포트)
 * 3. localhost 폴백
 */
export function getBackendBaseUrl() {
    // 1순위: 운영환경 고정 도메인
    const backendDomain = import.meta.env.VITE_BACKEND_DOMAIN;
    if (backendDomain && backendDomain.trim()) {
        return backendDomain;
    }

    // 2순위: 개발환경 동적 호스트
    const useDynamicHost = import.meta.env.VITE_USE_DYNAMIC_HOST === 'true';
    if (useDynamicHost) {
        const currentHost = window.location.hostname;
        const backendPort = import.meta.env.VITE_BACKEND_PORT || '9005';
        const protocol = window.location.protocol;
        return `${protocol}//${currentHost}:${backendPort}`;
    }

    // 3순위: localhost 폴백
    return `http://localhost:9005`;
}

/**
 * OAuth URL 생성 함수
 */
export function getOAuthURL(provider) {
    const backendUrl = getBackendBaseUrl();
    return `${backendUrl}/oauth2/authorization/${provider}`;
}

export const API_BASE_URL = getBackendBaseUrl();

export const OAUTH_URLS = {
    google: getOAuthURL('google'),
    kakao: getOAuthURL('kakao'),
    naver: getOAuthURL('naver'),
};
```

### 2. 소셜 로그인 컴포넌트 (동적 URL 적용)

```javascript
// SocialLoginButtons.jsx
import React from 'react';
import { getOAuthURL } from '../api/api';

const SocialLoginButtons = () => {
    const handleSocialLogin = (provider) => {
        // 동적으로 OAuth URL 생성
        const loginUrl = getOAuthURL(provider);
        
        // 현재 페이지 URL을 state로 전달하여 로그인 후 돌아올 수 있도록 함
        const currentUrl = window.location.href;
        const finalUrl = `${loginUrl}?state=${encodeURIComponent(currentUrl)}`;

        console.log(`${provider} 로그인 URL:`, finalUrl);
        window.location.href = finalUrl;
    };

    return (
        <div className="social-login-container">
            <button
                onClick={() => handleSocialLogin('google')}
                className="google-login-btn"
            >
                <img src="/icons/google.svg" alt="Google" />
                Google로 로그인
            </button>

            <button
                onClick={() => handleSocialLogin('kakao')}
                className="kakao-login-btn"
            >
                <img src="/icons/kakao.svg" alt="Kakao" />
                카카오 로그인
            </button>

            <button
                onClick={() => handleSocialLogin('naver')}
                className="naver-login-btn"
            >
                <img src="/icons/naver.svg" alt="Naver" />
                네이버 로그인
            </button>
        </div>
    );
};

export default SocialLoginButtons;
```

### 3. Vite 개발서버 동적 프록시 설정

```javascript
// vite.config.js
import { defineConfig, loadEnv } from 'vite';
import react from '@vitejs/plugin-react';

export default ({ mode }) => {
    const env = loadEnv(mode, process.cwd(), "");
    const FRONTEND_PORT = Number(env.VITE_FRONTEND_PORT || "5005");
    const BACKEND_PORT = env.VITE_BACKEND_PORT || "9005";
    const USE_DYNAMIC_HOST = (env.VITE_USE_DYNAMIC_HOST || "false") === "true";
    const BACKEND_DOMAIN = (env.VITE_BACKEND_DOMAIN || "").trim();

    /**
     * 동적 프록시 라우터
     */
    const dynamicRouter = (req) => {
        // 1순위: 고정 도메인
        if (BACKEND_DOMAIN) {
            const protocol = BACKEND_DOMAIN.startsWith('http') ? '' : 'http://';
            return `${protocol}${BACKEND_DOMAIN}`;
        }

        // 2순위: 동적 호스트
        if (USE_DYNAMIC_HOST) {
            const hostHeader = req.headers.host;
            const hostOnly = (hostHeader || 'localhost').split(':')[0] || 'localhost';
            return `http://${hostOnly}:${BACKEND_PORT}`;
        }

        // 3순위: localhost 폴백
        return `http://localhost:${BACKEND_PORT}`;
    };

    return defineConfig({
        plugins: [react()],
        server: {
            host: true,  // 0.0.0.0 바인딩으로 외부 IPv4 접근 허용
            port: FRONTEND_PORT,
            proxy: {
                '/oauth2': {
                    target: `http://localhost:${BACKEND_PORT}`,
                    changeOrigin: true,
                    secure: false,
                    router: dynamicRouter,
                },
                '/login': {
                    target: `http://localhost:${BACKEND_PORT}`,
                    changeOrigin: true,
                    secure: false,
                    router: dynamicRouter,
                },
                '/api/v1': {
                    target: `http://localhost:${BACKEND_PORT}`,
                    changeOrigin: true,
                    secure: false,
                    router: dynamicRouter,
                }
            }
        }
    });
};
```

### 4. 로그인 상태 관리

```javascript
// useAuth.js
import { createContext, useContext, useEffect, useState } from 'react';
import { apiMethods } from '../api/api';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
    const [user, setUser] = useState(null);
    const [isLoading, setIsLoading] = useState(true);

    useEffect(() => {
        checkAuthStatus();
    }, []);

    const checkAuthStatus = async () => {
        try {
            const response = await apiMethods.get('/auth/status');
            if (response.data.success) {
                setUser(response.data.user);
            }
        } catch (error) {
            console.log('인증 상태 확인 실패:', error);
        } finally {
            setIsLoading(false);
        }
    };

    return (
        <AuthContext.Provider value={{ user, isLoading, checkAuthStatus }}>
            {children}
        </AuthContext.Provider>
    );
};

export const useAuth = () => {
    const context = useContext(AuthContext);
    if (!context) {
        throw new Error('useAuth는 AuthProvider 내에서 사용되어야 합니다');
    }
    return context;
};
```

---

## 에러 처리 및 예외 상황

### 1. OAuth2 실패 핸들러 (동적 리다이렉트 적용)

```java
// OAuth2AuthenticationFailureHandler.java
@Component
public class OAuth2AuthenticationFailureHandler extends SimpleUrlAuthenticationFailureHandler {

    @Value("${app.frontend.use-dynamic-host:true}")
    private boolean useDynamicHost;

    @Value("${app.frontend.domain:}")
    private String frontendDomain;

    @Value("${app.frontend.port:5005}")
    private String frontendPort;

    /**
     * 동적 프론트엔드 URL 생성 (성공 핸들러와 동일한 로직)
     */
    private String getFrontendBaseUrl(HttpServletRequest request) {
        if (!frontendDomain.isEmpty()) {
            return frontendDomain;
        }
        
        if (useDynamicHost) {
            String scheme = request.getScheme();
            String serverName = request.getServerName();
            return String.format("%s://%s:%s", scheme, serverName, frontendPort);
        }
        
        return String.format("http://localhost:%s", frontendPort);
    }

    @Override
    public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response,
                                       AuthenticationException exception) throws IOException {

        String errorMessage = "소셜 로그인에 실패했습니다";
        String errorCode = "OAUTH2_FAILED";

        // 예외 타입별 세부 처리
        if (exception instanceof OAuth2AuthenticationException) {
            OAuth2AuthenticationException oauth2Exception = (OAuth2AuthenticationException) exception;
            String errorCodeFromProvider = oauth2Exception.getError().getErrorCode();

            switch (errorCodeFromProvider) {
                case "access_denied":
                    errorMessage = "소셜 로그인 권한을 거부하셨습니다";
                    errorCode = "ACCESS_DENIED";
                    break;
                case "invalid_request":
                    errorMessage = "잘못된 요청입니다";
                    errorCode = "INVALID_REQUEST";
                    break;
                default:
                    errorMessage = "소셜 로그인 중 오류가 발생했습니다";
            }
        }

        // 동적 프론트엔드 URL로 에러 리다이렉트
        String frontendBaseUrl = getFrontendBaseUrl(request);
        String targetUrl = UriComponentsBuilder.fromUriString(frontendBaseUrl + "/login")
                .queryParam("error", errorCode)
                .queryParam("message", errorMessage)
                .build().toUriString();

        log.warn("OAuth2 인증 실패 - 에러: {}, 리다이렉트: {}", errorCode, targetUrl);
        getRedirectStrategy().sendRedirect(request, response, targetUrl);
    }
}
```

### 2. 프론트엔드 에러 처리

```javascript
// LoginPage.jsx
import { useSearchParams } from 'react-router-dom';

const LoginPage = () => {
    const [searchParams] = useSearchParams();
    const error = searchParams.get('error');
    const message = searchParams.get('message');

    const getErrorMessage = (errorCode) => {
        switch (errorCode) {
            case 'ACCESS_DENIED':
                return '소셜 로그인 권한을 허용해주세요';
            case 'INVALID_REQUEST':
                return '로그인 요청이 올바르지 않습니다';
            case 'OAUTH2_FAILED':
                return '소셜 로그인에 실패했습니다. 다시 시도해주세요';
            case 'NETWORK_ERROR':
                return '네트워크 연결을 확인해주세요';
            default:
                return '로그인 중 오류가 발생했습니다';
        }
    };

    return (
        <div className="login-page">
            {error && (
                <div className="error-banner">
                    {message || getErrorMessage(error)}
                </div>
            )}

            <SocialLoginButtons />
        </div>
    );
};
```

### 3. 공통 예외 상황 처리

```java
// GlobalExceptionHandler.java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OAuth2AuthenticationException.class)
    public ResponseEntity<?> handleOAuth2Exception(OAuth2AuthenticationException e) {
        log.warn("OAuth2 인증 실패: {}", e.getMessage());

        return ResponseEntity.badRequest().body(Map.of(
            "success", false,
            "message", "소셜 로그인에 실패했습니다",
            "errorCode", "OAUTH2_FAILED",
            "details", e.getError().getDescription()
        ));
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<?> handleIllegalArgument(IllegalArgumentException e) {
        if (e.getMessage().contains("소셜 로그인 제공자")) {
            return ResponseEntity.badRequest().body(Map.of(
                "success", false,
                "message", "지원하지 않는 소셜 로그인입니다",
                "errorCode", "UNSUPPORTED_PROVIDER"
            ));
        }

        return ResponseEntity.badRequest().body(Map.of(
            "success", false,
            "message", e.getMessage(),
            "errorCode", "INVALID_REQUEST"
        ));
    }
}
```

---

## 보안 고려사항

### 1. CSRF 공격 방지

```yaml
# application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            # state 파라미터 자동 생성으로 CSRF 방지
            # Spring Security가 자동으로 처리
```

### 2. 동적 Redirect URI 검증

```java
// OAuth2AuthenticationSuccessHandler.java
private String determineRedirectUrl(HttpServletRequest request, User user, boolean isNewUser) {
    String baseUrl = getFrontendBaseUrl(request);  // 동적 URL 생성

    // 화이트리스트 검증
    if (!isValidRedirectUrl(baseUrl)) {
        throw new SecurityException("유효하지 않은 리다이렉트 URL입니다");
    }

    if (isNewUser || !user.isOnboardingCompleted()) {
        return baseUrl + "/onboarding";
    } else {
        return baseUrl + "/dashboard";
    }
}

private boolean isValidRedirectUrl(String url) {
    // 환경변수에서 허용된 호스트 목록 가져오기
    String allowedHostsStr = System.getenv("ALLOWED_HOSTS");
    if (allowedHostsStr == null || allowedHostsStr.isEmpty()) {
        return url.startsWith("http://localhost");
    }

    String[] allowedHosts = allowedHostsStr.split(",");
    for (String host : allowedHosts) {
        String allowedUrl = String.format("http://%s:%s", host.trim(), frontendPort);
        if (url.startsWith(allowedUrl)) {
            return true;
        }
    }

    return false;
}
```

### 3. 토큰 스코프 최소화

```yaml
# Google - 필요한 최소 정보만 요청
scope: openid, profile, email

# Kakao - 필수 정보만 요청
scope: account_email, profile_nickname

# Naver - 기본 정보만 요청
scope: name, email
```

### 4. 민감 정보 로깅 방지

```java
// OAuth2AuthenticationSuccessHandler.java
private void logSocialLoginSuccess(User user, SocialAccount.SocialProvider provider) {
    // 민감 정보(이메일, 실명) 로깅 금지
    log.info("소셜 로그인 성공 - 사용자 ID: {}, 제공자: {}",
            user.getId(), provider.name());

    // 개발 환경에서만 상세 로깅
    if (isDebugMode()) {
        log.debug("소셜 로그인 상세 - 닉네임: {}", user.getNickname());
    }
}
```

---

## 트러블슈팅

### 1. IPv4 주소 접속 시 OAuth 실패

#### 증상
- localhost에서는 정상 동작하지만 IPv4 주소 접속 시 실패
- "redirect_uri_mismatch" 에러 발생
- 네트워크 연결 오류

#### 원인
1. **백엔드 네트워크 바인딩 문제**: `server.address` 설정 누락
2. **하드코딩된 OAuth URL**: 프론트엔드에서 고정 URL 사용
3. **소셜 제공자 콘솔 설정 누락**: IPv4 기반 Redirect URI 미등록
4. **CORS 정책 문제**: 허용되지 않은 Origin에서의 요청

#### 해결책

**1) 백엔드 네트워크 바인딩 수정:**
```yaml
# application.yml
server:
  port: ${SERVER_PORT:9005}
  address: 0.0.0.0  # 모든 네트워크 인터페이스 바인딩
```

**2) 동적 URL 시스템 적용:**
```javascript
// 프론트엔드에서 동적 OAuth URL 생성
const loginUrl = getOAuthURL(provider);  // IPv4 주소 기반 URL 생성
```

**3) 소셜 제공자 콘솔에 IPv4 URI 추가:**
```bash
# Google Cloud Console
- 승인된 리디렉션 URI에 추가:
  http://IPv4주소:9005/login/oauth2/code/google

# Kakao Developers
- Redirect URI에 추가:
  http://IPv4주소:9005/login/oauth2/code/kakao

# Naver Developers  
- Callback URL에 추가:
  http://IPv4주소:9005/login/oauth2/code/naver
```

**4) CORS 설정 확인:**
```bash
# .env 파일에 IPv4 주소 추가
ALLOWED_HOSTS=localhost,IPv4주소1,IPv4주소2
```

### 2. 자주 발생하는 문제들

#### Redirect URI 불일치
```bash
# 에러: redirect_uri_mismatch
# 해결: 각 소셜 제공자 콘솔에서 정확한 URI 등록
- Google: http://현재호스트:9005/login/oauth2/code/google
- Kakao: http://현재호스트:9005/login/oauth2/code/kakao
- Naver: http://현재호스트:9005/login/oauth2/code/naver
```

#### CORS 에러 (동적 프록시 적용)
```javascript
// vite.config.js에서 동적 프록시 설정 확인
const dynamicRouter = (req) => {
  const hostOnly = (req.headers.host || 'localhost').split(':')[0];
  return `http://${hostOnly}:${BACKEND_PORT}`;
};

// 프록시 설정에 dynamicRouter 적용
proxy: {
  '/oauth2': {
    target: `http://localhost:${BACKEND_PORT}`,
    changeOrigin: true,
    secure: false,
    router: dynamicRouter  // 동적 라우팅 적용
  }
}
```

#### 이메일 정보 누락 (Kakao)
```java
// 이메일 동의 여부 확인 후 처리
Map<String, Object> kakaoAccount = (Map<String, Object>) attributes.get("kakao_account");
String email = (String) kakaoAccount.get("email");

if (email == null || email.isEmpty()) {
    // 이메일 없이 회원가입 처리 또는 이메일 입력 요구
    log.warn("Kakao 이메일 정보 누락 - 사용자 ID: {}", attributes.get("id"));
    throw new OAuth2AuthenticationException("이메일 정보가 필요합니다");
}
```

### 3. 디버깅 가이드

#### OAuth2 로그 활성화
```yaml
# application.yml
logging:
  level:
    org.springframework.security.oauth2: DEBUG
    org.springframework.web.client.RestTemplate: DEBUG
    com.tropical.backend.config.oauth2: DEBUG  # 커스텀 OAuth2 핸들러 로그
```

#### 동적 URL 생성 디버깅
```java
// OAuth2AuthenticationSuccessHandler.java
private void debugUrlGeneration(HttpServletRequest request) {
    if (log.isDebugEnabled()) {
        log.debug("=== 동적 URL 생성 디버깅 ===");
        log.debug("요청 호스트: {}", request.getServerName());
        log.debug("요청 포트: {}", request.getServerPort());
        log.debug("요청 스키마: {}", request.getScheme());
        log.debug("생성된 프론트엔드 URL: {}", getFrontendBaseUrl(request));
        log.debug("환경변수 - USE_DYNAMIC_HOST: {}", useDynamicHost);
        log.debug("환경변수 - FRONTEND_DOMAIN: {}", frontendDomain);
        log.debug("환경변수 - FRONTEND_PORT: {}", frontendPort);
    }
}
```

#### 프론트엔드 OAuth URL 디버깅
```javascript
// SocialLoginButtons.jsx에서 URL 확인
const handleSocialLogin = (provider) => {
    const backendUrl = getBackendBaseUrl();
    const oauthUrl = getOAuthURL(provider);
    
    console.group(`${provider} 소셜 로그인 디버깅`);
    console.log('현재 호스트:', window.location.hostname);
    console.log('현재 포트:', window.location.port);
    console.log('백엔드 URL:', backendUrl);
    console.log('OAuth URL:', oauthUrl);
    console.log('환경변수 - VITE_USE_DYNAMIC_HOST:', import.meta.env.VITE_USE_DYNAMIC_HOST);
    console.log('환경변수 - VITE_BACKEND_DOMAIN:', import.meta.env.VITE_BACKEND_DOMAIN);
    console.groupEnd();
    
    window.location.href = oauthUrl;
};
```

### 4. 성능 최적화

#### OAuth2 연결 풀 설정
```java
@Configuration
public class OAuth2HttpClientConfig {

    @Bean
    public RestTemplate oAuth2RestTemplate() {
        HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory();

        // OAuth Provider와의 연결 최적화
        factory.setConnectTimeout(3000);
        factory.setReadTimeout(10000);
        factory.setConnectionRequestTimeout(1000);

        return new RestTemplate(factory);
    }
}
```

### 5. 모니터링 및 통계

#### 소셜 로그인 성공률 모니터링
```java
@Component
public class SocialLoginMetrics {

    private final MeterRegistry meterRegistry;

    @EventListener
    public void handleSocialLoginSuccess(SocialLoginSuccessEvent event) {
        // 제공자별 성공 카운트
        Counter.builder("social.login.success")
            .tag("provider", event.getProvider().name())
            .tag("new_user", String.valueOf(event.isNewUser()))
            .tag("host_type", event.isLocalhost() ? "localhost" : "ipv4")
            .register(meterRegistry)
            .increment();
    }

    @EventListener
    public void handleSocialLoginFailure(SocialLoginFailureEvent event) {
        // 제공자별 실패 카운트
        Counter.builder("social.login.failure")
            .tag("provider", event.getProvider())
            .tag("error_code", event.getErrorCode())
            .tag("host_type", event.isLocalhost() ? "localhost" : "ipv4")
            .register(meterRegistry)
            .increment();
    }
}
```

### 6. 향후 개선 계획

#### 추가 소셜 제공자 지원 (GitHub 예시)
```java
// GitHub OAuth2 설정 예시
@Configuration
public class GitHubOAuth2Config {

    private ClientRegistration githubClientRegistration() {
        return ClientRegistration.withRegistrationId("github")
            .clientId("${GITHUB_CLIENT_ID}")
            .clientSecret("${GITHUB_CLIENT_SECRET}")
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .redirectUri("{baseUrl}/login/oauth2/code/{registrationId}")
            .scope("user:email")
            .authorizationUri("https://github.com/login/oauth/authorize")
            .tokenUri("https://github.com/login/oauth/access_token")
            .userInfoUri("https://api.github.com/user")
            .userNameAttributeName("id")
            .clientName("GitHub")
            .build();
    }
}
```

#### 소셜 계정 연동 해제
```java
@RestController
@RequestMapping("/api/social")
public class SocialAccountController {

    @DeleteMapping("/unlink/{provider}")
    public ResponseEntity<?> unlinkSocialAccount(@PathVariable String provider,
                                               HttpServletRequest request) {
        Long userId = getCurrentUserId();
        SocialAccount.SocialProvider socialProvider =
            SocialAccount.SocialProvider.fromRegistrationId(provider);

        boolean success = socialAccountService.unlinkSocialAccount(userId, socialProvider);

        if (success) {
            // 동적 프론트엔드 URL로 성공 응답
            String frontendUrl = getFrontendBaseUrl(request);
            return ResponseEntity.ok(Map.of(
                "success", true,
                "message", "소셜 계정 연동이 해제되었습니다",
                "redirectUrl", frontendUrl + "/settings"
            ));
        } else {
            return ResponseEntity.badRequest().body(Map.of(
                "success", false,
                "message", "연동 해제에 실패했습니다"
            ));
        }
    }
}
```

---

## 결론

TropiCal의 소셜 로그인 시스템은 **환경변수 기반 동적 URL 생성**을 통해 개발 환경의 유연성과 운영 환경의 안정성을 동시에 확보했습니다.

### 주요 성과

1. **환경 적응성**: localhost와 IPv4 주소 접속을 모두 지원하는 동적 시스템
2. **개발 효율성**: 새로운 개발 환경 추가 시 환경변수만 수정하면 되는 구조
3. **보안 강화**: 화이트리스트 기반 리다이렉트 URL 검증
4. **확장성**: 새로운 소셜 제공자 추가 용이한 아키텍처

### 핵심 기술 요소

- **Spring Security OAuth2**: 표준 기반 인증 처리
- **동적 URL 생성**: 환경변수 기반 우선순위 시스템
- **Vite 동적 프록시**: 개발서버에서의 자동 라우팅
- **통합 토큰 관리**: ONBOARDING_TOKEN과 ACCESS_TOKEN의 생명주기 관리

이 문서는 소셜 로그인 시스템의 구현부터 트러블슈팅까지 전 과정을 다루며, 특히 다중 개발 환경에서의 동적 URL 처리 방식은 향후 다른 프로젝트에서도 재사용 가능한 패턴을 제시합니다.