## 지금까지 배운 것과 Controller의 필요성

- IoC(제어의 역전) : 객체의 생성과 생명주기 관리를 개발자가 아닌 스프링 컨테이너에게 위임하는 것
- Bean(빈) : 스프링 컨테이너 안에서 생성되고 관리받는 자바 객체
- AOP(관점 지향 프로그래밍) : 비즈니스 로직과 공통 로직(로그, 트랜잭션 등)을 분리하여 횡단 관심사를 해결하는 것
- DI(의존성 주입) : 개발자가 자바 객체의 생성과 관리를 직접 신경 쓰지 않도록 외부에서 의존성을 주입해 주는 것
- Spring MVC & DispatcherServlet : 웹 요청이 들어왔을 때 DispatcherServlet을 중심으로 단계별로 어떻게 요청을 처리하고 다루는지에 대한 흐름

지금까지 배운 스프링 프레임워크의 기초가 “건물의 뼈대를 세우고 전기 배선이나 수도를 깔아놓은 기초 공사”라면 Controller는 외부에서 사람이 건물 안으로 들어올 수 있도록 만드는 “문”역할을 한다. 아무리 건물이 튼튼해도 문이 없으면 의미가 없듯이 Controller가 있어야 화면을 띄우거나 프론트엔드 개발자와 실제로 데이터를 주고받는 API를 만들 수 있다.

## Controller계층 개요 및 사용 이유

웹 요청(HTTP)을 자바 객체에 맞게 번역하고, 처리된 자바 객체를 다시 HTTP응답으로 번역해서 반환하는 계층이다.

→ 클라이언트의 요청을 받아 처리 흐름을 시작하고, 적절한 비즈니스 로직(Service)을 호출한 뒤, 응답 형태를 결정하는 역할을 담당하는 계층이다.

**계층 분리의 중요성** : 만약 Controller가 중개자 역할만 하지 않고, 내부에서 JSON데이터를 읽는 것뿐만 아니라 DB에 직접 접근하고 비밀번호를 암호화하는 등 다른 역할까지 수행한다면 Controller가 너무 비대해진다.  

→ 향후 웹 화면에서 모바일 앱으로 요청 형식이 조금만 달라져도 Controller를 통째로 뜯어고쳐야 하는 문제가 발생한다. 따라서 명확한 경계선을 가지고 역할을 쪼개야 한다.

사용 이유

- 웹 요청과 응답 처리 책임을 비즈니스 로직과 철저히 분리하여 변경 범위를 줄이고 요청 흐름을 일관되게 관리하기 위함이다.
- 프로젝트 진행 중 기획 요구사항 변경의 영향을 가장 많이 받는 곳은 클라이언트(프론트엔드)이다.
    
    만약 계층 분리가 안 되어 있다면 이러한 사소한 API 스펙 변경이 비즈니스 로직(Service 계층)까지 영향을 주어 불필요하게 코드를 수정해야 하는 상황이 생긴다. 계층 분리가 되어 있으면 고쳐야 할 지점이 명확하게 격리된다.
    

## 사용 방법

### 기본적인 동작 방식

- 클래스 위에 @Controller 또는 @RestController 어노테이션을 선언하면 애플리케이션 시작 시점에 스프링 컨테이너가 이를 Bean으로 등록하고 “이 클래스는 외부 HTTP요청을 처리하는 역할”임을 인식한다.
- 클래스 내부에 메서드를 만들고 @GetMapping(”/users”)같은 어노테이션을 붙여두면, DispatcherServlet과 HandlerMapping이 이 정보를 수집해 두었다가 “클라이언트가 GET방식으로 /users 주소로 요청하면 이 메서드를 실행해야지”하고 길을 찾아준다.

### @RequestMapping과 @GetMapping

- 라우팅 : 클라이언트가 브라우저 주소창이나 Postman으로 요청한 URL을 백엔드 서버의 어떤 자바 메서드가 처리할지 길을 뚫어주는 작업이다.
- @RequestMapping :  범용적인 어노테이션으로, 클래스 레벨과 메서드 레벨 모두 붙일 수 있다. 주로 클래스 맨 위에 붙여 해당 컨트롤러 내부 API들이 공통 기본 시작 주소(예 : /users)를 묶어주는 역할을 한다.
- HTTP 메서드별 매핑(동일한 주소, 다른 동작)
    - GET /users → 유저 목록 조회
    - POST /users → 새로운 유저 자원 추가(회원가입)
    - PUT /users/1 → 1번 유저의 정보를 통째로 수정
    - DELETE /users/1 → 1번 유저 삭제

## @Controller vs @RestController

두 어노테이션 모두 해당 클래스가 컨트롤러 역할을 한다는 점은 같지만, 스프링 MVC 내부에서 응답을 만들어내는 매커니즘은 완전히 다르다.

| **구분** | **@Controller** | **@RestController** |
| --- | --- | --- |
| **주 목적** | 화면(View, HTML)을 리턴하기 위한 목적 | 순수한 데이터(JSON/XML)를 리턴하기 위한 목적 |
| **개발 방식** | 전통적인 서버 사이드 렌더링(SSR) 방식 (JSP, Thymeleaf 등 사용) | 현대적인 프론트-백엔드 분리 구조 (React, Vue 등과 REST API 통신) |
| **동작 메커니즘** | 메서드가 문자열(예: `"home"`)을 반환하면 View Resolver(뷰 리졸버)가 개입하여 특정 폴더의 `home.html`을 찾아 데이터를 섞어 HTML 페이지를 반환함. | `@Controller` + `@ResponseBody`의 결합 형태. View Resolver가 관여하지 않고, MessageConverter(메시지 컨버터)가 개입함. |
| **응답 처리** | 반환된 문자열을 **화면 파일의 경로**로 해석 | 자바 객체(예: `UserResponse`)를 가로채서 JSON 문자열로 **직렬화**한 뒤, HTTP 응답 바디(Body)에 직접 담아 전달 |

## DTO(Data Transfer Object)

- 데이터 전송 객체 : 계층과 계층 사이, 또는 서버와 클라이언트 사이에서 순수하게 데이터를 실어나르기 위해 사용하는 객체이다.
- 복잡한 비즈니스 로직이나 데이터 설정 로직은 전혀 없으며, 순수하게 변수와 이를 넣고 뺄 수 있는 기능(getter, setter 혹은 생성자)만 단순하게 선언해서 사용한다.

```jsx
@Getter // 필드에 대한 getter 메소드 만들어주고
@NoArgsConstructor // 기본 생성자 
public class UserRequestDto {
    private String email;
    private String password;
    private String nickname;
}

@Getter
@NoArgsConstructor
public class UserResponseDto {
    private Long id;
    private String email;
    private String nickname;

    public UserResponseDto(User user) {
        this.id = user.getId();
        this.email = user.getEmail();
        this.nickname = user.getNickname();
    }
}
```

#### DTO클래스를 써야하는 이유

1. 보안 및 데이터 노출 제한 : 응답 데이터의 원천은 DB이다. 하지만 DB 유저 테이블의 모든 컬럼 정보를 클라이언트에게 다 반환할 필요는 없다. 만약 데이터베이스 엔티티 객체를 그대로 반환하면 불필요한 정보뿐만 아니라 비밀번호, 개인정보 같은 민감한 데이터까지 그대로 노출될 위험이 있다.
2. API 스펙의 최적화 : 해당 API의 요청과 응답에 딱 필요한 필드만 맞춤형으로 구성하여 통신 비용을 줄이고 안전성을 높일 수 있다.
3. 계층 간 결합도 분리 : DB구조(Entity)가 변경되더라도 클라이언트와 주고받는 API 스펙(DTO)은 유지할 수 있어서 시스템의 유연성이 높아진다.

## 질문

Q. Controller와 RestController의 차이가 무엇이냐?

두 어노테이션 모두 웹 요청을 받는 컨트롤러 역할을 하지만 **응답을 생성하는 방식**에서 차이가 있다. `@Controller`는 주로 HTML 화면(View)을 반환하기 위해 사용되며 **View Resolver**가 개입하여 문자열을 화면 경로로 해석한다.

반면 `@RestController`는 `@Controller`와 `@ResponseBody`가 결합된 형태로, 순수 데이터를 반환한다. **MessageConverter**가 자바 객체를 JSON 문자열로 직렬화하여 HTTP 응답 바디에 직접 담아 클라이언트에게 전달한다.

Q.DTO를 왜 사용해야 하냐?

**계층 간의 결합도를 분리**하고 **보안을 강화**하기 위해 사용한다. DB 엔티티 객체를 그대로 노출하면 민감한 개인정보까지 클라이언트에 전달될 위험이 있고, DB 설계가 바뀔 때 API 스펙까지 같이 흔들리게 된다. DTO를 사용하면 특정 API 요청과 응답에 딱 필요한 데이터만 선별하여 안전하고 독립적으로 전송할 수 있다.

## 정리

Controller는 외부의 HTTP 요청과 내부의 비즈니스 로직을 연결해주는 관문 역할을 하며, 그 사이에는 안전하고 깔끔하게 데이터만 실어 나르는 상자가 DTO이다. 

백엔드가 순수 API서버 역할을 할 때는 @RestController를 사용한다.
