DIP : 의존 관계를 맺는 ‘구조’를 뒤집는 것

의존성 주입(DI)을 배울 때 “변경 가능성이 큰 객체를 직접 new 키워드로 생성하면 결합도가 높아지니 외부에 주입받아야 한다”라고 배웠다.

DIP는 이 의존성 주입(DI)이 더 의미 있게 작동하도록 해주는 구조적 설계 원칙이다.

## DIP(Dependency Inversion Principle, 의존 역전 원칙)

- DIP를 적용하기 전에는 상위 모듈이 하위 모둘에 의존하지만, 적용 후에는 의존성의 방향이 완전히 뒤집어진다.
- 인터페이스(추상화)를 도입함으로써 ‘변하지 않는 핵심 비즈니스 정책’이 ‘자주 변하는 세부 구현 기술’의 직접적인 변경 영향으로부터 안전하게 분리되는 원리를 파악해야 한다.

```jsx
public class OrderService {

    private final MySqlOrderRepository orderRepository = new MySqlOrderRepository();

    public void processOrder(Order order) {
        // 주문 검증, 금액 계산 등 비즈니스 로직
        orderRepository.save(order);
    }
}
```

쇼핑몰 시스템을 예로 들면 모듈의 계층은 다음과 같이 나뉜다.

- 상위 모듈 : 주문 처리(OrderService), 회원가입 정책 등 고객이 물건을 살 수 있게 만드는 핵심 비즈니스 로직
- 하위 모듈 : MySQL에 저장하기, 카드사에 결제하기 요청하기 등 세부적인 구현 및 인프라 기술

**기존의 문제점** 

보통 아무런 고민 없이 코드를 짜면 OrderService 내부에서 MySqlOrderRepository를 직접 new로 생성해야한다. 

→수명이 길고 가장 중요한 상위 모듈이 언제든 바뀔 수 있는 하위 모듈에 종속된다.

→DB를 바꾸거나 비용 문제로 사양이 변경되면, 비즈니스 정책은 바뀐 게 없는데도 핵심 로직인 OrderService를 고쳐야 하는 부담이 발생한다.

## DIP위반 사례

#### 나쁜 코드 예시 : 구체 클래스에 직접 의존하는 구조

```jsx
// 하위 모듈 (구체적인 DB 구현 기술)
public class MySqlOrderRepository {
    public void save(String orderId) {
        System.out.println("MySQL에 주문을 저장합니다: " + orderId);
    }
}

// 상위 모듈 (비즈니스 정책)
public class OrderService {

    // 문제 1 : 구체 클래스(MySqlOrderRepository) 타입을 명시함
    // 문제 2 : new 키워드로 직접 생성하여 두 클래스가 강하게 결합됨
    private MySqlOrderRepository orderRepository = new MySqlOrderRepository();

    public void placeOrder(String orderId) {
        validate(orderId);
        orderRepository.save(orderId); // MySQL 구현 기술에 직접 종속됨
        System.out.println("주문 처리를 완료했습니다.");
    }

    private void validate(String orderId) {
        System.out.println("주문 검증 로직을 수행합니다.");
    }
}
```

- 이 상태에서 DB가 PostgreSQL로 바뀌면 비즈니스 로직의 변경이 없음에도 OrderService코드를 수정해야 하므로 DIP위반이다.
- 코드를 직접 수정해야 하므로 기존 코드가 변경에 닫혀있지 못해 OCP도 함께 무너진다. → DIP가 지켜지지 않으면 OCP도 취약해진다.

#### DIP를 적용한 구조(의존성 역전) - 추상화(인터페이스)에 의존하는 구조

```jsx
// 1. 상위 모듈과 하위 모듈이 둘 다 바라볼 추상화(계약서) 정의
// 본질적으로 상위 모듈인 OrderService가 "난 이런 기능이 필요해"라고 선언한 것
public interface OrderRepository {
    void save(String orderId);
}

// 2. 하위 모듈들은 상위 모듈이 정의한 인터페이스를 구현(의존)한다.
public class MySqlOrderRepository implements OrderRepository {
    @Override
    public void save(String orderId) {
        System.out.println("MySQL에 주문을 저장합니다: " + orderId);
    }
}

public class PostgresOrderRepository implements OrderRepository {
    @Override
    public void save(String orderId) {
        System.out.println("PostgreSQL에 주문을 저장합니다: " + orderId);
    }
}

// 3. 상위 모듈은 오직 추상화에만 의존하고, 구체 구현체는 외부에서 주입(DI)받는다.
public class OrderService {

    // 구체 클래스명(MySql...)이 완전히 사라짐
    private final OrderRepository orderRepository;

    // 생성자를 통해 의존성을 외부에서 주입받음 (DI)
    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    public void placeOrder(String orderId) {
        validate(orderId);
        orderRepository.save(orderId); // 다형성에 의해 실제 주입된 기술(MySQL 또는 Postgres)이 실행됨
        System.out.println("주문 처리를 완료했습니다.");
    }

    private void validate(String orderId) {
        System.out.println("주문 검증 로직을 수행합니다.");
    }
}
```

- 양쪽 모두가 인터페이스를 바라보게 되면서 상위 모듈이 하위 모듈에 의존하지 않게 되었다.
- 이제 새로운 DB기술이 추가되어도 OrderService코드는 바뀔 필요가 없으므로 OCP와 DIP를 동시에 달성한다.

#### 왜 ‘역전’이라는 단어가 붙을까?

의존성의 화살표 흐름이 뒤집히기 때문이다.

- DIP적용 전 : OrderService(상위) → MySqlRepository(하위)
    - 상위 모듈이 하위 모듈의 변화에 직접 영향을 받음(종속 관계)
- DIP적용 후 : OrderService(상위) → OrderRepository(인터페이스) ← MySqlRepository(하위)
    - 상위 모듈은 “나는 주문을 저장할 무언가가 필요해”라는 추상적인 계약(인터페이스)만 바라본다.
    - 하위 모듈(MySQL)역시 상위 모듈이 정의해 놓은 인터페이스를 구현하기 위해 인터페이스를 향해 의존한다.
    
    → 결과적으로 하위 모듈이 상위 모듈의 흐름을 따르게 되면서 화살표의 방향이 뒤집히게 된다.
    

## 질문

#### Q. OCP와 DIP의 관계는 무엇인가요?

OCP는 설계의 ‘목표’이고, DIP는 그 목표를 이루기 위한 구조적 ‘수단’이다.

“기존 코드를 고치지 않고 확장한다(OCP)”라는 목표를 달성하기 위해, 상위 모듈과 하위 모듈이 구체가 아닌 추상화에 의존하도록 “의존성 방향을 뒤집는 구조”를 만들어야 하기 때문이다.

#### Q. DIP, DI, IoC컨테이너의 차이는 무엇인가요?

DIP(의존 역전 원칙) : 추상화에 의존해야 한다는 설계원칙

DI(의존성 주입) : DIP 원칙을 코드로 실천하기 위해 의존성을 외부에서 넣어주는 구현 수단

Spring IoC컨테이너 : 이 DI조립 과정을 개발자가 일일이 하지 않고 자동으로 해주는 프레임워크 도구

## 정리

- DIP는 상위 모듈이 하위 모듈에 의존하던 전통적인 방향을 뒤집어, 양쪽 모두가 추상화(인터페이스)에 의존하도록 만드는 원칙이다.
- 기술 환경(DB, 메일 서버 등)은 언제든 바뀔 수 있다. DIP를 통해 세부 구현 기술의 변화가 핵심 비즈니스 로직에 직접적인 영향을 주지 않도록 할 수 있다.
