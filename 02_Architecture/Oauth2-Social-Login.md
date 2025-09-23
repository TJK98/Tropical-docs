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
2. [OAuth2 플로우 이해](#oauth2-플로우-이해)
3. [Google 로그인 구현](#google-로그인-구현)
4. [Kakao 로그인 구현](#kakao-로그인-구현)
5. [Naver 로그인 구현](#naver-로그인-구현)
6. [통합 인증 처리](#통합-인증-처리)
7. [프론트엔드 연동](#프론트엔드-연동)
8. [에러 처리 및 예외 상황](#에러-처리-및-예외-상황)
9. [보안 고려사항](#보안-고려사항)
10. [트러블슈팅](#트러블슈팅)

---

## 개요

TropiCal 프로젝트는 Spring Security OAuth2를 기반으로 Google, Kakao, Naver의 소셜 로그인을 통합 구현했습니다. 각 제공자별 특성을 고려한 맞춤형 설정과 일관된 사용자 경험을 제공합니다.

### 지원하는 소셜 제공자

- **Google**: OpenID Connect 표준 준수
- **Kakao**: REST API 기반 커스텀 구현
- **Naver**: REST API 기반 커스텀 구현

### 핵심 설계 원칙

- **표준 준수**: OAuth2/OIDC 표준을 최대한 활용
- **확장성**: 새로운 소셜 제공자 추가 용이
- **보안성**: PKCE, State 파라미터 등 보안 강화
- **사용자 경험**: 실패 시 명확한 안내 및 대안 제공

---

## OAuth2 플로우 이해

## OAuth2 플로우 이해

### 1. Authorization Code Grant 플로우

```
1. [사용자] → [프론트엔드]: 소셜 로그인 버튼 클릭
2. [프론트엔드] → [백엔드]: GET /oauth2/authorization/{provider}
3. [백엔드] → [소셜 제공자]: 인증 요청 (client_id, redirect_uri, scope)
4. [소셜 제공자] → [사용자]: 로그인 화면 표시
5. [사용자] → [소셜 제공자]: 로그인 및 권한 승인
6. [소셜 제공자] → [백엔드]: 인가 코드 전달 (GET /login/oauth2/code/{provider})
7. [백엔드] → [소셜 제공자]: Access Token 요청 (인가 코드 교환)
8. [소셜 제공자] → [백엔드]: Access Token 응답
9. [백엔드] → [소셜 제공자]: 사용자 정보 요청 (Access Token 사용)
10. [소셜 제공자] → [백엔드]: 사용자 정보 응답
11. [백엔드]: 사용자 생성/조회 및 JWT 토큰 발급
12. [백엔드] → [프론트엔드]: 리다이렉트 (JWT 쿠키 설정)
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

### 1. Google Cloud Console 설정

```bash
# Google Cloud Console (console.cloud.google.com)에서 설정
1. 프로젝트 생성 또는 선택
2. API 및 서비스 → 사용자 인증 정보
3. OAuth 2.0 클라이언트 ID 생성
   - 애플리케이션 유형: 웹 애플리케이션
   - 승인된 자바스크립트 원본: http://localhost:5005
   - 승인된 리디렉션 URI: http://localhost:9005/login/oauth2/code/google
```

### 2. application.yml 설정

```yaml
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
```

### 3. Google 사용자 정보 처리

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

### 4. Google 특징 및 고려사항

**장점:**

- OpenID Connect 표준 완전 준수
- Spring Security OAuth2와 완벽 호환
- 안정적인 API 제공
- 상세한 개발자 문서

**주의사항:**

```java
// Google은 'sub' 클레임을 사용자 고유 식별자로 사용
// 'email'은 변경 가능하므로 'sub'를 기본키로 사용해야 함
String googleUserId = (String) attributes.get("sub");  // 안정적
String email = (String) attributes.get("email");       // 변경 가능
```

---

## Kakao 로그인 구현

### 1. Kakao Developers 설정

```bash
# Kakao Developers (developers.kakao.com)에서 설정
1. 애플리케이션 추가
2. 플랫폼 설정 → Web 추가
   - 사이트 도메인: http://localhost:5005
3. 카카오 로그인 → 활성화 설정
   - Redirect URI: http://localhost:9005/login/oauth2/code/kakao
4. 동의항목 → 개인정보 항목 설정
   - 닉네임, 프로필 사진, 이메일 (선택 동의)
```

### 2. application.yml 설정

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          kakao:
            client-id: ${KAKAO_CLIENT_ID}
            client-secret: ${KAKAO_CLIENT_SECRET}
            client-authentication-method: client_secret_post
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            scope: account_email, profile_nickname

        provider:
          kakao:
            authorization-uri: https://kauth.kakao.com/oauth/authorize
            token-uri: https://kauth.kakao.com/oauth/token
            user-info-uri: https://kapi.kakao.com/v2/user/me
            user-name-attribute: id
```

### 3. Kakao 사용자 정보 처리

```java
// OAuth2AuthenticationSuccessHandler.java
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

### 4. Kakao API 응답 구조

```json
{
  "id": 1234567890,
  "connected_at": "2022-04-11T01:45:28Z",
  "kakao_account": {
    "email_needs_agreement": false,
    "is_email_valid": true,
    "is_email_verified": true,
    "email": "user@example.com",
    "profile_needs_agreement": false,
    "profile": {
      "nickname": "홍길동",
      "thumbnail_image_url": "http://yyy.kakao.com/.../img_110x110.jpg",
      "profile_image_url": "http://yyy.kakao.com/dn/.../img_640x640.jpg"
    }
  }
}
```

### 5. Kakao 특징 및 고려사항

**특징:**

- 숫자형 사용자 ID 제공
- 이메일 동의는 선택사항 (비즈니스 앱 인증 필요)
- 풍부한 프로필 정보 제공

**주의사항:**

```java
// 이메일 동의 여부 확인 필요
Map<String, Object> kakaoAccount = (Map<String, Object>) attributes.get("kakao_account");
Boolean emailAgreed = (Boolean) kakaoAccount.get("email_needs_agreement");

if (Boolean.TRUE.equals(emailAgreed)) {
    // 이메일 정보 없음 - 대체 로직 필요
    throw new OAuth2AuthenticationException("이메일 동의가 필요합니다");
}
```

---

## Naver 로그인 구현

### 1. Naver Developers 설정

```bash
# Naver Developers (developers.naver.com)에서 설정
1. 애플리케이션 등록
2. API 설정
   - 서비스 URL: http://localhost:5005
   - Callback URL: http://localhost:9005/login/oauth2/code/naver
3. 제공 정보 선택
   - 이메일 주소, 이름, 연락처 전화번호 등
```

### 2. application.yml 설정

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          naver:
            client-id: ${NAVER_CLIENT_ID}
            client-secret: ${NAVER_CLIENT_SECRET}
            client-authentication-method: client_secret_post
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            scope: name, email

        provider:
          naver:
            authorization-uri: https://nid.naver.com/oauth2.0/authorize
            token-uri: https://nid.naver.com/oauth2.0/token
            user-info-uri: https://openapi.naver.com/v1/nid/me
            user-name-attribute: response
```

### 3. Naver 사용자 정보 처리

```java
// OAuth2AuthenticationSuccessHandler.java
case NAVER -> {
    Map<String, Object> naverResponse = (Map<String, Object>) attributes.get("response");

    yield new SocialUserInfo(
        (String) naverResponse.get("id"),        // 네이버 사용자 ID
        (String) naverResponse.get("email"),     // 이메일
        (String) naverResponse.get("name")       // 이름
    );
}
```

### 4. Naver API 응답 구조

```json
{
  "resultcode": "00",
  "message": "success",
  "response": {
    "id": "32742776",
    "nickname": "홍길동",
    "name": "홍길동",
    "email": "user@example.com",
    "gender": "M",
    "age": "30-39",
    "birthday": "10-01",
    "profile_image": "https://ssl.pstatic.net/static/pwe/address/img_profile.png",
    "birthyear": "1990",
    "mobile": "010-1234-5678"
  }
}
```

### 5. Naver 특징 및 고려사항

**특징:**

- 문자열 형태의 사용자 ID
- 응답이 'response' 객체로 래핑됨
- 상세한 개인정보 제공 (생년월일, 연령대 등)

**주의사항:**

```java
// user-name-attribute가 'response'로 설정되어 있어
// 실제 사용자 정보는 response 객체 내부에 있음
Map<String, Object> response = (Map<String, Object>) attributes.get("response");
String userId = (String) response.get("id");
```

---

## 통합 인증 처리

### 1. 소셜 계정 매핑 전략

```java
// SocialAccount.java
public enum SocialProvider {
    KAKAO("kakao"),
    GOOGLE("google"),
    NAVER("naver");

    private final String registrationId;

    public static SocialProvider fromRegistrationId(String registrationId) {
        for (SocialProvider provider : values()) {
            if (provider.registrationId.equals(registrationId)) {
                return provider;
            }
        }
        throw new IllegalArgumentException("지원하지 않는 소셜 로그인 제공자입니다: " + registrationId);
    }
}
```

### 2. 통합 사용자 생성 로직

```java
// OAuth2AuthenticationSuccessHandler.java
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

        // 리다이렉트 URL 결정 및 토큰 발급
        String redirectUrl = determineRedirectUrl(user, isNewUser);
        issueTokens(user, response);

        getRedirectStrategy().sendRedirect(request, response, redirectUrl);

    } catch (Exception e) {
        handleAuthenticationFailure(request, response, registrationId, e);
    }
}
```

### 3. 토큰 발급 전략

```java
// OAuth2AuthenticationSuccessHandler.java
private void issueTokens(User user, HttpServletResponse response) {
    if (!user.isOnboardingCompleted()) {
        // 온보딩용 임시 토큰 발급 (30분)
        String tempToken = jwtTokenProvider.createOnboardingToken(user.getId(), user.getEmail());
        response.addCookie(CookieUtil.build("ACCESS_TOKEN", tempToken, 1800));
    } else {
        // 정식 JWT 토큰 발급
        String accessToken = jwtTokenProvider.createAccessToken(user.getId(), user.getEmail());
        String refreshToken = jwtTokenProvider.createRefreshToken(user.getId());

        int accessMaxAge = jwtTokenProvider.getAccessMaxAge();
        int refreshMaxAge = jwtTokenProvider.getRefreshMaxAge();

        response.addCookie(CookieUtil.build("ACCESS_TOKEN", accessToken, accessMaxAge));
        response.addCookie(CookieUtil.build("REFRESH_TOKEN", refreshToken, refreshMaxAge));
    }
}
```

---

## 프론트엔드 연동

### 1. 환경변수 설정

```javascript
// .env
VITE_GOOGLE_LOGIN_URL=http://localhost:9005/oauth2/authorization/google
VITE_KAKAO_LOGIN_URL=http://localhost:9005/oauth2/authorization/kakao
VITE_NAVER_LOGIN_URL=http://localhost:9005/oauth2/authorization/naver
```

### 2. OAuth URL 빌더

```javascript
// api.js
export function getOAuthURL(provider) {
    const envKey = `VITE_${provider.toUpperCase()}_LOGIN_URL`;
    return import.meta.env[envKey] || `${API_BASE_URL}/oauth2/authorization/${provider}`;
}

export const OAUTH_URLS = {
    google: getOAuthURL('google'),
    kakao: getOAuthURL('kakao'),
    naver: getOAuthURL('naver'),
};
```

### 3. 소셜 로그인 컴포넌트

```javascript
// SocialLoginButtons.jsx
import React from 'react';
import { OAUTH_URLS } from '../api/api';

const SocialLoginButtons = () => {
    const handleSocialLogin = (provider) => {
        // 현재 페이지 URL을 state로 전달하여 로그인 후 돌아올 수 있도록 함
        const currentUrl = window.location.href;
        const loginUrl = `${OAUTH_URLS[provider]}?state=${encodeURIComponent(currentUrl)}`;

        window.location.href = loginUrl;
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

### 1. OAuth2 실패 핸들러

```java
// OAuth2AuthenticationFailureHandler.java
@Component
public class OAuth2AuthenticationFailureHandler extends SimpleUrlAuthenticationFailureHandler {

    @Value("${app.frontend.base-url}")
    private String frontendBaseUrl;

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

        String targetUrl = UriComponentsBuilder.fromUriString(frontendBaseUrl + "/login")
                .queryParam("error", errorCode)
                .queryParam("message", errorMessage)
                .build().toUriString();

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

### 2. Redirect URI 검증

```java
// OAuth2AuthenticationSuccessHandler.java
private String determineRedirectUrl(User user, boolean isNewUser) {
    String baseUrl = frontendBaseUrl;

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
    List<String> allowedDomains = Arrays.asList(
        "http://localhost:5005",
        "https://tropical.example.com"
    );

    return allowedDomains.stream().anyMatch(url::startsWith);
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

### 1. 자주 발생하는 문제들

#### Redirect URI 불일치

```bash
# 에러: redirect_uri_mismatch
# 해결: 각 소셜 제공자 콘솔에서 정확한 URI 등록
- Google: http://localhost:9005/login/oauth2/code/google
- Kakao: http://localhost:9005/login/oauth2/code/kakao
- Naver: http://localhost:9005/login/oauth2/code/naver
```

#### CORS 에러

```javascript
// 개발환경 vite.config.js
export default defineConfig({
  server: {
    proxy: {
      '/oauth2': {
        target: 'http://localhost:9005',
        changeOrigin: true,
        secure: false
      },
      '/login': {
        target: 'http://localhost:9005',
        changeOrigin: true,
        secure: false
      }
    }
  }
});
```

#### 이메일 정보 누락 (Kakao)

```java
// 이메일 동의 여부 확인 후 처리
Map<String, Object> kakaoAccount = (Map<String, Object>) attributes.get("kakao_account");
String email = (String) kakaoAccount.get("email");

if (email == null || email.isEmpty()) {
    // 이메일 없이 회원가입 처리 또는 이메일 입력 요구
    throw new OAuth2AuthenticationException("이메일 정보가 필요합니다");
}
```

### 2. 디버깅 가이드

#### OAuth2 로그 활성화

```yaml
# application.yml
logging:
  level:
    org.springframework.security.oauth2: DEBUG
    org.springframework.web.client.RestTemplate: DEBUG
```

#### 사용자 정보 확인

```java
// OAuth2AuthenticationSuccessHandler.java
private void debugUserAttributes(OAuth2User oauth2User, String provider) {
    if (log.isDebugEnabled()) {
        log.debug("=== {} 사용자 정보 ===", provider);
        oauth2User.getAttributes().forEach((key, value) ->
            log.debug("{}: {}", key, value)
        );
        log.debug("=== 권한 정보 ===");
        oauth2User.getAuthorities().forEach(authority ->
            log.debug("Authority: {}", authority.getAuthority())
        );
    }
}
```

### 3. 성능 최적화

#### RestTemplate 연결 풀 설정

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

        // Connection Pool 설정
        PoolingHttpClientConnectionManager connectionManager =
            new PoolingHttpClientConnectionManager();
        connectionManager.setMaxTotal(100);
        connectionManager.setDefaultMaxPerRoute(20);

        CloseableHttpClient httpClient = HttpClientBuilder.create()
            .setConnectionManager(connectionManager)
            .setRetryHandler(new DefaultHttpRequestRetryHandler(3, true))
            .build();

        factory.setHttpClient(httpClient);

        return new RestTemplate(factory);
    }
}
```

#### OAuth2 응답 캐싱

```java
@Service
public class CachedOAuth2UserService {

    private final RedisTemplate<String, Object> redisTemplate;
    private static final String USER_CACHE_PREFIX = "oauth2:user:";
    private static final Duration CACHE_TTL = Duration.ofMinutes(10);

    public OAuth2User loadUser(OAuth2UserRequest userRequest) {
        String cacheKey = USER_CACHE_PREFIX +
                         userRequest.getClientRegistration().getRegistrationId() + ":" +
                         userRequest.getAccessToken().getTokenValue().hashCode();

        // 캐시에서 사용자 정보 조회
        OAuth2User cachedUser = (OAuth2User) redisTemplate.opsForValue().get(cacheKey);
        if (cachedUser != null) {
            return cachedUser;
        }

        // 실제 API 호출
        OAuth2User user = super.loadUser(userRequest);

        // 캐시에 저장 (10분 TTL)
        redisTemplate.opsForValue().set(cacheKey, user, CACHE_TTL);

        return user;
    }
}
```

### 4. 통계 및 모니터링

#### 소셜 로그인 통계 수집

```java
@Component
public class SocialLoginMetrics {

    private final MeterRegistry meterRegistry;
    private final Counter googleLoginCounter;
    private final Counter kakaoLoginCounter;
    private final Counter naverLoginCounter;

    @EventListener
    public void handleSocialLoginSuccess(SocialLoginSuccessEvent event) {
        Counter counter = switch (event.getProvider()) {
            case GOOGLE -> googleLoginCounter;
            case KAKAO -> kakaoLoginCounter;
            case NAVER -> naverLoginCounter;
        };

        counter.increment(Tags.of(
            "provider", event.getProvider().name(),
            "new_user", String.valueOf(event.isNewUser())
        ));

        // 소셜 로그인 성공률 계산
        Gauge.builder("social.login.success_rate")
            .tag("provider", event.getProvider().name())
            .register(meterRegistry, this, this::calculateSuccessRate);
    }
}
```

### 5. 향후 개선 계획

#### 추가 소셜 제공자 지원

```java
// GitHub 로그인 추가 예시
@Configuration
public class GitHubOAuth2Config {

    @Bean
    public ClientRegistrationRepository clientRegistrationRepository() {
        return new InMemoryClientRegistrationRepository(
            googleClientRegistration(),
            kakaoClientRegistration(),
            naverClientRegistration(),
            githubClientRegistration()  // 새로운 제공자 추가
        );
    }

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
    public ResponseEntity<?> unlinkSocialAccount(@PathVariable String provider) {
        Long userId = getCurrentUserId();
        SocialAccount.SocialProvider socialProvider =
            SocialAccount.SocialProvider.fromRegistrationId(provider);

        boolean success = socialAccountService.unlinkSocialAccount(userId, socialProvider);

        if (success) {
            return ResponseEntity.ok("소셜 계정 연동이 해제되었습니다");
        } else {
            return ResponseEntity.badRequest().body("연동 해제에 실패했습니다");
        }
    }

    @GetMapping("/linked")
    public ResponseEntity<?> getLinkedAccounts() {
        Long userId = getCurrentUserId();
        Map<SocialAccount.SocialProvider, Boolean> linkedAccounts =
            socialAccountService.getUserSocialProviderStatus(userId);

        return ResponseEntity.ok(linkedAccounts);
    }
}
```
