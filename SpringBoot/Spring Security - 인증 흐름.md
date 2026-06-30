## 가져가야 할 핵심

- 로그인 인증은 컨트롤러가 아니라 시큐리티 필터 체인 단계에서 끝난다. 비즈니스 로직 도달하기 전에 이미 누구인지가 판별된다.
- 로그인 한번은 한 객체가 다 처리하지 않고, 여러 부품들이 책임을 나눠서 맡는다. 요청을 가로채는 필터, 일을 배분하는 매니저, 실제로 검증하는 프로바이더, 사용자 찾아오는 userDetailService같은 부품들의 역할을 알아야 한다.
- 이 과정의 결과물은 Authentication이라는 인증 정보를 담는 곳이 있고, 미인증 상태로 출발해서 검증을 통과하면 인증 완료 상태로 바뀌어서 SecurityContext에 저장된다.

## 핵심 객체와 중간 계층

#### Authentication (인증 정보의 그릇)

“누가 인증을 시도하는가”와 “인증이 성공했는가”에 대한 정보를 담는 상자이다. 내부에는 사용자의 신원 정보(Principal), 비밀번호 같은 자격 증명(Credentials), 유저가 가진 권한 목록(Authorities)가 담긴다.

**두 가지 상태**

1. 미인증 상태 : 사용자가 로그인 버튼을 누른 직후, 아이디와 비밀번호만 담겨 검증을 기다리는 상태
2. 인증 완료 상태 : 검증을 통과한 후, DB에서 조회한 실제 사용자 정보와 권한 목록이 채워진 상태

→ Authentication 객체가 ‘미인증’에서 ‘인증 완료’상태로 전환되는 과정을 추적하는 과정

#### UserDetails & UserDetailsService (중간 계층 어댑터)

- **UserDetails** : 우리 서비스의 유저 테이블 구조나 컬럼명은 제각각이어서 스프링 시큐리티는 우리의 DB구조를 알지 못 한다. 따라서 시큐리티가 이해할 수 있는 표준 사용자 정보 형식으로 맞춰주는 어댑터 역할의 인터페이스이다.(아이디, 암호화된 비밀번호, 권한, 계정 만료 여부 등을 포함)
- **UserDetailService** : 아이디(username)를 줄 테니 우리 DB에서 사용자를 찾아 `UserDetails` 형태로 리턴해 달라고 명력받는 조회 전용 서비스 인터페이스이다.

## 폼 로그인 처리 내부 아키텍처 및 동작 방식

<img width="2100" height="1270" alt="image" src="https://github.com/user-attachments/assets/75a9bfd9-4615-4b3e-abff-be0be71a5c81" />


SecurityFilterChain이라는 박스 안에서 벌어지는 일들들 확대해서 보는 것

#### 1. UsernamePasswordAuthenticationFilter (입구 및 흐름 제어)

- 폼 로그인을 활성화하면 동작하는 필터로, 로그인 처리URL로 들어오는 요청을 가로챈다.(GET 요청이나 다른 URL은 그냥 통과시킨다)
- 로그인이 안 될 때 디버깅의 첫 단계는 “요청이 이 필터까지 도달했는가?”를 확인한는 것이다.

#### 2. Authentication 객체 생성 (미인증 상태)

- 필터는 HTTP 요청에서 사용자가 입력한 아이디와 비밀번호를 꺼내서     `UsernamePasswordAuthenticationToken`(Authentication의 구현체)라는 상자에 담는다.
- 이 시점에는 아직 검증되지 않은 미인증 상태이다.

### 3. AuthenticationManager (조정자 / 관리자)

- 필터는 직접 검증하지 않고 `AuthenticationManager`에게 이 미인증 상자를 넘긴다.
- 매니저도 직접 실무를 보지 않고, “이 인증 방식(폼 로그인, 소셜 로그인, API Key 인증 등)을 처리할 수 있는 적절한 사람이 누구지”를 판단해서 알맞는 `AuthenticationProvider`에게 위임한다.
- 책임을 나눈 이유 : 새로운 인증 방식을 추가할 때 매니저는 그대로 두고 새로운 프로바이더만 만들어서 갈아끼우면 되니까 결합도가 낮아지고 확장이 깔끔해진다.

#### 4. AuthenticationProvider (실무자 / 실제 검증 진행)

가장 일반적인 폼 로그인에서는 `DaoAuthenticationProvider`가 선택되어 실무를 맡아. 여기서 **3가지 핵심 작업**이 순서대로 일어난다.

1. 사용자 조회 : `UserDetailsService.loadUserByUsername(username)`을 호출해 DB에서 유저를 찾아온다. 유저가 없다면 `UsernameNotFoundException`이 발생하며 중단된다. 조회가 성공하면 표준 규격인 `UserDetails`로 돌려받는다.
2. 비밀번호 비교 : 사용자 가 입력한 비밀번호와 DB에서 가져온 암호화된 비밀번호를 `PasswordEncoder`를 사용해 비교한다. 
    
    → 복호화하는 게 아니라, 입력받은 평문을 동일한 알고리즘으로 해시하여 DB의 해시 값과 일치하는지 확인하는 방식이다. 일치하지 않으면 예외가 터진다.
    
3. 인증 완료 객체 생성 : 비밀번호까지 맞으면, 프로바이더는 유저 정보와 권한 목록을 채운 ‘인증 완료 상태’의 새로운 Authentication객체를 만들어 매니저를 거쳐 필터로 돌려보낸다.

#### SecurityContext저장 및 세션 유지

- 인증 완료 객체를 받은 필터는 이를 `SecurityContex`에 저장한다. 이 컨텍스트는 `SecurityContextHolder(ThreadLocal)` 에 보관되므로, 이 요청이 끝날 때까지 컨트롤러나 서비스 등 코드 어디에서나 유저 정보를 꺼내 쓸 수 있게 된다.(파라미터로 들고 다닐 필요 없음)
- 상태 유지 : 스레드 로컬 공간은 요청이 끝나면 사라지기 때문에, 다음 요청에서도 로그인 상태를 유지하기 위해 시큐리티는 이 `SecurityContext`를 `HttpSession`(톰캣 세션) 공간에 복사해서 보관한다.

**SecurityConfig.java (보안 설정 빈)**

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/login", "/css/**").permitAll() // 로그인 페이지와 정적 리소스는 무조건 허용
                .anyRequest().authenticated()                     // 그 외 모든 요청은 인증 필수
            )
            .formLogin(form -> form
                .loginPage("/login")                  // 내가 만든 커스텀 로그인 페이지 URL
                .loginProcessingUrl("/login-process") // HTML Form 태그의 action URL (필터가 가로챌 주소)
                .defaultSuccessUrl("/", true)         // 로그인 성공 시 이동할 메인 페이지
                .permitAll()
            );

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(); // 비밀번호 단방향 해시 암호화 컴포넌트 등록
    }
}
```

**CustomUserDetailsService.java (실제 DB 조회 구현체)**

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    public CustomUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    // Provider가 아이디를 던져주며 호출하는 메소드
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // 1. 우리 DB에서 이메일(아이디)로 유저 조회
        User user = userRepository.findByEmail(username)
                .orElseThrow(() -> new UsernameNotFoundException("해당 유저를 찾을 수 없습니다: " + username));

        // 2. 시큐리티가 이해할 수 있는 표준 규격 객체(UserDetails의 구현체인 User)로 변환해서 리턴
        return new org.springframework.security.core.userdetails.User(
                user.getEmail(),
                user.getPassword(), // 암호화되어 저장된 비밀번호
                Collections.emptyList() // 우선 권한 목록은 빈 리스트로 설정
        );
    }
}
```

## 정리

- 큰 그림은 로그인 요청 하나가 시큐리티 필터 체인 안에서 여러 부품을 거쳐서 처리된다.
- 부품의 책임이 다 다른데 알아두기
- UsernamePasswordAuthenticationFilter는 로그인 요청 가로채고 미인증 상자(`Authentication`)를 만들어서 흐름을 시작하는 **입구**.
- AuthenticationManager는 들어온 인증 요청을 보고 어떤 프로바이더에게 일을 맡길지 결정하는 관리자
- AuthenticationProviderUserDetailService는 실제 DB 데이터와 비교하고 비밀번호 일치 여부를 검증하는 진짜 인증 실무자
- UserDetailService는 우리 DB에 접근해서 사용자를 찾아 시큐리티 규격(UserDetails)으로 변환해주는 조회 대리인
- 이 모든 과정의 결과물이 Authencation 객체이다.
    - 미인증 상태였다가 검증 통과하면 인증 완료상태의 객체로 다뤄진다. 이게 시큐리티 컨텍스트에 저장된다.
