## 가져가야 할 핵심

- 전략 패턴은 변하는 처리 방식을 고정된 목적에서 떼어내는 패턴이다. “결제한다는 목적은 그대로 카드냐 계좌 이체냐 하는 방식만 따로 분리한다” → 이문장이 핵심
- 전략 패턴은 if문이 끊임없이 늘어나면서 변경이 어디로 번질지 모르게 되는 문제를 막기 위해서 사용할 수 있다. → 구조적인 측면에서 해결하는 패턴
- 전략 패턴 적용하고 나면 코드 고칠 지점이 분리가 된다. 메서드에 분기를 추가하는 게 아니라 전략을 하나 더 추가하는 방식이다.
- 전략 패턴이랑 DI를 구분할 수 있어야 한다.

## 정의

- 변경 가능한 행위를 인터페이스로 분리하고, 실행 시점에 구현체를 교체할 수 있도록 설계하는 객체지향 패턴
- **행위의 분리** : 여기서 말하는 ‘행위’는 단순한 한 줄 짜리 메서드가 아니다. 특정 비즈니스 상황에서 적용되는 처리 규칙이나 전체 절차를 뜻한다.
    
    → 카드 결제와 계좌 이체는 검증 절차부터 외부 API 호출 순서까지 흐름 전체가 다르므로 각각 독립된 전략 객체로 추출함
    
- **실행 시점의 교체** : 컴파일하는 시점에 “이 기능은 무조건 이 클래스를 사용할거다”라고 하드코딩 하는 게 아니라, 프로그램이 실제로 돌아가는 순간(런타임) 유저의 선택이나 데이터 상태에 따라 동적으로 전략을 갈아끼울 수 있다.

1. **컨텍스트(Context)** : 전략을 사용하는 주체(ex. PaymentService). “결제를 처리한다”라는 고정된 목적만 알고 있다.
2. **전략 객체(Strategy)** : 실제 세부 처리 규칙을 담고 있는 객체(ex. CardPaymentStrategy). 구체적인 처리 방식은 내부에 감추고, 바깥에는 인터페이스라는 ‘동일한 형태의 계약’만 내보낸다.

→ 컨텍스트는 기능이 동작해야 한다는 것만 알 뿐, 그걸 카드로 돌릴지 계좌 이체로 돌릴지 세부 구현은 알 필요가 없다. 

→ 컨텍스트 입장에서는 어떤 전략이 들어오든 호출 방식이 똑같다. 

→ 전략 객체들은 서로의 자리를 자유롭게 바꿔서 끼울 수 있다.

```java
// 변경되는 결제 행위를 추상화한 전략 인터페이스
public interface PaymentStrategy {
    // 컨텍스트가 전략에게 기대하는 단일 동작
    void pay(int amount);
}

// 카드 결제 처리 규칙을 담은 전략 구현체
public class CardPaymentStrategy implements PaymentStrategy {
    @Override
    public void pay(int amount) {
        // 카드 방식에 해당하는 구체 처리
        System.out.println("카드 결제로 " + amount + "원 결제");
    }
}

// 계좌 이체 처리 규칙을 담은 전략 구현체
public class AccountTransferStrategy implements PaymentStrategy {
    @Override
    public void pay(int amount) {
        // 계좌 이체 방식에 해당하는 구체 처리
        System.out.println("계좌 이체로 " + amount + "원 결제");
    }
}

// 전략을 주입받아 사용하는 컨텍스트
public class PaymentService {

    // 구체 구현이 아닌 전략 인터페이스에만 의존
    private final PaymentStrategy paymentStrategy;

    // 사용할 전략을 외부에서 주입받아 보관
    public PaymentService(PaymentStrategy paymentStrategy) {
        this.paymentStrategy = paymentStrategy;
    }

    // 결제 방식 판단 없이 전략에게 처리를 위임
    public void processPayment(int amount) {
        paymentStrategy.pay(amount);
    }
}
```

## 전략 패턴이 해결하는 문제 : if-else과다

**패턴 적용 전**

```java
public void processPayment(String type, int amount) {
    if ("CARD".equals(type)) {
        validateCard(amount);         // 카드 전용 검증 추가
        cardPay(amount);
    } else if ("ACCOUNT".equals(type)) {
        validateAccount(amount);      // 계좌 전용 검증 추가
        accountTransfer(amount);
    } else if ("EASY".equals(type)) { // 기획 추가로 계속 늘어나는 분기문
        easyPay(amount);              
    }
}
```

→ 새로운 간편결제(토스, 카카오 등)가 추가될 때마다 메서드가 한없이 길어진다.

→ 어떤 변경이 다른 분기 어떤 영향을 줄지 한눈에 안 보이고, 코드 한 줄 고치려다가 다른 결제 수단이 고장 나는 위험에 노출된다.

**디자인 패턴을 사용할 때 가장 중요한 것은 “어떤 영역이 변경 빈도가 높은가”를 식별하는 것이다.**

- 변하지 않는 것(고정된 목적) : `processPayment`가 결제를 처리해야 한다는 사실
- 자주 변하는 것(구체적 방식) : 카드로 할지, 계좌로 할지, 간편결제로 할지 등의 세부 규칙

→ 기존 메서드에 조건문을 계속 쌓는 대신, 결제 수단이 늘어날 때마다 “전략 클래스를 하나 더 추가”하는 구조로 정리한다.

→ 정책이 바뀌어도 고칠 곳이 해당 전략 클래스 내부 한곳으로 격리된다 → 계방-폐쇄 원칙과 연결

**개발자는 기획자나 비즈니스 부서와의 소통을 통해 이 패턴의 필요성을 미리 감지한다.**

- 결제 도메인 : 단순히 카드/계좌 추가를 넘어, 특정 PG사와 협의해서 “우리 토스페이스페이 연동해주면 결제 시 3,000원 할인 이벤트 들어간다”같은 프로모션 비즈니스 규칙이 런타임에 유연하게 끼어들어야 할 때 전략패턴 사용
- 쿠폰 발급 도메인
    1. 유저가 버튼을 직접 눌러서 받는 정적 발급
    2. 매달 특정 등급 회원에게 일괄 지급하는 등급별 동적 발급
    3. 관리자가 명단을 액셀로 업로드해서 인플루언서들에게만 주는 타겟팅 발급

→ 목적(쿠폰 발급)은 하나인데 처리 방식(발급 조건 및 대상 선정)이 계속 늘어나는 상황

## 사용 방법

변경되는 행위를 인터페이스로 정의하고, 각 구현을 독립 클래스로 분리한 뒤 실행 시점에 주입해 선택적으로 사용하도록 구성

→ 변하는 부분을 찾고 그 부분이 따라야 할 약속을 정하고, 약속을 각각 구현으로 떼어낸 다음에 구현들을 사용할 컨텍스트를 만들고 마지막에 어떤 구현을 쓸지 실행 시점에 고른다.

#### 전략 분리 대상 식별

→ 뭐가 변하고 뭐가 변하지 않는가를 갈라야 한다.

→ 앞에 예시에서는 변하지 않는건 결제를 처리한다는 목적이고, 카드냐 계좌이체냐 간편결제냐에 따라서 규칙은 계속 달라진다.

→ 목적은 고정인데 방식이 달라지는 지점 식별

#### 전략 인터페이스 정의(공통 계약)

분리 대상을 정했으면 여러 처리 방식이 공통으로 따라야 할 계약을 인터페이스로 정의

```java
// 여러 결제 방식이 공통으로 따르는 계약 정의
public interface PaymentStrategy {
    // 카드인지 계좌인지 드러내지 않는 호출 형태 고정
    void pay(int amount);
}
```

→ amout를 받아서 pay를 한다는 형태만 약속

→ 전략을 사용하는 객체가 전략에게 어떤식으로 말을 걸지 호출 형태만 먼저 고정하는 단계

#### 전략 구현체 분리

if-else로 나뉘어져있던 걸 각각 독립된 클래스로 이동

```java
// 조건 분기 하나가 대응되는 카드 결제 전략
public class CardPaymentStrategy implements PaymentStrategy {
    @Override
    public void pay(int amount) {
        // 카드 결제 규칙만 담당하며 다른 전략과 분리
        System.out.println("카드 결제로 " + amount + "원 결제");
    }
}
```

```java
// 또 다른 조건 분기가 대응되는 계좌 이체 전략
public class AccountTransferStrategy implements PaymentStrategy {
    @Override
    public void pay(int amount) {
        // 계좌 이체 규칙만 담당하여 카드 전략에 영향 없음
        System.out.println("계좌 이체로 " + amount + "원 결제");
    }
}
```

→ 조건 분기 하나가 여기서는 구현체 하나로 대응된다.

#### 컨텍스트(전략을 사용하는 객체)구성

```java
// 결제 방식 판단을 걷어낸 컨텍스트
public class PaymentService {

    // 전략 인터페이스에만 의존하여 구체 구현을 모름
    private final PaymentStrategy paymentStrategy;

    // 사용할 전략을 외부에서 주입
    public PaymentService(PaymentStrategy paymentStrategy) {
        this.paymentStrategy = paymentStrategy;
    }

    // 전략 호출만 담당하고 방식 선택에는 관여하지 않음
    public void processPayment(int amount) {
        paymentStrategy.pay(amount);
    }
}
```

→ 인터페이스 하나에만 의존하고 주어진 전략을 호출하는 책임

→ 서비스는 결제를 처리해야한다는 사실만 알고 있다.

#### 실행 시점 전략 선택

어떤 전략을 쓸지는 실행시점에 정한다.

```java
// 사용할 전략을 실행 시점에 골라 컨텍스트에 주입
PaymentService paymentService =
        new PaymentService(new CardPaymentStrategy());

// 컨텍스트는 그대로 두고 전략만 바꿔 끼우는 호출
paymentService.processPayment(10000);
```

결제 방식을 바꾸고 싶다면 결제 서비스를 고치는 것이 아니라 주입하는 전략 객체만 바꾸면 된다.

→ new로 하지 않고 스프링에게 맡김

```java
// 카드 결제 전략을 빈으로 등록
@Component
public class CardPaymentStrategy implements PaymentStrategy {
    @Override
    public void pay(int amount) {
        System.out.println("카드 결제로 " + amount + "원 결제");
    }
}

// 계좌 이체 전략을 빈으로 등록
@Component
public class AccountTransferStrategy implements PaymentStrategy {
    @Override
    public void pay(int amount) {
        System.out.println("계좌 이체로 " + amount + "원 결제");
    }
}
```

→ 전략 구현체를 Bean으로 등록해두면 된다.

```java
// 전략 구현체를 직접 만들지 않고 주입받는 컨텍스트
@Service
public class PaymentService {

    // 컨테이너가 조립해 주는 전략에 의존
    private final PaymentStrategy paymentStrategy;

    // 어떤 구현체가 들어올지 모른 채 주입만 받음
    public PaymentService(PaymentStrategy paymentStrategy) {
        this.paymentStrategy = paymentStrategy;
    }

    // 주입된 전략에게 결제 처리를 위임
    public void processPayment(int amount) {
        paymentStrategy.pay(amount);
    }
}
```

→ 코드 직접 조립하는 것을 스프링 컨테이너가 대신 해준다.

**결국  DI랑 똑같은거 아닌가?**

→ 해결하려는 문제의 관점이 완전히 다른 기술이다.

- **전략 패턴** : “설계의 관점”. 변하는 행위 자체를 어떤 객체 구조로 떼어내고 추상화할지 다루는 방법론이다. 핵심 질문은 “이 비즈니스의 처리 규칙이 언제, 어떻게 확장되는가?”에 있다.
- **의존성 주입** : “연결의 관점”. 객체 간의 의존 관계를 객체 내부에서 직접 결정하지 않고 바깥에서 넣어주는 기술적인 메커니즘. 핵심 질문은 “객체 생성과 제어권을 어떻게 바깥으로 뺄 것인가?”에 있따.

→ 전략 패턴으로 갈아 끼울 수 있는 유연한 구조를 먼저 설계해 두고, 그 전략 객체들을 실제로 안전하게 끼워 넣는 수단으로 DI기술을 가져다 쓰는 관계이다.

- 만약 스프링 없이 new를 써서 전략을 직접 갈아 끼웠다면? → DI는 없지만 전략 패턴은 성립함
- 갈아 끼울 전략 인터페이스나 대역이 아예 없는 껍데기뿐인 단일 구현 클래스를 스프링 생성자로 받기만 했다면? → 전략 패턴은 없고 DI만 쓴 것임

## 정리

- 동일한 비즈니스 목적인데 처리 방식이 자꾸 늘어날 때, 무한 if문을 방어하고 구조적으로 행위를 추가, 확장하기 위해 사용한다.
- 분리하는 책임은 결제를 한다는 고정된 목적은 컨텍스트에 남기고, 어떻게 결제하는가 라는 바뀌는 방식은 전략 객체로 떼어낸다.
- 새로운 정책이 추가되어도 기존 서비스 로직을 건드리지 않고, 새로운 전략 클래스를 한나 더 추가하는 형태로 안전하게 변경 범위를 고정한다.
- 전략패턴은 “행위를 분리하는 객체지향 설계 구조”이고, DI는 그 분리된 전략들을 “외부에서 안전하게 주입해 주는 기술적 도구”로서 서로 협력하는 관계다.
