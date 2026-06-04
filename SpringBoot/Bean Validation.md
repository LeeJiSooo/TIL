## 유효성 검사의 필요성

Controller와 Service계층을 나누고 DTO를 통해 데이터를 주고받더라도 코드가 지저분해지는 주범이 있다. “사용자가 보낸 데이터가 올바른가”를 검사하는 수많은 if문 조건들이다.

예 : “제목은 26자 이하여야 한다”, “비밀번호는 8자리 이상이어야 한다”, “이메일은 필수다”

- 프론트엔드 검증 : 화면단에서 자바스크립트 등으로 입력값을 제한하는 것은 ‘사용자 편의성(UX)’을 위한 것이다. 오타나 잘못된 입력을 빠르게 안내해 서버 통신 횟수를 줄여준다.
- 백엔드 검증(필수) : 백엔드는 그 어떤 상황에서도 외부의 입력을 신뢰해서는 안된다. 악의적인 사용자는 프론트엔드 화면을 거치지 않고 Postman이나 curl같은 도구로 서버 API에 직접 비정상적인 데이터를 꽂아 넣을 수 있기 때문이다. 따라서 보안과 데이터 무결성을 위해 백엔드 자체 검증은 필수적이다.

## Bean Validation이란?

객체의 값이 특정 제약 조건을 만족하는지 기술 사양에 따라 자동으로 검증해 주는 표준 기반 검증 기능이다.

검증 규칙의 객체 응답

- REST API환경에서 프론트엔드가 보낸 JSON 데이터는 낱개의 변수가 아니라 하나의 DTO객체(덩어리)로 변환되어 들어온다.
- Bean Validation은 검증 로직을 컨트롤러나 서비스 메서드 내부에 적지 않는다. 대신 데이터를 담는 객체(DTO)구조 자체에 어노테이션으로 제약 조건을 선언해 버린다.
- 이메일 형식 검증 규칙이 회원가입 DTO에 한 번 고정되면, 이 DTO가 회원가입에 쓰이든, 정보 수정에 쓰이든, 관리자 페이지에 쓰이든 상관없이 언제나 동일한 기준으로 일관되게 검증한다.

[나쁜 예시] 수동으로 유효성 검증을 수행하는 Controller

```jsx
@RestController
@RequestMapping("/orders")
public class OrderController {

    @PostMapping
    public ResponseEntity<String> createOrder(@RequestBody CreateOrderRequest request) {
        // 비즈니스 로직 시작 전에 매번 반복되는 지저분한 if문 검증들
        if (request.getProductId() == null) {
            return ResponseEntity.badRequest().body("상품 ID는 필수입니다.");
        }
        if (request.getQuantity() < 1) {
            return ResponseEntity.badRequest().body("수량은 1개 이상이어야 합니다.");
        }
        if (request.getAddress() == null || request.getAddress().trim().isEmpty()) {
            return ResponseEntity.badRequest().body("배송지 주소는 필수입니다.");
        }
				
        // --- 여기서부터가 진짜 처리하고 싶은 핵심 비즈니스 로직 ---
        orderService.create(request);
        return ResponseEntity.ok("주문 성공");
    }
}
```

진짜 하고 싶은 핵심 비즈니스 코드가 수많은 if문에 가려져 한눈에 들어오지 않는다. 만약 입력 받아야할 필드가 3개가 아니라 15, 20개로 늘어나면 컨트롤러 메서드 전체가 if문으로 도배되어 가독성과 유지보수성이 떨어진다.

[좋은 예시] Bean Validation을 적용한 구조

1) 제약 조건이 선언된 DTO

```jsx
public class CreateOrderRequest {

    @NotNull(message = "상품 ID는 필수입니다.") // null 허용 안 함
    private Long productId;

    @Min(value = 1, message = "수량은 1개 이상이어야 합니다.") // 최소 1 이상
    private int quantity;

    @NotBlank(message = "배송지 주소는 필수입니다.") // null 안 되고, 빈칸만 있는 것도 안 됨
    private String address;

    // Getter, Setter 생략
}
```

DTO 코드만 봐도 이 객체가 어떤 상태여야 정상적인 데이터인지 한눈에 파악할 수 있다.

2) 조건문이 완전히 사라진 Controller

```jsx
@RestController
@RequestMapping("/orders")
public class OrderController {

    @PostMapping
    public ResponseEntity<String> createOrder(
            @Valid @RequestBody CreateOrderRequest request // @Valid 하나로 수십 줄의 if문 대체
    ) {
        // 검증을 완벽히 통과해야만 이 라인부터 코드가 실행된다.
        orderService.create(request);
        return ResponseEntity.ok("주문 성공");
    }
}
```

## 컨트롤러 레벨의 유효성 검사 동작 원리

1. 클라이언트가 보낸 JSON데이터를 HttpMessageConverter가 자바 객체(CreateOrderRequest)로 변환합니다.
2. 컨트롤러 메서드를 실행하기 직전 스프링이 가로채서 파라미터에 붙은 @Valid 어노테이션을 확인하고 DTO객체 내부의 제약 조건들(@NotNull, @Min 등)을 검사한다.
3. 만약에 검증에 실패하면?
    
    즉시 `MethodArgumentNotValidException`이라는 예외를 발생시킨다.
    
    스프링 기본 설정에 의해 클라이언트에게 `400 Bad Request` 응답을 반환하며, 컨트롤러 내부 코드는 단 한 줄도 실행되지 않고 튕겨 나간다. 시스템 내부로 불량품 객체가 진입하는 것을 완벽히 방지하는 스크리닝 역할을 한다.
    

## Service 레벨의 유효성 검사와 AOP 동작 원리

컨트롤러에서 이미 @Valid로 검사했는데, 서비스 계층에서 또 검증해야 하나?

- 서비스 계층은 애플리케이션 내부의 비즈니스 진입점이다. **orderService.create()**메서드는 웹 브라우저(Controller)를 통해서만 호출되는 것이 아니라, **저녁 6시 정기 주문 배치(Batch)스케줄러나 메시지 큐의 비동기 리스너** 등 내부 요소를 통해서도 호출될 수 있다.
- 이때는 웹 기술인 Controller를 거치지 않으므로 @Valid를 우회하여 불량품 객체가 서비스 로직으로 직접 들어올 위험이 있다. 이를 방지하기 위해 서비스 레벨에서도 검증 시스템을 구축한다.

```jsx
@Service
@Validated // 1. 클래스 레벨에 선언하여 제약 조건 검증 기능을 활성화
public class OrderService {

    // 2. 파라미터 앞에 @Valid를 붙여 검증 대상 명시
    public void create(@Valid CreateOrderRequest request) {
        // 비즈니스 로직
    }
}
```

- **동작 원리 (스프링 내부의 AOP):**

1. 클래스 위에 `@Validated`가 붙어 있으면, 스프링 컨테이너는 애플리케이션 구동 시점에 `OrderService` 빈을 생성할 때, 순수 객체 대신 이를 한 번 감싼 '껍데기 프록시(Proxy) 객체'를 만들어 빈으로 등록한다.
2. 스케줄러든 누구든 `create()` 메서드를 호출하려고 하면, 진짜 서비스 로직이 실행되기 전에 **AOP 프록시 메커니즘**에 의해 프록시가 먼저 파라미터(`CreateOrderRequest`)를 가로챈다.
3. 제약 조건을 검증한 후, 문제가 없을 때만 진짜 서비스 메서드를 실행시킨다. 만약 실패하면 `ConstraintViolationException` 계열의 예외를 발생시킨다.

→ 우리가 직접 비즈니스 로직에 공통 관심사를 분리하는 AOP 코드를 작성하지 않더라도, 스프링 프레임워크 내부에서는 이미 Bean Validation 기능을 구현하기 위해 **AOP 개념을 적극적으로 활용**하고 있다.

## 정리

- BeanValidation 핵심은 검증 로직을 메서드 레벨에 두지 않고 데이터를 담는 객체 구조 자체에 어노테이션으로 제약 조건을 나타내는 방법
- 프론트엔드의 유효성 검사는 사용자의 편의를 위한것이지 믿어서는 안되고 백엔드에서 별도로 필수로 해야한다.
- 별도로 Service를 호출하는 경우 서비스 레벨에서도 유효성 검사가 필요한 경우들이 생긴다.
