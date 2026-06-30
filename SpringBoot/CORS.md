## 가져가야 할 핵심

- CORS 막는 기술이 아니라 여는 기술이다. 브라우저가 기본적으로 막아둔 “다른 출처로의 요청”을 서버의 허락을 받아서 안전하게 열어주는 약속이다.
- 결정하는 쪽과 집행하는 쪽이 다르다. “이 출처를 허용한다”고 정하는건 서버인데 그 응답을 보고 실제로 통과시킬지 막을지를 판단하는 건 브라우저이다.
- 브라우저는 요청을 두 갈래로 나누어서 처리한다. 바로 보내서 처리하는 것도 있는데 보내도 되는지 먼저 물어보고 처리하는 것도 있다. → 단순 요청이랑 예비 요청

## CORS의 정의와 전제 조건 : SOP(동일 출처 정책)

- SOP(Same-Origin Policy, 동일 출처 정책) : 하나의 출처에서 로드된 스크립트가 다른 출처의 리소스와 상호작용하는 것을 제한하는 브라우저의 보안 모델이다.
- 이 제한이 없다면 악성 사이트를 접속했을 때, 그 사이트의 자바스크립트가 유저가 로그인해 둔 은행 사이트의 API를 마음대로 호출해 돈을 송금하는 등의 금융 사고가 발생할 수 있다.  브라우저는 이를 원천 차단한다.

출처의 판별 기준 : 프로토콜, 호스트, 포트 3가지가 모두 완전히 같아야 동일 출처로 판단한다. 하나라도 다르면 교차 출처가 된다.

https://my-site.com

- `http://my-site.com` ➔ **다른 출처** (프로토콜이 `https` vs `http`로 다름
- `https://dev.my-site.com` ➔ **다른 출처** (서브 도메인 호스트가 다름)
- `https://my-site.com:5600` ➔ **다른 출처** (포트 번호가 명시되어 다름)

## CORS의 본질

- CORS는 여는 기술이다. CORS는 요청을 막는 기술이 아니라 브라우저가 기본적으로 막아둔 SOP정책을 서버 허락을 받아 안전하게 완화하여 얼어주는 표준 약속이다.
- 역할분담
    - 허용 여부 결정 주체 = 서버 (응답 헤더에 “이 출처는 내 리소스를 써도 좋다”라고 허락 조건을 명시한다.)
    - 최종 집행 주체 = 브라우저 (서버가 준 응답 헤더를 검사하여 진짜 자바스크립트에 데이터를 넘겨줄지, 아니면 차단할지 최종 판단한다.)
- CORS의 적용 범위 : 오직 브라우저에서 실행되는 자바스크립트 엔진에서만 발생한다. 서버가 다른 서버를 직접 호출(서버 간 통신)할 때는 CORS정책이 적용되지 않는다.

**요즘 웹 구조에서 CORS가 필수적인 이유**

- 과거에는 서버가 HTML을 직접 템플릿 엔진으로 만들어 서빙했으므로 출처가 다를 일이 없었다.
- 현재는 프론트엔드와 백엔드 서버를 분리하여 배포하는 것이 표준이다. 프론트엔드는 `https://my-app.com`에, API 서버는 `https://api.my-app.com` 에 띄우기 때문에 도메인이 달라 교차 출처 요청이 필수적이다. MSA구조나 외부 오픈 API를 클라이언트에서 직접 부를 때도 마찬가지이다.

## 브라우저의 두 가지 요청 처리 방식

브라우저는 교차 출처 요청을 조건에 따라 두 갈래로 나누어 처리한다.

### 단순 요청(Simple Request)

별도의 예비 절차 없이 브라우저가 본 요청을 서버로 바로 보내는 방식이다. 단 아래의 조건을 모두 만족해야만 한다.

- **조건 1:** HTTP 메소드가 `GET`, `POST`, `HEAD` 중 하나여야 함.
- **조건 2:** `Content-Type` 헤더가 `multipart/form-data`, `text/plain`, `application/x-www-form-urlencoded` 중 하나여야 함. (`application/json`은 탈락)
- **조건 3:** 커스텀 헤더를 사용하지 않고 브라우저의 기본 헤더만 포함되어야 함.

#### 단순 요청 흐름

1. 브라우저가 요청을 보낼 때 Origin 헤더에 현재 프런트엔드의 출처를 자동으로 담아 보낸다.
2. 서버는 요청을 정상 처리한 뒤에, 응답 헤더에 `Access-Control-Allow-Origin` 값으로 허용 출처를 담아서 반환한다.
3. 브라우저가 응답 헤더를 확인하고, 서버가 허용한 출처 목록에 자신의 Origin이 없다면 CORS에러를 내며 응답 데이터 통과를 거절한다.

→CORS 에러가 발생했을 때 백엔드 로그를 보면 서버는 정상적으로 200OK 응답으로 보낸 경우가 많음. 즉, 통신 실패나 서버다운이 아니라 서버는 일을 다 처리했는데 브라우저 단계에서 최종 차단한 것임.

### 예비 요청

우리가 흔히 쓰는`application/json` 데이터 포맷을 사용하거나 커스텀 헤더(예: `Authorization` 인증 토큰)를 붙이면 단순 요청 조건을 벗어나므로 **무조건 예비 요청으로 빠진다.**

#### 예비 요청 흐름

1. **OPTIONS메서드로 예비 요청 전송** : 본 요청을 보내기 전, 브라우저가 OPTIONS 메서드를 사용해서 서버에 먼저 보낸다. “내가 앞으로 이런 출처에서, 이런 메서드랑 헤더로 진짜 요청을 보낼 건데 허용해 줄 수 있어?”라고 물어보는 것이다.
2. **서버의 정책 응답** : 서버는 자신이 허용하는 정책들을 응답 헤더에 담아서 돌려준다.
- `Access-Control-Allow-Origin`: 허용 출처
- `Access-Control-Allow-Methods`: 허용 HTTP 메소드 목록
- `Access-Control-Allow-Headers`: 허용 헤더 목록
1. **본 요청 전송 여부 판단** : 브라우저가 이 응답을 분석하여 안전하다고 판단되면 그제야 진짜 본 요청을 보낸다. 만약 예비 요청에서 거부당하면 본 요청은 서버로 출발조차 하지 않고 CORS에러를 띄운다.

개발자 도구 네트워크 탭에서 내가 본 적 없는 OPTIONS요청이 먼저 찍혀 있는 것은 브라우저가 예비 요청을 알아서 보낸 정상 현상이다.

## Spring Security기반 CORS 설정 방법

CORS는 스프링 MVC 레벨에서도 설정할 수 있지만, 설정이 유실되거나 보안 필터를 거치기 전에 차단되는 한계가 있어서 시큐리티 레벨에서 관리하는 것을 권장

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // 1. 시큐리티 필터 체인에 CORS 설정 활성화
            .cors(cors -> cors.configurationSource(corsConfigurationSource())); 
        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        
        // ① 허용할 프론트엔드 출처(Origin) 지정
        configuration.setAllowedOrigins(List.of("https://my-app.com", "http://localhost:3000")); 
        
        // ② 허용할 HTTP 메소드 지정
        configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));      
        
        // ③ 허용할 헤더 지정 (JWT 토큰용 Authorization, JSON용 Content-Type 등)
        configuration.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Custom-Header")); 
        
        // ④ 쿠키나 인증 헤더(Credentials)를 요청에 포함할 수 있게 허용
        configuration.setAllowCredentials(true);  
        
        // ⑤ 브라우저가 예비 요청(Preflight) 결과를 매번 보내지 않고 캐싱할 시간 (1시간)
        configuration.setMaxAge(3600L);           

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        // 모든 URL 경로(/**)에 대해 위에서 설정한 CORS 규칙을 적용
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
}
```

## 정리

- CORS는 허용하는 메커니즘이다. 브라우저의 동일 출처 정책 때문에 막혀있는 교차 출처 요청을 서버의 허락을 받아서 안전하게 열어주는 표준 약속이다.
- 판단과 결정의 주체가 다르다. 허용정책을 정하는 것은 서버이고, 그 정책을 응답 헤더로 받아서 최종적으로 차단할지 통과시킬 지는 브라우저가 집행한다.
- CORS에러는 서버 통신 실패가 아니라 브라우저 단계의 차단이다.
- 요청은 단순요청이랑 예비요청 두가지로 나뉜다. 예비요청은 OPTIONS로 먼저 요청을 보내서 허락받는 과정을 거친다.
