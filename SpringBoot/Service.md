## Service 계층의 정의와 핵심 역할

- Controller가 웹 요청을 받아 JSON 객체(DTO)로 변환한 후 들어오게 되는 계층으로 백엔드 개발자가 구현하면서 가장 많은 시간을 쓰는 핵심 공간이다.
- 서비스 계층은 단순히 비즈니스 로직을 모아둔 집합소가 아니다. 여러 하위 작업(DB조회, 검증, 외부 API 호출 등)을 쪼개고 조합하여 하나의 의미 있는 비즈니스 흐름을 만들어내는 역할을 한다.

**배달 앱의 “주문 생성”비즈니스 흐름 예시**

프론트엔드에서는 단순하게 `POST /api/orders` 요청 하나만 보낸 것이지만, 백엔드의 **Service 계층** 내부에서는 다음과 같은 수많은 개별 작업들이 유기적으로 처리된다.

1. 사용자가 정상 상태인지, 정지된 계정은 아닌지 회원 규칙 검증
2. 주문한 식당이 현재 영업 중인지 확인
3. 주문한 메뉴의 재고가 남아 있는지 DB 조회 및 검증
4. VIP 회원 여부를 확인하여 할인 쿠폰을 적용하고 최종 결제 금액 계산
5. DB 주문 테이블에 새로운 주문 정보 `INSERT`
6. 외부 결제사(PG) API를 호출하여 결제 승인 요청
7. 결제 완료 후 사용자 앱으로 알림 메시지 발송

Service 계층은 이 수많은 작업들을 **어떤 순서로 처리할지, 어떤 조건일 때 예외(Exception)를 발생시킬지, 모든 과정이 끝났을 때 어떤 결과를 반환할지** 결정한다.

## 웹 기술과 비즈니스 로직의 격리가 주는 이점

#### 비즈니스 진입점의 공통화(코드 파편화 방지)

- 일반적인 주문은 사용자가 모바일 앱(Controller)을 통해 요청하지만, 회사가 성장하여 **"매일 저녁 6시에 정기 구독 유저의 주문을 자동으로 생성하는 배치(Batch) 스케줄러"** 기능이 추가되었다고 가정해 보자
- 배치는 HTTP 요청을 거치지 않고 서버 내부 스케줄러에 의해 자동으로 돌아간다. 이때 Controller가 아닌 `OrderService`의 주문 생성 메서드를 그대로 호출(비즈니스 진입점 호출)하면 된다.
- 만약 이 개념을 무시하고 닥치는 대로 코딩하면, Controller 안에도 주문 로직이 있고 스케줄러 안에도 비슷한 주문 로직이 들어가는 **스파게티 코드**가 된다. 추후 기획팀에서 "VIP 할인율을 4%에서 5%로 올립시다"라고 했을 때, 모든 코드를 찾아 고쳐야 하며 장애 발생 확률이 높아진다. (예: 모바일 앱 주문은 15% 할인인데, 정기구독 웹 주문은 10% 할인이 적용되는 비정상 상황 발생)

#### 테스트 용이성

- 할인율 계산 같은 핵심 로직을 테스트하기 위해 **단위 테스트(Unit Test)** 코드를 작성할 때, 로직이 Controller에 있으면 가짜 HTTP 요청을 생성하는 `MockMvc` 등을 써야 하므로 테스트가 무겁고, 느리고, 복잡해진다.
- 반면 로직이 웹 기술이 섞이지 않은 `OrderService` 안에 순수 자바 코드로 분리되어 있으면, 단순히 서비스 객체를 생성하여 자바 레벨에서 메서드의 비즈니스 로직만 **매우 빠르고 가볍게 검증**할 수 있다.

## 관심사의 분리

[나쁜 예시] Controller가 너무 많은 책임을 가진 상태

```jsx
// [나쁜 예시: Controller가 너무 많은 책임을 가진 상태]
@RestController
@RequestMapping("/orders")
@RequiredArgsConstructor
public class OrderController {

    private final OrderRepository orderRepository;

    @PostMapping
    public Long createOrder(@RequestBody CreateOrderRequest request) {
				
				// 1. 수량 체크(비즈니스 검증)
        if (request.getQuantity() <= 0) {
            throw new IllegalArgumentException("quantity must be positive");
        }
        
        // 2.총액 계산
        int totalPrice = request.getPrice() * request.getQuantity();

				// 3. 도메인 객체 조립
        Order order = new Order(
                request.getProductId(),
                request.getQuantity(),
                totalPrice
        );

				// 4. 데이터베이스에 저장
        orderRepository.save(order);

				// 5. 응답 반환
        return order.getId();
    }
}
```

하나의 메서드가 HTTP 요청 수신, 비즈니스 검증, 연산, 엔티티 조립, DB 저장까지 모두 처리한다. 여기에 VIP할인, 포인트 적립 등이 추가되면 코드가 수백 줄로 늘어나 고칠 수 없는 유지보수의 무덤이 된다.

[좋은 예시] Service 계층으로 도려내어 분리한 상태

1) 분리된 OrderService (순수 비즈니스 로직)

```jsx
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;

    // HTTP 관련 코드가 단 한 줄도 없는 순수 자바 영역
    public Long create(CreateOrderRequest request) {
        // 수량 검증
        if (request.getQuantity() <= 0) {
            throw new IllegalArgumentException("quantity must be positive");
        }
					
        // 가격 계산
        int totalPrice = request.getPrice() * request.getQuantity();

        // 객체 생성
        Order order = new Order(
                request.getProductId(),
                request.getQuantity(),
                totalPrice
        );

        // 저장
        orderRepository.save(order);

        return order.getId();
    }
}
```

@RestController, @PostMapping 같은 웹 기술 관련 단어가 전혀 없으며, 오직 도메인 지식(주문, 수량, 가격)과 관련된 코드만 명확하게 남아있다.

2) 가벼워진 OrderController (웹 계층)

```jsx
@RestController
@RequestMapping("/orders")
@RequiredArgsConstructor
public class OrderController {

    // Repository를 직접 보지 않고 Service에 의존함
    private final OrderService orderService;

    @PostMapping
    public Long createOrder(@RequestBody CreateOrderRequest request) {
        // 클라이언트 요청을 받아 그대로 Service에 토스 (위임)
        return orderService.create(request);
    }
}
```

- 컨트롤러는 계산이 어떻게 되는지, DB에 어떻게 저장이 되는지 알 필요가 없다.
- 관심사의 분리 결과 : API설계 스펙(URL, 파라미터 구조 등)이 바뀌면 Controller만 고치면 되고, 주문 할인율이나 비즈니스 규칙이 바뀌면 Service만 고치면 된다. 서로의 변경이 영향을 주지 않는다.

## 트랜잭션(@Transactional)경계 설정 및 내부 원리

#### 트랜잭션(Transaction)이란?

- 데이터베이스의 상태를 다룰 때 적용하는 논리적인 작업 단위이다.
- 전부 성공(Commit)하거나 전부 실패(Rollback)하여 원래다로 되돌리는 원자성을 보장하여 데이터의 무결성과 정합성을 지켜준다.

**정합성이 깨지는 예시(PostService 게시글 작성)**

1. 유저의 총 게시글 카운트를 1 증가시킴(UPDATE)
2. 게시글 테이블에 새로운 글 정보를 삽입(INSERT)

만약 1번은 성공했는데 2번 과정에서 DB에러가 발생해 실패한다면?

→ 글은 안 써졌는데 사용자의 게시글 수만 올라가는 데이터 정합성 파괴가 일어난다.

→ 이를 방지하기 위해 메서드에 @Transactional을 선언한다. 메서드가 시작될 때 DB트랜잭션을 열고, 정상적으로 종료되면 커밋하며, 예외가 잡히면 전부 롤백한다.

#### 왜 트랜잭션의 경계는 Service계층이어야 하는가?

- 비즈니스 로직의 시작과 끝이 곧 트랜잭션의 시작과 끝이어야 데이터의 무결성이 지켜진다.
- 비즈니스 로직의 흐름을 조율하고 여러 Ropository(DB접근)를 하나의 논리적 단위로 묶어내는 최초의 진입점이 바로 Service계층의 메서드이기 때문에 트랜잭션 경계는 반드시 Service계층이 되어야한다.

#### @Transactional(readOnly = true) 옵션의 성능 최적화 원리

데이터를 수정·등록하지 않고 **오직 읽기(조회)만 수행할 때** 사용하는 옵션이다. 이 옵션을 붙이면 내부적으로 다음과 같은 이점이 생긴다.

1. **더티 체킹(Dirty Checking) 생략을 통한 메모리 절약:** JPA는 기본적으로 트랜잭션 내에서 조회한 엔티티의 변경을 감지하기 위해, 최초 조회 상태의 복사본인 스냅샷(Snapshot)을 메모리에 보관한다. 하지만 `readOnly = true`를 명시하면 스프링과 JPA는 읽기 전용임을 인지하고 **스냅샷을 보관하지 않는다.** 따라서 메모리 사용량이 줄어든다.
2. **불필요한 플러시(Flush) 생략:** 트랜잭션이 끝날 때 변경 사항을 DB에 반영하려는 flush 과정이 생략되므로 CPU 자원을 아낄 수 있다.
3. **DB 레벨의 최적화:** 데이터베이스 제품(MySQL 등)에 따라 읽기 전용 트랜잭션은 커넥션을 분리하거나 락(Lock) 관리를 최소화하여 조회 성능을 높여준다.

```jsx
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;

    @Transactional // 쓰기/수정이 일어나므로 일반 트랜잭션
    public UserResponseDto createUser(UserRequestDto request) {
        User user = new User(request.getEmail(), request.getPassword(), request.getNickname());
        User savedUser = userRepository.save(user);
        return new UserResponseDto(savedUser);
    }

    @Transactional(readOnly = true) // 단순 조회이므로 읽기 전용 최적화 적용
    public UserResponseDto getUser(Long userId) {
        User user = userRepository.findById(userId)
                .orElseThrow(() -> new IllegalArgumentException("User not found"));
        return new UserResponseDto(user);
    }
}
```

## 질문

**Q.Controller에 비즈니스 로직을 다 작성해도 프로그램은 돌아가는데, 굳이 Service 계층을 분리해야 하는 이유는 무엇인가요?**

가장 큰 이유는 관심사의 분리를 통해 유지보수성을 높이리 위함이다. 웹 요청을 처리하는 계층과 순수 비즈니스 규칙을 처리하는 계층을 분리해 두면, 프론트엔드 요구나 API스펙이 바뀌더라도 비즈니스 로직에는 영향을 주지 않는다. 

또한 웹 기술이 섞이지 않은 순수 자바 코드로 로직이 격리되어 있어 코드의 재사용성(배치, 스케줄러 등에서 호출)이 극대화되고, 복잡한 Mock환경 없이 가볍고 빠르게 단위테스트를 수행할 수 있다는 장점이 있다.

**Q. 스프링 프레임워크에서 트랜잭션 처리는 어느 계층에서 해야 하며, 그 이유는 무엇인가요?**

Service계층에서 해야 한다. 트랜잭션은 데이터베이스의 정합성을 지키기 위해 여러 작업을 하나의 논리적인 단위로 묶는 경계선이다. 

비즈니스 시나리오 안에서 다양한 Repository의 메서드들을 호출하고 조율하는 최초의 진입점이 Service계층의 메서드이기 때문에, 해당 메서드의 시작과 끝이 트랜잭션의 시작과 끝이 되도록 설정하는 것이 올바른 설계이다.
