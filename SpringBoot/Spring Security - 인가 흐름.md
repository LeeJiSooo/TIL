## 가져가야 할 핵심

- 인가는 인증이 끝난 뒤에 진행된다. 시큐리티 컨텍스트에 들어있는 Authentication을 전제로 이 사람이 작업을 할 권한이 있는지를 따지는 단계이다.
- 판단의 주체는 둘로 나뉜다. Authorization Filter는 흐름만 제어하고 실제 허용하고 거부하는 판단은 AuthorizationManager가 요청정보와 Authentication 설정 규칙을 비교해서 내린다.
- 결과는 허용되면 컨트롤러까지 가고, 거부되면 AccessDeniedException이 발생한다. 인증 자체가 안됐으면 401, 인증은 됐는데 권한이 없으면 403이다.

## 인가의 정의와 전제 조건

- 인증된 사용자가 특정 리소스(URL, 메소드 등)에 접근하거나 특정 행위를 할 수 있는 권한이 있는지 최종적으로 확인하고 허락하는 과정이다.
- 반든시 인증이 선행되어야 한다. 신원이 확인되지 않은 사람을 두고 어떤 권한을 줄지 논의하는 것은 의미가 없기 때문이다.
- 앞단의 인증 필터를 거쳐 `SecurityContext`내부에 유효한 `Authentication` 객체가 존재한다는 사실이 인가를 시작할 수 있는 전제이다.

## 인가의 흐름 및 동작 원리

!image.png

사용자가 GET /admin/dashboard같은 요청을 보냈다고 가정

→ 이 요청은 앞서 인증 필터를 통과해서 로그인이 된 상태이다.

→ 인증이 끝나서 시큐리티 컨텍스트에 Authentication객체가 들어가 있는 상태이다.

→ 여기서부터 인가가 시작된다.

#### AuthorizationFilter(흐름 제어)

- 인증을 마친 요청이 필터 체인의 후반부에 위치한 AuthorizationFilter에 도달한다.
- 이 필터는 인가 판단을 직접 내리지 않고, 현재 요청 정보와 인증된 사용자 정보(`Authentication`)를 통째로 `AuthorizationManager`에게 넘기며 결정을 위임한다.
- 책임 분리의 이점 : 인증 흐름에서의 Manager-Provider의 관계처럼, 흐름을 제어하는 쪽(필터)과 실제 권한을 판단하는 쪽(매니저)을 분리해서 둔다.
    
    → 인가 규칙이나 비즈니스 정책이 바뀌어도 필터 코드는 손댈 필요 없이 매니저의 판단 로직만 유연하게 교체할 수 있다.
    

#### AuthorizationManager(실제 권한 판단 실무자)

- 위임받은 `AuthorizationManager` 는 우리가 `SecurityConfig` 설정 파일에 등록해 둔 인가 규칙들을 위에서부터 차례때로 살펴본다.
- 현재 요청된 URL 경로가 요구하는 권한(예: `/admin/`은 `ROLE_ADMIN` 필요)이 무엇인지 확인하고, 사용자의 `Authentication` 객체가 그 권한을 실제로 보유하고 있는지 비교하고 검증한다.

#### 검사 결과에 따른 두 갈래 흐름(허용 vs 거부)

접근 허용 : 권한이 충분하다고 판단되면 다음 필터로 요청을 흘려보내고, 요청은 최종적으로 비즈니스 로직이 있는 컨트롤러에 도달한다.

접근 거부 : 권한이 부족하면 `AuthorizationFilter`가 `AccessDeniedException`예외를 발생시켜 흐름을 차단한다. 이 예외는 필터 체인 앞단에 위치한 `ExceptionTranslationFilter`가 낚아채서 후처리를 진행한다.

#### 401 Unauthorized vs 403 Forbidden

- 401 : 애초에 인증 자체가 안된 상태이다. 익명 사용자가 인증이 필수인 자원에 접근하려할 때 발생하며, 보통 로그인 페이지로 리다이렉트 시키거나 401에러를 응답
- 403 : 인증은 성공해서 누구인지는 알겠으나, 해당 자원에 접근할 권한이 부족한 상태이다. 일반 회원이 관리자 페이지에 접근할 때 `AccessDeniedException`에 의해 403Forbidden응답으로 끝남

## 인가 규칙 설정 방식(URL 기반 vs 메서드 기반)

이 둘은 경쟁 관계가 아닌 상호 보완적인 계층 관계이다.

#### URL 기반 보안 처리

SecurityConfig에서 URL패턴을 통해 넓은 범위의 큰 골격을 막아서 부적절한 요청을 처리하는 역할이다.

```java
.authorizeHttpRequests(auth -> auth
    // 1. /admin/** 패턴은 'ADMIN' 역할을 가진 사용자만 접근 가능
    .requestMatchers("/admin/**").hasRole("ADMIN") 
    
    // 2. /my-page/** 패턴은 'USER' 또는 'ADMIN' 역할을 가진 사용자만 접근 가능
    .requestMatchers("/my-page/**").hasAnyRole("USER", "ADMIN")
    
    // 3. 메인, 로그인, 회원가입 패턴은 인증 여부와 관계없이 무조건 접근 허용
    .requestMatchers("/", "/login", "/signup").permitAll()
    
    // 4. 그 외의 모든 요청은 일단 '인증된 사용자'만 접근 가능
    .anyRequest().authenticated()
)
```

**주의할 점** 

- 인가 규칙들은 위에서 아래로 순서대로 적용되며, 가장 먼저 일치하는 규칙이 선택되면 아래 라인은 무시된다. `anyRequest().authenticated()` 같은 포괄적인 규칙을 맨 위로 올리면, 그 아래에 있는 세부 규칙들(`permitAll` 등)이 동작하지 않아 **좁은 범위에서 넓은 범위 순서로 작성**해야 한다.
- `hasRole` vs `hasAuthority` 차이
    - 스프링 시큐리티는 기본적으로 역할을 다룰 때 앞에 ROLE_이라는 접두사를 붙여서 관리한다.
    - `hasRole("ADMIN")`: 자동으로 앞에 접두사를 붙여 내부적으로 `ROLE_ADMIN`이라는 권한을 찾아줘. (**역할 단위 제어**)
    - `hasAuthority("ROLE_ADMIN")`: 접두사를 자동으로 붙이지 않고, 개발자가 적은 문자열 그대로를 권한 이름으로 매핑해 검사해. (**특정 행위에 대한 세밀한 권한 제어**)

#### 메서드 기반 보안 처리

URL만으로는 “이 글의 작성자가 본인인가?”, “이 데이터의 소유자 권한인가?”와 같은 데이터 내용 기반의 세부적인 조건을 판단할 수 없다. 서비스나 컨트롤러 계층의 메서드에 직접 보안 어노테이션을 붙여서 제어한다.

```java
// [실행 전 검사] 관리자이거나, 게시글의 작성자가 현재 로그인한 사용자일 때만 메소드 실행 허용
@PreAuthorize("hasRole('ADMIN') or @postRepository.findById(#postId).get().getAuthor() == authentication.principal.username")
public void updatePost(Long postId, String newContent) { ... }

// [실행 후 검사] 일단 데이터를 조회해 온 뒤, 반환된 Post의 작성자가 현재 사용자가 아니면 예외 발생
@PostAuthorize("hasRole('ADMIN') or returnObject.author == authentication.principal.username")
public Post getPost(Long postId) { ... }
```

- 이러한 세부 데이터 권한 로직을 무조건 어노테이션으로만 짜야 하는 건 아니고, 일반 비즈니스 로직 내부에서 구현해도 된다.
- URL보안이 입구에서 허가된 사람만 들여보내는 1차 방어선이라면, 메서드 보안은 건물 내부 방 안에서 특정 데이터를 접근할 수 있는지 한 번 더 확인하는 2차 방어선이다.  → 이 둘은 같이 사용한다.

## 익명 사용자

스프링 시큐리티는 로그인하지 않은 사용자도 시스템 내에서 객체로 관리하는 메커니즘을 가지고 있다.

- 로그인을 안 한 상태로 요청을 보내면 시큐리티는 내부적으로 `AnonymousAuthenticationToken`이라는 특별한 토큰을 강제로 준다. 이 토큰은 내부적으로 `ROLE_ANONYMOUS` 권한을 갖는다.
- 왜 이렇게 하냐? 로그인을 했든 안 했든 상관없이 모든 웹 요청이 `Authentication` 객체를 가진 상태로 통일되기 때문이다.
    
    → 시큐리티 내부 코드나 인가 규칙 로직이 조건문 분기(nulll 체크 등)없이 일관되고 깔끔하게 처리될 수 있다.
    

**익명 사용자 관련 규칙**

- `permitAll()`: 토큰의 종류(익명 상태든, 일반 유저든, 관리자든)를 전혀 따지지 않고 **무조건 문을 열어주는 규칙**.
- `isAnonymous()`: 오직 `AnonymousAuthenticationToken`을 가진 상태, 즉 **로그인하지 않은 순수 익명 사용자에게만 접근을 허용하는 규칙** (예: 로그인 페이지, 회원가입 페이지 등).

## 정리

- 인가는 인증이 완료되어 `SecurityContext`에 `Authentication` 상자가 담긴 후에 작동한다.
- `AuthorizationFilter`는 요청을 낚아채는 흐름 제어만 맡고, 실제 규칙 비교와 권한 합격 여부는 `AuthorizationManager`가 전담한다.
- 권한이 부족하면 `AccessDeniedException`이 터지며, 앞단의 필터가 이를 낚아채 **403 Forbidden** 응답으로 클라이언트에게 돌려준다.
- 설정은 URL기반으로 할 수도 있고, 아니면 메서드 기반으로 할 수도 있다. 둘은 같이 쓰인다.
