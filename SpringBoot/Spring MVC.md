Spring MVC는 백엔드 서버 내부, 즉 “인터넷 망을 통해 HTTP요청이 들어오는 순간부터, 처리를 마치고 HTTP응답을 내보낼 때까지의 웹 계층”전체를 관장한다.

```jsx
[클라이언트: 모바일 앱/리액트] 
      │ (주문하기 클릭! HTTP Request)
      ▼
┌────────────────────────────────────────┐
│  [백엔드: Spring Boot 서버]               │
│  ─────── Spring MVC Layer ───────      │ 
│  1. 요청 접수 및 파싱 (DispatcherServlet)  │
│  2. 라우팅 및 객체 변환 (Mapping/Adapter)   │
│  3. 비즈니스 로직 호출 (Controller)         │
│                                        │
│  ─────── Business Layer ────────       │
│  4. 데이터 검증 및 주문 처리 (Service)       │
│  5. 데이터베이스 반영 (Repository -> DB)    │
└────────────────────────────────────────┘
      │ (주문 완료! HTTP Response - JSON)
      ▼
[클라이언트: 화면에 완료 UI 렌더링]
```

## 프레임워크 vs 라이브러리(제어의 역전 : IoC)

라이브러리 : 자신이 주도권을 가진다. 내가 필요할 때 원하는 도구를 호출한다.

프레임워크 : 프레임워크가 주도권을 가진다. 

제어의 역전(IoC) : 개발자가 직접 main()메서드를 만들고 순차적으로 코드를 실행하는 것이 아니라, 프레임워크가 먼저 실행되어 대기하다가 요청이 들어오면 내가 작성한 컨트롤러 메서드를 알아서 호출하는 현상을 말한다.

## Spring MVC의 핵심 구성요소

!image.png

- DispatcherServlet :  모든 HTTP 요청을 단 하나의 정문으로 받음. 공통 보안, 인코딩, 로깅을 단 한 곳에서 처리하여 효율성을 극대화함.
- HandlerMapping :  `@GetMapping("/users")` 같은 어노테이션 정보를 켜질 때 맵 구조로 등록해 두고, 요청 주소에 맞는 담당 직원을 찾아줌.
- ArgumentResolver : 클라이언트가 보낸 쌩 문자열이나 JSON을 자바 객체(`Long id`, `@RequestBody Object`)로 변환해 줌. 실패 시 **400 Bad Request** 발생.
- Controller : 파라미터를 받아 비즈니스 로직(Service)을 호출하고 결과 데이터를 반환함.
- MessageConverter : **현대 REST API의 핵심.** 자바 객체를 클라이언트가 이해할 수 있는 문자열인 **JSON**으로 변환하여 HTTP 응답 바디에 바로 꽂아줌.
- ViewResolver / View : (전통적 방식) Thymeleaf나 JSP 환경에서 문자열을 받아 실제 HTML 파일 경로를 찾고 데이터를 섞어 화면을 그려줌.

## REST API 전체 동작 흐름

```jsx
[HTTP Request] 
      │
      ▼
1. DispatcherServlet (요청 접수, HttpServletRequest 객체로 포장)
      │
      ▼
2. HandlerMapping (주소록 확인: "/order"는 OrderController가 담당하네!)
      │
      ▼
3. HandlerAdapter (주내용 전달을 위해 어댑터 연결)
      │
      ▼
4. ArgumentResolver (문자열 데이터를 자바 객체인 OrderRequest로 파싱 ── 실패 시 400 에러)
      │
      ▼
5. Controller 실행 (개발자가 짠 서비스 로직 작동 -> DB 잔액 차감 -> OrderResponse 자바 객체 반환)
      │
      ▼
6. MessageConverter (자바 객체를 텍스트인 JSON 문자열로 변환)
      │
      ▼
7. DispatcherServlet (완성된 JSON을 HTTP Response Body에 담아서 클라이언트에 반환)
      │
      ▼
[HTTP Response (JSON)]
```

1. **DispatcherServlet이 요청 수신**
    
    프론트엔드가 보낸 HTTP Request요청 자체(HTTP 메시지 덩어리)자체는 자바가 읽고 처리하는게 불가능하다. 그냥 문자열로만 되어있어 그걸 파싱하고 우리가 다룰 수 있게 만드는게 필요하다.
    
    디스패처 서브릿은 HTTP메시지 덩어리를 HTTP요청 자바 객체(HttpServletRequest)로 감싼다.
    
    이후 작업들이 편하게 진행될 수 있도록 세팅한다. → 인코딩이 어떻게 들어왔는지? 파일 업로드 요청인지?를 확인하고 뼈대를 잡는다. 여기서는 비즈니스 요청이 처리되지 않는다.
    
2. **HandlerMapping이 처리할 Controller를 찾는다.**
    
    디스패처 서블릿이 핸들러매핑(수첩)을 꺼내서 지금 들어온 요청이 우리가 갖고있는 컨트롤러 중 어떤거지?를 확인한다.
    
    핸들러 매핑은 스프링 실행할때 annotation을 다 스캔해서 매핑 테이블 하나를 만들어둔다.
    
    → 여기서 정확하게 일치하는 메서드의 컨트롤러를 찾아서 디스패치 서블릿에게 알려준다. 만약에 여기서 일치하는게 없으면 404에러를 던진다.
    
3. **HandlerAdapter가 Controller실행을 준비한다.**
    
    누구한테 일을 시킬지 컨트롤러를 찾는다. 
    
4. **ArgumentResolver가 메서드 파라미터 구성**
    
    핸들러 어댑터가 컨트롤러를 찌르기 전에 ArgumentResolver를 시켜서 파라미터 객체가 있다면 이걸 자바 객체로 생성하고 변환을 한다.
    
    ArgumentResolver는 HTTP Request 요청으로 있는 문자열을 뜯어서 자바에 맞는 객체나 값으로 변환시켜주는 역할을 한다.
    
    → 이 단계에서 만약 숫자가 들어갈 자리에 문자가 들어오거나 JSON파싱이 실패하게 되면 400에러를 던진다.
    

프레임워크가 해준 것들

---

---

우리가 작성한 코드가 실행되는 영역

1. **Controller메서드 실행**
    
    핸들러 어댑터가 컨트롤러 메서드를 호출해주고 파라미터를 집어넣어 준다. 그러면 우리가 작성한 서비스 코드가 돌아가고 컨트롤러가 다시 반환값을 리턴한다.
    

---

프레임워크가 해준 것들

1. **반환값을 응답 형태로 변환**
    
    컨트롤러가 작업을 마치고 리턴을한다. 리액트앱은 자바 객체를 모르기 때문에 바로 클라이언트에 던지면 안되고 JSON으로 변환을 해줘야 한다. MessageConverter는 백엔드가 리턴한 자바 객체를 가로채서 텍스트 기반의 JSON문자열로 변환한다. 프론트엔드랑 통신하는 REST API 서버면 여기서 변환된 JSON텍스트 데이터가 HTTP Response Body에 실리게된다.
    
2. **ViewResolver가 View결정(전통적인 템플릿 엔진을 사용할때만 거치는 과정)**

1. **View가 최종 응답 생성(전통적인 템플릿 엔진을 사용할때만 거치는 과정)**
    
    HTML문서랑 비즈니스 로직에서 처리된 값을 렌더링해서 완성된 HTML문서를 반환하는과정
    

### 문제가 터졌을 때 흐름을 그리며 디버깅 위치를 찾아야한다.

- **404 Not Found가 떴다?**
    - `DispatcherServlet`이 요청을 받아 `HandlerMapping`(주소록)을 뒤졌는데 일치하는 URL이나 HTTP 메서드(GET/POST)를 찾지 못했다.
    - 컨트롤러에 오타가 있나? `@RestController` 어노테이션을 빼먹었나? 프론트가 주소를 잘못 보냈나?
- **400 Bad Request가 떴다?**
    - `HandlerMapping`이 직원(컨트롤러)은 잘 찾았는데, `ArgumentResolver`가 데이터를 변환하는 과정에서 실패했다.
    - 숫자가 들어와야 하는 `Long id` 자리에 프론트가 `"abc"`라는 문자열을 보냈나? JSON 형식이 깨졌나? `@Valid` 검증에서 걸렸나?
- **500 Internal Server Error가 떴다?**
    - 프레임워크 단계는 완벽히 통과했는데, 우리가 작성한 자바 코드 내부(Controller, Service, DB)에서 예외가 터졌다.
    - `NullPointerException`이 발생했나? DB SQL 문법 오류가 있나? 비즈니스 로직 조건이 틀렸나?
- **로그인/토큰 검증 같은 공통 처리는 어디서 하나?**
    - 100개의 컨트롤러 메서드 첫 줄에 로그인 확인 코드를 넣는 것은 비효율적이다.
    - 이 흐름을 알면 "디바이스 서블릿이 컨트롤러를 호출하기 직전 길목"에 인터셉터(Interceptor)나 필터(Filter)라는 검문소를 세워 한 번에 처리할 수 있다는 해결책을 떠올릴 수 있다.
