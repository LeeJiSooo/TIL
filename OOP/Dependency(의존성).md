## 의존성(Dependency)의 본질

- 객체지향 세계에서 객체들은 각자 역할과 책임을 가지고 서로 메시지를 주고받으며 협력한다.
- 협력한다는 것은 ‘누군가가 다른 누군가를 필요로 한다’는 것을 의미한다
    
    → 요리사가 요리(책임)를 하려면 칼이 필요하고, 자동차가 나아가려면 엔진이 필요하다.
    

내가 책임을 다하기 위해 다른 객체의 도움이 필요한 상태를 의존성이라고 한다.

“의존한다” = “변경의 영향을 받는다”

OderService가 PaymentService의 pay()메서드를 호출한다고 할때 결제 객체의 pay()메서드가 processPayment()로 바뀌거나 파라미터가 추가되면 이를 호출하던 OrderService의 코드도 함께 수정해야 한다. 즉, 한 객체의 변경이 다른 객체의 동작이나 수정으로 이어지는 가능성이 의존성이다.

- 의존성은 제거 대상이 아니다. 변경의 영향을 받는 게 싫다고 의존성을 아예 없애버리면 객체 간의 협력이 불가능해진다. → 객체지향 프로그래밍 자체를 포기하겠다는 뜻
- 의존성 자체가 나쁜 것이 아니다. 핵심은 의존성을 완전히 없애는 것이 아니라, 결합도를 낮추어 적절하게 통제하고 관리하는 것이다.

하나의 객체가 모든 책임을 진 구조(내부에서 new 직접 생성)

```jsx
class OrderService {
    public void placeOrder(Order order) {
        // 1. 주문 검증 -> [자신의 책임]
        if (order.getAmount() <= 0) {
            throw new IllegalArgumentException("주문 금액은 0보다 커야 합니다.");
        }

        // 2. 결제 방식의 선택과 생성하고 실행 -> [남의 책임]
        CardPayment payment = new CardPayment(); 
        payment.pay(order.getAmount());

        // 3. 저장 방식의 선택과 생성, 실행 -> [남의 책임]
        OrderRepository repository = new OrderRepository();
        repository.save(order);
    }
}
```

1. 강한 결합과 변경 취약성
    1. “카카오페이 결제를 추가해 주세요” 또는 “MySQL 대신 Redis에 저장합시다”라는 요구사항이 오면 정작 본연의 책임인 ‘주문 검증 로직’은 하나도 바뀌지 않았는데 OrderService클래스를 열어서 코드를 뜯어 고쳐야 한다. 수정할 때마다 다른 로직을 건드려 실수할 확률이 높아지고 시스템 안정성이 떨어진다.
2. 테스트 불가능
    1. placeOrder()가 잘 동작하는지 검증하는 테스트 코드를 돌릴 때마다 내부에서 new CardPayment()이 실행되어 테스트를 할 때마다 실제로 카드사 요청이 날아가 돈이 빠져나가는 말이 안되는 상황이 발생한다.

의존성을 선언하고 외부에서 주입받는 구조

```jsx
class OrderService {
    // 의존성 선언: "나는 구체적인 종류는 모르겠고, 결제처리기와 저장소가 필요하다"
    private final PaymentProcessor paymentProcessor; // 인터페이스 기반
    private final OrderRepository orderRepository;

    // 생성자를 통해 의존성을 외부로부터 주입(Injection)받음
    public OrderService(PaymentProcessor paymentProcessor, OrderRepository orderRepository) {
        this.paymentProcessor = paymentProcessor;
        this.orderRepository = orderRepository;
    }

    public void placeOrder(Order order) {
        // 1. 주문 검증 (자신의 책임에만 집중)
        if (order.getAmount() <= 0) { throw new IllegalArgumentException("..."); }

        // 2. 구현 세부사항은 모른 채, 역할에 메시지만 전송
        paymentProcessor.pay(order.getAmount());
        orderRepository.save(order);
    }
}
```

1. 유연한 확장과 결합도 낮추기(DIP원칙)
    1. OrderService 내부에는 new CardPayment() 같은 구체적인 코드가 없다. “난 결제 기능(PaymentProcessor 인터페이스)이 있는 객체 아무거나 들어오면 내 할 일만 하겠다”라고 선언해 둔 상태이다. 나중에 카카오페이로 바뀌든 네이버페이로 바뀌든 OrderService코드는 단 한 줄도 고칠 필요가 없다.
    
2. 순수한 비즈니스 로직 테스트 가능
    1. 테스트 환경에서는 진짜 결제 객체 대신 가짜 객체를 만들어서 생성자로 넣어주면 된다. “결제가 성공했다고 치고 응답해줘”라고 가짜 객체를 설정해 두면 카드사 요청 없이도 내가 작성한 주문 검증 로직만 순수하고 안전하게 테스트할 수 있다.

1. 효율적인 분업
    1. 역할과 책임이 인터페이스 단위로 쪼개져 있으므로 A개발자는 OrderService를 B개발자는 결제 모듈을, C개발자는 저장소 모듈을 각자 클래스에서 평화롭게 작업할 수 있어 충돌이 방지된다.

1. 협업의 병목
    1. 개발자들이 하나의 거대한 OrderService 클래스 파일만 붙잡고 동시다발적으로 코드를 수정해야 하므로 충돌이 빈번하게 일어난다.

## 스프링 프레임워크의 존재 이유 : IoC와 DI

그럼 누군가는 new CardPayment()를 해서 넣어줘야 하는데 누가하냐?

개발자가 직접 객체 생성 팩토리 클래스를 짜서 조립할 수도 있지만 이 생성 조립 책임마저 외부에 위임하기 위해 사용하는 것이 바로 스프링 프레임워크(Spring IoC 컨테이너)이다.

- IoC(Inversion of Control - 제어의 역전)
    - 과거에는 OrderService가 주도권을 갖고 자신이 피리요한 객체를 직접 생성했다.(제어권이 개발자/코드에 있음)
    - 스프링을 사용하면 객체의 생성, 관리, 소멸의 주도권이 스프링 컨테이너로 넘어간다.(제어권이 프레임워크로 역전됨)
- DI(Dependency Injection - 의존성 주입)
    - 개발자는 스프링과 약속된 문법(애노테이션 등)으로 선언만 해두면 스프링이 애플리케이션 실행 시점에 필요한 객체를 알아서 new로 만들어서 생성자를 통해 꽂아준다.

→ 스프링 덕분에 개발자는 오직 비즈니스 가치를 만드는 것에만 집중할 수 있다.

## 정리

- 의존성은 객체지향에서 피할 수 없는 협력의 결과물이며 나쁜 것이 아니라 어떻게 관리하느냐가 핵심이다.
- 클래스 내부에서 new 키워드로 다른 객체를 직접 생성하는 행위는 결합도를 극도로 높여 변경을 어렵게 만들고 테스트를 불가능하게 만든다.
- 이를 해결하기 위해 객체는 인터페이스를 통해 의존성을 선언하고 실제 객체의 생성과 조립 책임을 스프링(IoC/DI)같은 도구에 위임함으로써 유연하고 단단한 아키텍처를 완성할 수 있다.
