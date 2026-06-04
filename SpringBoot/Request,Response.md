네트워크를 타고 날아오는 물리적인 HTTP메시지 자체를 개발자가 직접 한줄씩 파싱하는 것이 아니다. 스프링 프레임워크 레벨에서 이를 자동으로 파싱해여 자바 객체로 변환해 주기 때문에 개발자는 비즈니스 로직에만 집중할 수 있다.

## HTTP Request/Response의 물리적 구조

HTTP 메시지는 크기 전송 방향과 관계없이 크게 세 개지 영역으로 구성되어 있다.

<img width="820" height="303" alt="image" src="https://github.com/user-attachments/assets/098c5018-df01-4c37-8aa8-32d1f27815a2" />

#### StartLine(시작 줄)

- 요청(Request Line) : 첫 줄에는 어떤 행위를 하는지(HTTP Method), 어떤 경로에 요청하는지(URL/URI), 그리고 어떤 규격을 사용하는지(HTTP프로토콜 버전)를 나타낸다.
- 응답(Status Line) : 프로토콜 버전과 함께 요청의 처리 결과를 나타내는 상태코드와 상태 메시지가 들어간다.

#### Header(헤더) : 메타데이터 영역

- 메시지 본문(Body)에 대한 설명이나 통신에 필요한 부가 정보(메타데이터)가 들어간다.
- User-Agent : 요청을 보낸 클라이언트 종류(크롬, 사파리, curl 등)
- Content-Type : 보내는 Body데이터의 형식(예 : 대다수의 현대 API통신에서는 application/json)
- 기타 정보 : 데이터의 전체 길이, 캐시의 유효기간 등
- 인증 정보 : JWT토큰 같은 인증 정보도 헤더의 Authorization영역에 실려서 전송된다.

#### Body(바디) : 실제 데이터 영역

- 서버에서 처리하거나, 프론트엔드에서 서버로 보낼 실제 값이 들어오는 공간이다.
- 주로 JSON형식의 데이터가 이 바디를 통해 오고 간다.

## 스프링 어노테이션들

#### @PathVariable(경로 변수)

- URL 경로에 붙어있는 값을 자바 메서드의 변수로 추출할 때 사용한다.
- URL매핑에 정의된 {orderId} 부분과 파라미터의 변수명(orderId)이 일치하면 스프링이 자동으로 값을 매핑해준다.
- 클라이언트가 /orders/123 으로 요청하면, 내부의 ArgumentResolver가 경로를 분석하여 “123”이라는 문자열을 Long타입의 숫자 변수로 자동 변환해준다.

```jsx
@GetMapping("/orders/{orderId}")
public OrderResponse getOrder(@PathVariable Long orderId) {
    return new OrderResponse(orderId);
}
```

#### @RequestParam(쿼리 파리미터 / 쿼리 스트링)

- URL뒤에 물음표를 붙이고 적어 넣는 Key-Value형태의 쿼리 스트링 값을 뽑아올 때 사용한다.
- 특정 조건에 따른 필터링, 정렬, 검색어 등을 전달할 때 유용하다.
- required = false 설정을 주면 해당 값이 필수 조건이 아니게 된다. “조건을 안 보내면 전체를 조회하고, 보내면 그 상태로 필터링해서 보여주겠다”는 처리가 가능하다.
- GET/orders?status=READY로 요청이 들어오면, status변수에 “READY”라는 문자열이 담긴다.

```jsx
@GetMapping("/orders")
public List<OrderResponse> getOrders(
        @RequestParam(required = false) String status
) {
    return List.of();
}
```

#### @RequestHeader(요청 헤더)

- HTTP 헤더에 담긴 값을 메서드 파라미터로 직접 전달받기 위해 사용한다.
- 인증 토큰(JWT), 사용자 정보, 혹은 클라이언트가 보낸 특수한 메타데이터를 컨트롤러에서 꺼내 써야 할 때 적용한다.

```jsx
@GetMapping("/orders/{orderId}")
public OrderResponse getOrder(
        @PathVariable Long orderId,
        @RequestHeader("Authorization") String token
) {
    // 헤더에서 꺼낸 Authorization 토큰을 활용한 로직 수행 가능
    return new OrderResponse(orderId);
}
```

#### @RequestBody(요청 바디)

- HTTP 본문(Body)에 JSON형태로 담겨 들어오는 큰 데이터를 자바 객체(DTO)로 변환하여 받을 때 사용한다.
- 회원가입이나 주문 요청처럼 보내야 하는 데이터가 많고 복잡할 때는 URL에 다 담을 수 없으므로 Body에 JSON포맷 {”productId : 100, “quantity” : 2}으로 담아 보낸다.
- 동작원리
    1. 컨트롤러 파라미터 앞에 @RequestBody를 발견하면 스프링의 MessageConverter가 개입한다.
    2. 클라이언트가 보낸 데이터가 JSON이네? 그리고 자바 메서드는 CreateOrderRequest객체를 원하네?를 확인한다.
    3. 내부적으로 Jackson라이브러리가 리플렉션 기술을 이용해 JSON문자열을 파싱하고, 일치하는 필드를 찾아 매칭시킨 후 자바 객체를 생성해 전달한다.
    
     
    
    ```jsx
    @GetMapping("/orders/{orderId}")
    public OrderResponse getOrder(
            @PathVariable Long orderId,
            @RequestHeader("Authorization") String token
    ) {
        // 헤더에서 꺼낸 Authorization 토큰을 활용한 로직 수행 가능
        return new OrderResponse(orderId);
    }
    ```
    
    ```jsx
    @Getter // 필드에 대한 getter 메서드 생성
    @NoArgsConstructor // 기본 생성자 생성
    public class CreateOrderRequest {
        private Long productId;
        private int quantity;
    }
    ```
    

Jackson라이브러리가 JSON을 자바 객체로 바꿀 때 우선 ‘텅 빈 객체’를 먼저 생성한 뒤 내부적으로 필드 값을 채워 넣는 과정을 거친다. 따라서 @NoArgsConstructor(기본 생성자)와 객체 조회를 위한 @Getter가 누락되면 변환 과정에서 400 또는 500에러가 발생하므로 절대 빼먹으면 안된다.

## Path Variable vs Query Parameter

| **구분** | **Path Variable (/users/10)** | **Query Parameter (/orders?status=READY)** |
| --- | --- | --- |
| **핵심 목적** | **리소스의 식별 (Identification)** | **리소스의 필터링 / 정렬 / 조건 (Filtering)** |
| **의미적 기준** | 그 값이 없으면 타겟팅 자체가 성립되지 않는 **고유한 자원**을 가리킬 때 | 대상(전체 목록)은 정해져 있으나 옵션을 주어 **걸러서 보고 싶을 때** |
| **예시 상황** | "10번 유저를 보여줘" | "전체 주문 중 READY 상태인 것만 골라줘" |
| **생략 가능 여부** | **생략 불가** (생략 시 아예 다른 URL(`/users`)이 되어 의미가 완전히 변함) | **생략 가능** (조건이 없으면 전체 데이터를 보여주는 식으로 요청 자체가 성립됨) |

게시판에서 게시글 목록을 보여줄 때 1번페이지를 볼지, 한 번에 10개씩 볼지 20개씩 볼지 결정하는 페이지네이션도 리소스를 필터링하고 제한하는 조건에 해당하므로 쿼리 파라미터를 사용한다.

## 정리

- HTTP는 시작 줄, 헤더, 바디의 물리적 구조를 지니며, 이 텍스트 메시지를 파싱해 자바 객체로 변환해 주는 복잡한 작업은 스프링 프레임워크가 전담한다.
- 리소스 그 자체를 가리키고 식별할 때는 고정된 경로인 PathVariable을, 리소스를 정렬하거나 필터링하는 조건에는 Query Parameter를 선택한다.
