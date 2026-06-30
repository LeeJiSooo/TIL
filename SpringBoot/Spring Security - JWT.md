## 가져가야 할 핵심

- 서버가 로그인 상태를 어디에 두느냐? 세션 방식은 서버가 기억하고 JWT는 매 요청마다 토큰으로 다시 증명
- JWT는 암호화된 비밀 상자 같은 게 아니고 Header, Payload, Signature로 이루어져 있고, Payload는 누구나 읽을 수 있다. → 민감 정보 넣으면 안됨
- 이 방식이 실제로 동작하는 중심에는 JwtFilter가 있다. 매 요청마다 헤더에 베어러 토큰 검증하고 유효하면 Authentication 객체 만들어서 시큐리티 컨텍스트 홀더에 넣어준다.
- JWT 검증 실패 같은 예외는 컨트롤러에 닿기 전에 필터 단계에서 터진다. @RestControllerAdvice에서 못 잡는다. 별도의 핸들러 처리한다.

## 세션 기반 인증 vs 토큰 기반 인증(JWT)

서버가 로그인 상태를 ‘어디에 기억하느냐’에 따라 아키텍처 확장성이 갈린다.

- 세션 기반 인증(Stateful) : 서버가 유저의 로그인 상태를 자신의 메모리나 DB에 직접 저장하고 유지한다.
- 토큰 기반 인증(Stateless) : 서버는 로그인 상태를 기억하지 않는다. 대신 유저가 매 요청마다 토큰을 들고 와서 자신이 인증된 사용자임을 매번 새롭게 증명한다.

#### 수평확장시 세션의 한계와 토큰의 이점

- 세션의 한계 : 동시 접속자가 수십만 명으로 늘어나면 서버 메모리에 부담이 커진다. 가장 큰 문제는 서버를 여러 대로 늘릴 때(수평 확장)발생한다.
    
    유저가 1번 서버에서 로그인했는데 다음 요청이 2번 서버로 가면, 2번 서버는 이 유저를 모르기 때문에 로그인이 풀려버린다. → 서버 간에 세션을 동기화하는 세션 클러스터링이나 별도 저장소(레디스 등)가 필요해서 인프라가 복잡해진다.
    
- JWT의 이점 : 서버는 상태를 전혀 저장하지 않는다. 토큰 자체에 “나는 user1이고, ADMIN 권한이 있으며, 1시간 뒤에 만료된다”라는 정보가 다 들어있다.
    
    서버는 이 토큰이 진짜 자기가 발급한 게 맞는지 서명만 검증하면 된다.
    
    1번 서버든 100번 서버든 비밀키만 공유하고 있다면 어느 서버로 요청이 가도 완벽하게 인증을 처리할 수 있어서 수평 확장에 유리하다.
    

#### 트레이드 오프

- 세션 : 서버가 상태의 절대적인 결정을 쥐고 있으므로, 특정 유저를 강제 로그아웃시키거나 중복 로그인을 막는 등의 제어가 매우 쉽다.
- JWT : 한 번 발급되면 만료될 때가지 서버가 강제로 무효화하기 어렵다. 토큰을 도난당해도 막기가 까다롭기 때문에, 만료 시간이 짧은 엑세스토큰과 이를 재발급하는 리프레시 토큰을 조합하거나 별도의 블랙리스트를 구축해서 보완한다.

## JWT(JSON Web Token)의 구조

- Header(헤더) : 토큰의 타입과 서명에 사용한 알고리즘 정보가 담긴다.
- Payload(페이로드) : 실제 데이터가 들어간다. 유저 고유 아이디, 토큰 유효기간, 권한 목록 등이 포함된다.
- Signature(시그니처/서명) : 헤더와 페이로드를 합친 뒤, 서버만 알고 있는 비밀키로 해시 서명한 값이다.

**JWT가 보장하는 것은 ‘기밀성’이 아니라 ‘무결성’이다!!**

- 페이로드는 암호화된 게 아니라 단순히 Base64로 인코딩되어 있을 뿐이다. 누구나 디코딩 사이트에 넣으면 내부 내용을 그대로 읽을 수 있다.
    
     → 패스워드나 주민등록번호 같은 민감한 개인정보를 넣으면 안된다.
    
- 시그니처 존재 이유 : 누구나 내용을 볼 수는 있지만, 중간에 누군가 페이로드의 데이터를 살짝 수정하는 순간 서버의 비밀키로 계산한 시그니처 값과 달라지게 된다.
    
    → JWT는 “이 토큰 안의 내용은 내가 발급한 이후로 단 한 글자도 바뀌지 않았다”는 무결성을 보장하는 도구이다.
    

## JWT인증 아키텍처와 동작 흐름

!image.png

스프링 시큐리티에서 JWT방식이 돌아갈 수 있도록 만들어주는 것은 우리가 커스텀하게 구현할 JwtFilter이다.

#### 전체 동작 흐름

1. 요청 전송 : 클라이언트가 HTTP 요청 헤더에 토큰을 실어 보낸다.(`Authorization: Bearer <JWT_토큰>`)
2. 필터 가로채기 : JwtFilter가 이 요청을 가로채서 헤더의 토큰 문자열을 뽑아낸다.
3. 유효성 검증 : 검증 컴포넌트인 TokenProvider에게 토큰을 넘겨 서명이 올바른지, 만료되지는 않았는지 검사한다.
4. 인증 처리(성공 시) : 토큰이 유효하다면 페이로드에서 유저 ID와 권한을 꺼내 `Authentication`객체를 생성한다. 그리고 이 객체를 `SecurityContextHolder`에 저장한다. → 컨트롤러 영역으로 넘어갔을 때 로그인된 유저로 인식됨.
5. 익명 처리(토큰이 없거나 무효할 시) : 토큰이 없거나 가짜라면 아무것도 저장하지 않고 다음 필터로 넘긴다. 이 경우 시큐리티는 이 요청을 익명 사용자로 취급하며, 만약 권한이 필요한 자원이었다면 후반부 인가 필터 단계에서 막힌다.
6. Context삭제(Stateless) : 컨트롤러에서 비즈니스 로직 처리가 끝나고 응답이 나가는 순간, `SecurityContextHolder` 에 보관했던 인증 정보는 완전히 파기된다. 서버는 아무 성태도 저장하지 않는 원칙을 지킨다.

**SecurityConfig.java (시큐리티 옵션 조정)**

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtFilter jwtFilter;
    private final CorsFilter corsFilter; // 별도로 설정한 CORS 필터

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // 1. JWT 환경이므로 CSRF 보호 비활성화 (토큰을 헤더로 수동 전달하니까 안전함)
            .csrf(csrf -> csrf.disable())

            // 2. 전통적인 폼 로그인 및 기본 HTTP 인증 비활성화
            .formLogin(form -> form.disable())
            .httpBasic(basic -> basic.disable())

            // 3. [가장 중요] 세션을 절대 만들지도 말고, 쓰지도 말라고 선언 (STATELESS)
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )

            // 4. URL별 인가 규칙 설정
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**", "/error").permitAll() // 인증 없이 접근 허용
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )

            // 5. 필터 배치 순서 결정
            // 오리지널 로그인 필터(UsernamePasswordAuthenticationFilter)가 돌기 전에 우리 JwtFilter를 먼저 실행한다!
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .addFilterBefore(corsFilter, JwtFilter.class); // CORS 검사는 보안 검사보다 훨씬 앞단에서 처리되는 게 권장됨

        return http.build();
    }
}
```

**JwtFilter.java (매 요청을 가로채 검증하는 필터)**

```java
@Component
@RequiredArgsConstructor
public class JwtFilter extends OncePerRequestFilter { // 한 요청당 딱 한 번만 실행되는 것을 보장하는 필터 상속

    private final TokenProvider tokenProvider; // JWT 생성 및 검증을 전담하는 커스텀 컴포넌트

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {

        // 1. HTTP Request Header에서 'Bearer '를 떼어내고 순수 토큰만 추출
        String token = resolveToken(request);

        // 2. 토큰이 존재하고, 서명 및 만료일이 완벽하게 유효하다면
        if (token != null && tokenProvider.validateToken(token)) {
            // 3. 토큰에서 유저 정보와 권한을 뜯어내 Authentication 객체로 만든 뒤 context에 안착시킴
            Authentication authentication = tokenProvider.getAuthentication(token);
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        // 4. 내 할 일 다 했으니 체인의 다음 필터로 요청을 넘겨줌
        filterChain.doFilter(request, response);
    }

    // 헤더 파싱 도와주는 메서드
    private String resolveToken(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7); // "Bearer " 뒷부분의 실제 토큰 문자열만 잘라냄
        }
        return null;
    }
}
```

## 필터 단계의 인증/인가 예외 처리

**@RestControllerAdvice가 시큐리티 에러를 잡지 못하는 이유**

일반적으로 우리는 스프링부트에서 예외 처리를 할 때 `@RestControllerAdvice`나 `@ExceptionHandler` 를 팩토리처럼 사용한다. 하지만 요청이 `DispatcherServlet`을 지나 컨트롤러 영역 안으로 진입한 이후에 터진 에러만 수집할 수 있다.

JWT검증 실패나 권한 부족은 컨트롤러에 닿기도 전인 가장 앞단의 서블릿 필터 체인 단계에서 터지기 때문에, 전역 예외 처리가 안됨.

**해결책 : ExceptionTranslationFilter의 두 가지 핸들러 활용**

스프링 시큐리티는 필터 계층에서 에러가 터졌을 때 안전하게 커스텀 JSON응답을 내려줄 수 있도록 두가지 전용 장치를 설정 클래스에 등록할 수 있게 해준다.

```java
// SecurityConfig 내부에 예외 핸들링 지정 방법
http.exceptionHandling(exception -> exception
    .authenticationEntryPoint(new CustomAuthenticationEntryPoint()) // 인증 실패(401) 처리 전담
    .accessDeniedHandler(new CustomAccessDeniedHandler())           // 인가 실패(403) 처리 전담
);
```

**AuthenticationEntryPoint (401 Unauthorized 전담)**

```java
@Component
public class CustomAuthenticationEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
                         AuthenticationException authException) throws IOException {
        // 컨트롤러 advice를 못 타니까, 서블릿 response 객체에 직접 JSON 포맷과 401 상태코드를 구워 구체적인 에러 메시지를 응답함
        response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "인증에 실패했습니다. 유효한 토큰이 없습니다.");
    }
}
```

**AccessDeniedHandler (403 Forbidden 전담)**

인증은 제대로 돼서 유저가 누구인지는 알겠으나, 일반 유저가 /api/admin/같은 관리자 전용 경로에 접근하여 권한 부족(인가 실패)로 튕겼을 때 실행되어 403 응답을 내려준다.

## 정리

- 세션은 서버가 유저 상태를 기억하는 방식이고, JWT는 서버가 무상태(Stateless)를 유지하며 유저가 매번 토큰으로 스스로를 증명하는 방식이다.
- JWT는 Header, Payload, Signiture로 구성되며, 페이로드는 누구나 읽을 수 있으므로 민감 정보를 넣으면 안된다. 시그니처는 위변조를 막는 무결성을 보장한다.
- `SessionCreationPolicy.STATELESS`로 세션을 완전히 끄고, `UsernamePasswordAuthenticationFilter` 앞단에 커스텀 `JwtFilter`를 끼워 넣어 매 요청마다 토큰을 검증하고 `SecurityContext`에 인증 객체를 넣는다.
- 필터 단계에서 터지는 JWT 에러는 `@RestControllerAdvice`로 잡을 수 없다. 따라서 `AuthenticationEntryPoint(401)`와 `AccessDeniedHandler(403)`라는 전용 핸들러를 시큐리티에 직접 등록해 처리해야 한다.
