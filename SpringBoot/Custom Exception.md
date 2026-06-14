- **예외(Exception)의 본질**
    - 예외는 단순히 프로그램의 '버그'나 '시스템 문제'만을 의미하는 것이 아니다.
    - 비즈니스 로직 상 **비정상적인 상태를 알리고 프로그램의 흐름을 제어하라는 강력한 신호**이다.
- **기본 예외의 한계 (기본 400, 500 에러의 문제)**
    - 서비스 계층에서 로직을 수행하다가 더 이상 진행할 수 없는 상태가 되면 `throw`로 예외를 상위로 던져야 한다.
    - 이때 자바가 기본 제공하는 `IllegalArgumentException` 같은 예외를 그대로 던지고 따로 핸들링하지 않으면, 스프링은 기본적으로 HTTP 상태 코드 400이나 500으로만 에러를 뱉어낸다.
    - **결과:** 프론트엔드(FE) 입장에서는 백엔드에서 왜 에러가 났는지 정확한 맥락(유저가 없는 건지, DB가 죽은 건지 등)을 구별할 수 없고, 결국 유저에게도 불친절한 안내를 할 수밖에 없다.
- **해결책: Custom Exception + Global Exception Handler**
    - 개발자가 직접 비즈니스 언어로 정의한 Custom Exception(사용자 정의 예외)을 만들고, **Global Exception Handler**를 통해 전역에서 예외를 가로챈다.
    - 이를 통해 서버에서 발생하는 에러를 깔끔하고 **일관된 규격으로 포장**하여 FE에 정확하게 전달할 수 있다.

## 가져가야 할 핵심

1. 자바의 범용적인 기본 예외 대신, 비즈니스 맥락을 담은 커스텀 익셉션을 만들어야 하는 이유를 이해한다.
2. 수많은 에러 클래스를 무분별하게 만들지 않고, 상속 구조를 활용해 예외를 계층화하여 설계한다.
3. `@RestControllerAdvice`를 활용해 Spring 전역에서 발생하는 예외를 한곳으로 모으고, 이를 **일관된 응답 구조(`ErrorResponseDto`)로 변환하는 메커니즘**을 파악한다.

## Custom Exception

애플리케이션의 특정 오류 상황을 명확한 의미로 표현하기 위해 개발자가 직접 정의한 예외 클래스

### "명확한 의미"가 중요한 이유: 기술적 예외 vs 비즈니스 예외

- **기술적 예외 (자바/스프링 기본 제공)**
    - `NullPointerException`, `IllegalArgumentException`, `SQLException` 등.
    - 이 예외들은 '기술적인 실패 원인'을 설명하는 데 초점이 맞춰져 있다. (값이 비어있다, 인자가 잘못됐다, SQL 문법이 틀렸다 등)
- **비즈니스 예외**
    - 우리가 개발하는 서비스는 순수 자바 기술 덩어리가 아니라 배달앱, 쇼핑몰, 커뮤니티 같은 **비즈니스 도메인**이다.
    - 비즈니스 흐름에서는 기술적 에러가 아니라 **업무 규칙에서 발생한 의도된 실패 상태**가 빈번하게 일어난다.
    - *ex) "주문하려는 상품의 재고가 없다", "이 게시글을 삭제할 권한이 없다", "존재하지 않는 회원 번호로 조회했다"*
- **Custom Exception이 주는 가치 (가독성과 도메인 지식)**
    - 위와 같은 비즈니스 예외들을 그냥 `IllegalArgumentException`으로 던지면, 코드를 읽는 다른 개발자들은 이 예외가 도대체 '어떤 비즈니스 규칙을 위반해서' 던져졌는지 파악하기 위해 코드를 다 뜯어보며 시간을 써야 한다.
    - 이럴 때 이름을 명확하게 지어준 커스텀 익셉션을 사용하면 코드 레벨에서 기획(업무 규칙)을 바로 파악할 수 있다.
        - `OutOfStockException` (재고 부족)
        - `ForbiddenAccessException` (접근 권한 없음)
        - `UserNotFoundException` (유저 없음)
    - 예외를 던지는 코드를 보는 순간 "아, 어떤 비즈니스 규칙을 위반했구나!"를 직관적으로 알 수 있게 된다.

## Custom Exception을 사용하는 이유

오류 상황을 의미 단위로 구분해 예외 처리와 응답 규칙을 일관되게 관리하고, 예외 흐름을 명확하게 통제하기 위해서이다.

### FE와 BE 간의 일관된 응답 규격 관리(협업 효율성)

- FE에서 백엔드로 API 요청(`axios`, `fetch`)을 보낼 때, 성공(200대)이면 `.then()` 절로 빠지고 실패(400, 500대)면 `.catch()` 절로 빠져서 처리한다.
- 백엔드에서 에러 응답을 대충 던지면 400 에러일 때의 JSON 모양, 401 에러일 때의 모양, 404 에러일 때의 모양이 다 제각각이 된다.
    - *ex) 어떤 건 `{ "message" : "에러" }`, 어떤 건 `{ "error_code" : 123 }`*
- 응답 규격이 다르면 FE는 에러 메시지를 화면에 띄우기 위해 에러 타입마다 복잡한 조건문(`if`)을 가동해야 한다.
- 하지만 커스텀 익셉션과 전역 예외 처리기를 도입하면 에러가 나갈 때 **의미별로 분류해서 완전히 통일된 하나의 응답 구조(규격)로 출력**할 수 있으므로, FE 쪽에서도 공통 에러 핸들링 코드를 짜기가 매우 편하고 좋아진다.

### 예외 흐름을 명확하게 통제(정상 흐름과 오류 흐름의 분리)

**나쁜 예시 : 실패 상황에 null을 반환하는 방식**

```java
// 서비스 계층
public OrderResponse getOrder(Long orderId) {
    Order order = orderRepository.findById(orderId).orElse(null);

    // 객체가 비어있는지 하나씩 검사해서 null을 반환
    if (order == null) {
        return null;
    }
    return new OrderResponse(order);
}
```

```java
// 컨트롤러 계층
@GetMapping("/{orderId}")
public ResponseEntity<?> getOrder(@PathVariable Long orderId) {
    OrderResponse result = orderService.getOrder(orderId);
    
    // 상위 컨트롤러에서도 null 검사를 또 한 번 반복해야 함
    if (result == null) { 
        return ResponseEntity.status(404).body("주문이 없습니다.");
    }
    return ResponseEntity.ok(result);
}
```

**문제점:** 실패 상황을 표현하기 위해 `null`을 리턴하다 보니, 메소드를 호출한 컨트롤러 계층에서 반환받은 값이 진짜 데이터인지 `null`인지 매번 방어적인 검사 코드를 짜야 한다. 에러 처리 코드가 비즈니스 로직 뒤섞여 코드가 지저분해진다.

**좋은 예시 :  Custom Exception을 활용한 흐름 통제**

```java
// 서비스 계층
public OrderResponse getOrder(Long orderId) {
    Order order = orderRepository.findById(orderId)
            // 데이터가 없으면 고민하지 않고 예외를 바로 터트린다.
            .orElseThrow(() -> new NotFoundException("ORDER_NOT_FOUND"));

    // 이 줄까지 코드가 도달했다면 order는 존재한다는 뜻
    return new OrderResponse(order);
}
```

```java
// 컨트롤러 계층
@GetMapping("/{orderId}")
public ResponseEntity<OrderResponse> getOrder(@PathVariable Long orderId) {
    // 무조건 성공한다고 가정하고 별도의 null 체크 없이 정상 흐름만 작성
    OrderResponse result = orderService.getOrder(orderId);
    return ResponseEntity.ok(result);
}
```

**장점:** 컨트롤러나 서비스에서 일일이 지저분하게 `if (obj == null)` 같은 방어적 코드를 짤 필요가 없다. 데이터가 없으면 `NotFoundException`이 터지면서 실행이 그 즉시 중단되고, 전역 예외 처리기(Global Exception Handler)가 이를 낚아채서 처리해 준다. **정상 흐름과 오류 흐름이 깔끔하게 분리되어 훨씬 객체지향적인 코드가 완성된다.**

## Custom Exception 구현 및 사용 방법

### 단계 1: 예외 클래스 정의 (상속 구조를 통한 계층화)

예외 클래스를 상황마다 하나하나 다 새로 만드는 것은 비효율적이다. 공통 분모가 되는 부모 예외 클래스를 두고, 이를 상속받는 구조로 계층화를 진행한다.

1. **모든 비즈니스 에러의 공통 부모가 될 최상위 예외 정의**

```java
@Getter // 글로벌 핸들러가 code와 status를 꺼내서 쓸 수 있도록 Getter 추가
public class BusinessException extends RuntimeException {

    private final String code;       // FE가 에러를 식별할 문자열 코드 (예: USER_NOT_FOUND)
    private final HttpStatus status; // HTTP 상태 코드 (예: 404)

    // 생성자에서 두 가지 필수 필드를 무조건 받도록 선언
    public class BusinessException(String code, HttpStatus status) {
        super(code);
        this.code = code;
        this.status = status;
    }
}
```

1. **부모를 상속받아 구체적인 상태를 나타내는 하위 예외 정의**

```java
public class NotFoundException extends BusinessException {
    
    // 생성될 때 부모 생성자에 아예 404(HttpStatus.NOT_FOUND)를 강제로 세팅
    public NotFoundException(String code) {
        super(code, HttpStatus.NOT_FOUND);
    }
}
```

이렇게 구현하면 `NotFoundException`은 태어날 때부터 "무조건 HTTP 상태 코드는 404로 나간다"라고 역할을 굳혀버릴 수 있어서 편리하다.

### 단계 2: 오류 상황에서 throw로 예외 발생

앞서 좋은 예시에서 보았듯, 데이터가 없거나 업무 규칙을 위반하는 순간에 `throw` 문으로 예외를 터트려 상위 계층으로 던진다.

### 단계 3: 전역 예외 처리기(Global Exception Handler)에서 응답 매핑

서비스 계층에서 던진 예외는 스프링의 대표적인 횡단 관심사(AOP) 장치인 `@RestControllerAdvice` 클래스에서 모아서 처리한다. 이 어노테이션이 붙은 클래스는 애플리케이션 내의 모든 컨트롤러에서 터져 나오는 예외를 감시하고 낚아채는 역할을 한다.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // "BusinessException 또는 그 자식 예외(NotFoundException 등)가 터지면 무조건 내가 잡아서 처리할게"
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponseDto> handleBusiness(BusinessException exception) {
        
        // 예외 객체가 들고 온 status 필드값으로 HTTP 응답 상태 코드를 결정하고,
        // code 필드값은 공통 응답 바디(ErrorResponseDto)에 예쁘게 포장해서 던져준다.
        return ResponseEntity
                .status(exception.getStatus()) // Getter로 status 추출
                .body(ErrorResponseDto.of(exception.getCode())); // Getter로 code 추출 후 DTO 변환
    }
}
```

`@ExceptionHandler`는 특정 예외를 낚아채겠다는 선언이다. 여기에도 **다형성**이 적용되기 때문에, `NotFoundException`이 발생하더라도 부모 타입이 `BusinessException`이므로 이 메소드가 한꺼번에 낚아채서 일관되게 처리할 수 있다.

## 정리

- 자바나 스프링이 제공하는 범용적이 기술 예외 대신, 비즈니스 도메인 언어로 이름을 붙인 커스텀 익셉션을 만들고 상속 계층을 구성한다.
- 서비스 로직 진행 중 실패 상황이 오면 `null` 반환 대신 `throw로` 예외를 던져 정상 흐름과 오류 흐름을 명확히 분리한다.
- 던져진 예외는 전역 예외 처리기(`@RestControllerAdvice`)에서 다형성을 이용해 낚아챈 후, **일관된 규격의 `ResponseEntity<ErrorResponseDto>`로 변환하여 FE에 반환**한다.
