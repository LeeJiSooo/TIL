## 가져가야 할 핵심

- 팩토리는 객체를 만드는 책임을 그 객체를 쓰는 코드에서 떼어내는 패턴
- 객체 new로 만드는 코드가 많이 있으면, 생성 방식이 바뀔 때마다 그 자리를 다 고쳐야하는데 팩토리는 이것을 해결한다.
- 책임 - 팩토리는 결제를 처리한다와 같은 원래 일과 결제 객체를 어떻게 만든다 같은 생성 책임을 분리한다.

설명할 수 있어야 하는 것

- 팩토리가 어떤 문제를 해결하는 패턴인지
- 객체를 직접 new로 만들면 왜 변경에 약해지는지
- 스프링부트에는 팩토리 역할이 어떻게 녹아들어있는지

## 디자인 패턴

- 개발자들이 실제 소프트웨어를 설계하고 구현하면서 수없이 부딪힌 반복적인 설계 문제들을 해결하기 위해 정립된 ‘검증된 해결 구조(설계 템플릿)
- 코드는 시간이 지나면 고치기 어려워진다.
    
    요구사항이 추가되고 외부api가 연동되면서 복잡성이 증가하기 때문
    
    “변화가 자주 일어나는 부분”을 기준으로 책임을 쪼개고 분리하여 변경 지점을 한 곳으로 수렴시키는 도구이다.
    
    → 객체지향 원칙(SOLID)와 스프링의 IoC/DI와도 맞닿아 있는다.
    

## 팩토리 패턴이 해결하는 문제

팩토리가 다루는 단 하나의 문제는 ‘객체를 만드는 일(생성 책임)’이다.

팩토리를 쓰지 않은 코드(생성과 비즈니스 로직의 결합)

```java
public class PaymentService {
    public void pay(String type) {
        Payment payment;

        // 문제점 1: 결제 방식에 따라 어떤 구현체를 만들지 호출부가 직접 분기문(if-else)을 작성
        if ("CARD".equals(type)) {
            // 문제점 2: 구체적인 구현체(CardPayment)를 직접 생성하며, 재료가 되는 의존 객체까지 직접 다 알고 조립함
            payment = new CardPayment(cardClient, feePolicy);
        } else if ("BANK".equals(type)) {
            payment = new BankPayment(bankClient, feePolicy);
        } else {
            throw new IllegalArgumentException();
        }

        // 본래 책임: 결제 실행
        payment.pay();
    }
}
```

→ PaymentService는 ‘결제를 처리하는 비즈니스 로직’만 신경 써야 한다.  하지만 지금은 어떤 결제 구현 클래스들이 존재하는지, 개별 객체를 만들려면 어떤 부품이 필요한지, 조건 분기는 어떻게 해야 하는 지까지 생성에 관한 것을 다 알고 있다.

→ 기획이 바뀌어 새로운 결제 수단이 추가되거나, CardPayment생성자 파라미터(부품)가 하나만 늘어나도, new코드가 있는 모든 서비스 레이어의 소스코드를 찾아서 일일이 고쳐야 한다. 

## 팩토리 패턴의 도입 : 생성 책임의 격리

객체를 만드는 전용 공장 클래스(Factory)를 두어 생성에 대한 모든 것을 격리시키는 방식이다.

**생성 책임을 전담하는 팩토리 클래스 추출**

```java
public class PaymentFactory {

    // 오직 객체 생성 및 조립만 담당하는 전용 공장
    public Payment create(String type) {
        if ("CARD".equals(type)) {
            return new CardPayment(cardClient, feePolicy); // 의존 조립 포함
        }
        if ("BANK".equals(type)) {
            return new BankPayment(bankClient, feePolicy);
        }
        throw new IllegalArgumentException();
    }
}
```

**팩토리를 사용하는 서비스 코드(클라이언트)**

```java
public class PaymentService {

    private final PaymentFactory paymentFactory; // 이제 서비스는 구체 클래스를 모르고 오직 팩토리에만 의존함

    public PaymentService(PaymentFactory paymentFactory) {
        this.paymentFactory = paymentFactory;
    }

    public void pay(String type) {
        // 구체 클래스를 완전히 모른 채, 팩토리에게 필요한 객체만 달라고 요청
        Payment payment = paymentFactory.create(type);

        // 오직 본래 책임인 결제 로직 실행에만 집중
        payment.pay();
    }
}
```

→ 이제 PaymentService는 CardPayment나 BankPayment같은 구체 클래스 이름조차 모른다. 객체를 만들 때 무슨 재료가 필요한지, 분기 처리는 어떻게 하는지 신경 쓰지 않는다. 오직 “결제 객체가 필요하다”는 사실 하나만 표현한다.

## 사용 이유

1. 변경 지점의 수럼(유지보수 용이)
    
    구현체가 교체되거나, 생성 규칙이 바뀌거나, 객체 생성 직전에 가동되는 초기화 로직이 추가되더라도 오직 팩토리 내부 코드 한 곳만 고치면 끝난다.
    
2. 의존 관계 조립의 은닉 
    
    객체 하나를 만드는 데 필요한 부품이 5~6개로 많아지면, 호출부 코드가 지저분해지고 결합도가 꼬인다. 
    
    이를 팩토리가 감춰주기 때문에 호출부 코드가 명료하고 깔끔해진다.
    
3. 테스트 신뢰성 향상
    
    호출부가 객체가 직접 new를 하지 않기 때문에, 단위 테스트를 할 때 실제 팩토리 대신 테스트용 팩토리를 갈아 끼우거나 생성 결과를 목객체로 마음대로 조정하기 쉬워진다.
    

## 스프링부트와 팩토리의 관계

**그럼 앞으로 스프링에서 새로운 도메인 객체를 만들 때마다 무조건 팩토리클래스를 다 생성해야되는가?**

→ **아니다.**

→ **스프링 컨테이너 자체가 이미 거대한 거대한 팩토리다.**

스프링 프레임워크를 쓰는 순간, 우리가 수동으로 팩토리를 안 만들어도 된다.

왜냐하면 스프링 컨테이너 자체가 전역 애플리케이션의 객체 생성 및 의존성 조립을 전담하는 가장 거대한 팩토리 역할을 수행해 주고 있기 때문이다.

```java
@Service
public class OrderService {
    private final PaymentClient paymentClient;

    // 우리가 직접 new로 조립하지 않아도, 거대 팩토리(스프링)가 알아서 다 만들어진 부품을 주입(DI)해 줌
    public OrderService(PaymentClient paymentClient) {
        this.paymentClient = paymentClient;
    }
}
```

우리가 @Component로 대상을 마킹하거나, @Configuration설정 클래스 안에서 @Bean으로 생성 규칙을 정의해 두기만 하면, 실제 객체를 찍어내고 생명주기를 관리하는 팩토리는 스프링 컨테이너가 알아서 처리해준다.

```java
// 구현 클래스를 컨테이너에 등록해 생성 대상으로 지정
@Component
public class CardPaymentClient implements PaymentClient {
    ...
}
```

```java
// 객체 생성 규칙을 설정 클래스에 선언적으로 정의
@Configuration
public class PaymentConfig {

    // 어떤 타입을 어떤 구현체로 만들지 컨테이너에 알려 주는 생성 규칙
    @Bean
    public PaymentClient paymentClient() {
        // 실제 생성은 여기서 정의하되 관리 책임은 컨테이너로 위임
        return new CardPaymentClient();
    }
}
```

## 스프링 환경에서 팩토리 클래스를 직접 짜야 할 때

스프링이 대부분 생성을 알아서 해준다고 했지만 모든 생성 규칙을 설정만으로 깔끔하게 표현하기 어려운 경우가 있다.

→ 조건에 따라 어떤 구현체를 골라야할지 판단이 복잡하거나 객체를 만드는 과정에 도메인 규칙이 끼어들거나 여러 부품을 조립해서 하나의 객체로 만들어야하는 경우

→ 이럴때는 팩토리 클래스를 직접 만들어서 어떤 객체를 고르고 어떻게 조합하는가라는 책임을 명시적으로 코드에 나타낸다.

```java
// 선택 책임만 담당하는 Factory로 생성 자체는 컨테이너가 처리
@Component
public class PaymentFactory {

    // new로 만들지 않고 컨테이너가 만든 Bean을 주입받아 보관
    private final CardPayment cardPayment;
    private final BankPayment bankPayment;

    // 이미 생성된 결제 구현체들을 생성자로 주입
    public PaymentFactory(CardPayment cardPayment, BankPayment bankPayment) {
        this.cardPayment = cardPayment;
        this.bankPayment = bankPayment;
    }

    public Payment getPayment(String type) {

        // 주입받은 Bean 중에서 타입에 맞는 객체를 선택
        if ("CARD".equals(type)) {
            // 직접 생성이 아니라 이미 만들어진 객체를 반환
            return cardPayment;
        }
        if ("BANK".equals(type)) {
            // 직접 생성이 아니라 이미 만들어진 객체를 반환
            return bankPayment;
        }

        // 정의되지 않은 타입에 대한 예외 처리
        throw new IllegalArgumentException();
    }
}
```

→ 동일하게 new는 안하고 있다. 객체 생성하는 일은 여전히 스프링이 한다.

→ 그럼 이 팩토리가 하는 일은 이미 만들어진 Bean 중에서 어떤걸 골라서 줄까하고 결정하는 일을 한다.

→ 프레임워크를 쓰면 생성 자체는 컨테이너에게 맡기고 선택이라는 책임만 가져갈 수 있다.

```java
@Service
public class PaymentService {

    // 구현체 선택 책임을 넘긴 Factory에만 의존
    private final PaymentFactory paymentFactory;

    public PaymentService(PaymentFactory paymentFactory) {
        // 컨테이너가 주입해 주는 Factory를 전달받음
        this.paymentFactory = paymentFactory;
    }

    public void pay(String type) {
        // 어떤 구현체인지 모른 채 필요한 결제 객체만 요청
        Payment payment = paymentFactory.getPayment(type);
        
        Order order = new Order();
        // 주문 안에는 배송 객체도 있고 결제 객체도 있고 쿠폰 사용에 order_coupon도 있습니다
        // JPA 생각하면 Order를 중심으로 다른 객체들 연관관계로 가지고 있을 수 있어요
        // 유저 입장에서는 주문 하고 주문을 생성한건데 우리 입장에서는 Order 객체 하나 만들고 그 안에 다른 객체들을 set해줘야하는 경우가 생깁니다.
        order.setPayment
        order.setOrderCoupon
        // -> 객체를 서비스 레벨에서 조립해서 만드는게 됩니다. 그럼 서비스안에서 객체를 조립하고 생성하는 책임이 생기는거고 생성 책임을 분리해야하는가?라는 신호가 될 수 있다.
        

        // 전달받은 객체로 결제 실행
        payment.pay();
    }
}
```

JPA를 사용하다 보면, 주문을 생성할 때 내부 연관관계 객체(결제, 쿠폰, 배송 등)들을 서비스 레이어 안에서 일일이 세팅하고 조립하는 코드가 길어지기 쉽다.

서비스 코드 내부에 “객체 간 복잡한 조립 및 생성 책임”이 섞이기 시작하면 비즈니스 로직 레이어가 오염되고 있다는 신호이다.  → 복잡한 도메인 조립 전담 팩토리를 도입해서 격리해 주어야 한다.

Q. 스프링 부트 프레임워크를 활용한 프로젝트를 진행하면서 팩토리 패턴을 직접 설계해서 써본 경험이 있냐 ? 

경험이 없다면..

현재까지 진행한 프로젝트에서는 요구사항에 다른 객체 생성 규칙이 비교적 단순했기 때문에, 스프링 컨테이너가 제공하는 기본 IoC/DI 기능과 @Configuration/@Bean 설정을 통한 선언적 방식으로 충분히 생성 책임을 격리할 수 있었다.

굳이 조건 분기가 없는 상황에서 커스텀 팩토리 클래스를 직접 만들면 불필요한 클래스 개수만 늘어나 가독성을 해친다고 판단했다.

다만, 향후 런타임 조건에 따른 도메인 객체 조립 로직이 복잡해진다면, 팩토리 패턴을 도입해 서비스 계층을 보호할 계획이다.

## 정리

- 팩토리가 해결하는 문제는 객체를 만드는 규칙이 코드 여기저기에 흩어지는 상황이다.
- new가 흩어지면 생성 규칙이 바뀔때마다 모든 호출부를 고쳐야하고 과정에서 빠뜨리면 문제가 생긴다.
- 스프링 프레임워크 자체의 기본 뼈대인 IoC컨테이너가 전역 애플리케이션의 대형 팩토리 역할을 대신해 주고 있다.
- 무조건 패턴을 사용해 클래스를 늘리지 말고, 단순 생성은 스프링 컨테이너의 DI를 사용하고, 도메인 조건 분기가 얽히고 복잡한 조립 규칙이 필요해지는 시점에 팩토리 패턴을 사용해야 한다.
