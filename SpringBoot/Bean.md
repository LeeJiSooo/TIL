## 일반 자바 객체 VS 스프링 빈(Bean)

두 객체는 메모리(Heap)에 올라간다는 점은 같지만 “누가 만들고, 누가 죽을 때까지 통제하는가”에서 갈린다.

- 일반 자바 객체(DTO, 엔티티 등)
    - 개발자가 코드에서 필요할 때 new로 직접 생성한다.
    - 클라이언트의 요청이 들어올 때마다 동적으로 생겼다가 응답을 주고 참조가 끊기면 가비지 컬렉터(GC)에 의해 사라지는 일회성 소모품이다.
- 스프링 빈(Bean)
    - 스프링 IoC컨테이너가 생성과 소멸을 전담한다.
    - 서버가 켜질 때 딱 한 번 생성되어 Application Context에 박힌다. 태어나는 순간부터 의존성 주입, 초기화, 사용, 소멸까지 모든 생명주기를 컨테이너가 철저하게 통제하는 시스템의 핵심 부품이다.
    

## 빈 등록 방식

#### 컴포넌트 스캔(자동 등록 : @Component, @Service, @Repository)

내가 직접 작성한 비즈니스 로직 클래스들

클래스 위에 어노테이션만 붙여두면 스프링이 구동될 때 패키지를 뒤져서 자동으로 컨테이너에 등록한다.

#### 설정 클래스 기반(수동 등록 : @Configuration + @Bean)

외부 오픈소스 라이브러리(읽기 전용)를 빈으로 등록해야 하거나 환경에 따라 동적으로 객체를 다르게 생성해야 할 때 사용한다.

카카오페이에서 제공한 KakaoPayClient나 암호화 도구인 BCryptPasswordEncoder는 남이 만든 라이브러리라 소스코드를 열어 @Component를 붙일 수 없다.

```jsx
@Configuration // 이 클래스는 빈 설정을 전담하는 파일이다.
public class AppConfig {

		// 메소드 위에 @Bean이 붙어있다
		// "스프링아 내가 이 메소드 안에 직접 외부 객체 만들어서 리턴할테니까 
		// 니가 그 리턴된 녀석들 받아서 빈으로 네 창고에 보관하고 관리해줘
    @Bean // 이 메서드가 리턴하는 객체를 낚아채서 스프링 빈 창고에 넣는다.
    public PasswordEncoder passwordEncoder() {
        // 남이 만든 클래스지만 내가 여기서 new를 써서 직접 생성
		    // 필요한 초기 세팅이 있다면 여기에 넣어서 완성본을 리턴한다.
        return new BCryptPasswordEncoder();
    }
}
```

## 빈 스코프(Bean Scope) : 왜 싱글톤 빈은 ‘무상태(Stateless)’여야 하는가?

스프링 빈은 아무 설정을 안 하면 무조건 싱글톤 스코프로 생성된다. 즉 애플리케이션 전체에서 단 하나의 인스턴스만 만들어 공유한다.

초당 1만 명의 사용자가 몰릴 때마다 OrderService를 1만번 new하고 GC로 지우면 서버가 OOM(메모리 고갈)으로 뻗어버린다. 하나만 튼튼하게 만들어 돌려쓰면 객체 생성 비용이 줄어든다.

**상태를 가진(Sateful)빈은 절대 사용하면 안된다.**

싱글톤 빈은 메모리에 딱 하나만 존재하므로 클래스 바로 밑에 선언하는 인스턴스 변수(필드)도 단 하나만 존재한다. 이 공간을 수많은 쓰레드가 공유하면 동시성 장애가 터진다.

```jsx
@Service
public class OrderService {
    // 공유 메모리에 저장되는 인스턴스 변수
    private String currentLoginUserId; 

    public void createOrder(String userId, String itemName) {
        this.currentLoginUserId = userId; // 사용자A가 들어와서 "userA" 저장
        
        // 찰나의 순간에 쓰레드가 컨텍스트 스위칭되어 사용자B가 들어옴!
        // 이 변수는 하나뿐이라 사용자B의 ID인 "userB"로 덮어써짐!!

        // 다시 사용자A의 쓰레드가 깨어나서 아래 코드를 실행하면?
        System.out.println(this.currentLoginUserId + "님이 주문 완료."); 
        // 결과: userB님이 주문 완료 (A가 주문했는데 영수증은 B가 나옴)
    }
}
```

**무상태(Stateless)빈 설계(표준)**

모든 상태 데이터는 클래스 레벨의 변수가 아니라 메서드의 파라미터나 지역변수로만 다뤄야 한다.

```jsx
@Service
public class OrderService {
    // 주입받은 Repository는 애초에 상태가 없는 객체라 안전함
    private final OrderRepository orderRepository; 

    public void createOrder(String userId, String itemName) {
        // 지역 변수로 선언
        String currentLoginUserId = userId; 
        
        // 지역 변수는 쓰레드마다 독립적으로 가지는 'Stack' 메모리에 생성됩니다.
        // 1만 명이 동시에 들어와도 각자의 Stack 공간에 있어서 동시성 안전
        System.out.println(currentLoginUserId + "님이 " + itemName + "을 주문했습니다.");
    }
}
```

```jsx
┌────────────────────────────────────────────────────────┐
│ [JVM Heap 메모리 (공유 영역)]                              │
│                                                        │
│  ┌──────────────────────────────┐                      │
│  │ 싱글톤 빈: OrderService 객체     │                      │
│  │                               │                      │
│  │  - orderRepository (인스턴스 변수) ──> 로직 객체 주소      │
│  └──────────────────────────────┘                      │
└────────────────────────────────────────────────────────┘
                            ▲
        ┌───────────────────┴───────────────────┐
        │                                       │
┌───────────────────────┐               ┌───────────────────────┐
│ [Thread A (유저 A)]    │               │ [Thread B (유저 B)]    │
│ [Stack 메모리 (격리)]    │               │ [Stack 메모리 (격리)]    │
│                       │               │                       │
│ - userId: "userA"     │               │ - userId: "userB"     │
│ - itemName: "치킨"     │               │ - itemName: "피자"     │
└───────────────────────┘               └───────────────────────┘
```

## Spring Boot에서 IoC, DI, Bean의 관계

스프링부트에서 IoC, DI, Bean은 각각 독립된 개념처럼 보이지만 실제로는 하나의 구조를 이루는 세 요소이다. 이 셋은 “객체를 어떻게 만들고, 어떻게 연결하고, 누가 그 책임을 가지는가”라는 질문에 대한 역할 분담이라고 보면 된다.

#### IoC

전통적인 OOP에서는 객체가 스스로 필요한 객체를 생성하고, 실행 흐름을 직접 제어한다.

```jsx
public class OrderService {

    private final OrderRepository orderRepository;

    public OrderService() {
        this.orderRepository = new OrderRepository();
    }

    public void createOrder() {
        orderRepository.save();
    }
}
```

예를 들어 주문을 처리하는 OrderService가 내부에서 new OrderRepository()를 호출해 저장소 객체를 직접 만들고 사용한다.

이 구조는 OrderService가 무엇을 사용하는지뿐 아니라, 어떻게 생성되는지까지 함께 알고 있어야 한다.

IoC는 이러한 구조를 전환해, 객체 생성과 제어 흐름의 주체를 애플리케이션 코드가 아니라 외부 컨테이너로 이동시킨다. 

즉 객체가 언제 생성되고 어떤 객체가 사용되는지를 객체 자신이 결정하지 않는다.

```jsx
@Service
public class OrderService {

    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    public void createOrder() {
        orderRepository.save();
    }
}
```

이제 주문 처리 흐름에서 `OrderService`는 더 이상 “언제, 어떤 방식으로 Repository를 만들지”를 제어하지 않고, 그 제어권을 Spring과 같은 외부 컨테이너에 넘기게 된다.

이것이 “객체가 주도하던 제어권을 프레임워크가 가져간다”는 구조적 전환이다.

이와 같이 **제어권이 컨테이너로 이동한 구조 위에서 DI(Dependency Injection)** 가 동작한다.

#### DI

DI는 IoC라는 큰 틀 안에서, **객체들이 서로 어떤 의존 관계로 연결될지를 외부에서 결정하고 주입하는 방식이다.**

```jsx
public interface OrderRepository {
    void save();
}

@Repository
public class JpaOrderRepository implements OrderRepository {

    @Override
    public void save() {
        // JPA 기반 저장 로직
    }
}
```

`OrderService`는 주문을 저장하기 위해 “저장 기능을 수행하는 대상이 필요하다”라는 요구만 표현한다.

이때 `OrderService`는 `JpaOrderRepository`인지, `MyBatisOrderRepository`인지를 알지 못하며, `new` 키워드로 객체를 직접 생성하지도 않는다.

어떤 구현을 사용할지, 그 객체를 언제 생성할지는 모두 컨테이너가 설정과 조건에 따라 결정하고, 그 결과를 생성자나 필드를 통해 `OrderService`에 주입한다.

이 과정에서 **Bean** 이 등장한다.

#### Bean

Bean은 **Spring IoC 컨테이너가 생성하고 관리하는 객체**이다.

```jsx
@Service
public class OrderService { ... }

@Repository
public class JpaOrderRepository implements OrderRepository { ... }
```

`OrderService`, `OrderRepository` 같은 구성 요소들은 컨테이너에 의해 Bean으로 등록되고 관리된다.

DI는 아무 객체나 대상으로 동작하지 않고, 컨테이너가 관리하는 Bean들 사이에서만 이루어진다.

즉, 컨테이너가 “이 객체는 `OrderService`이고, 저 객체는 `OrderRepository`다”라고 인지하고 있어야 의존 관계를 연결해 줄 수 있다.

그래서 Bean은 IoC에서 제어권을 넘겨받은 컨테이너가 DI를 통해 객체들을 연결하기 위해 사용하는 관리 대상이자 연결 단위라고 할 수 있다.

| 개념 | 역할 | 예시 |
| --- | --- | --- |
| IoC | 객체 생성과 제어 흐름의 주체를 컨테이너로 이동시키는 구조적 전환 | `OrderService`가 Repository 생성 시점을 결정하지 않음 |
| DI | 컨테이너가 객체 간 의존 관계를 외부에서 결정해 주입하는 방식 | `OrderService` 생성 시 `OrderRepository`를 주입 |
| Bean | 컨테이너가 생성·관리·연결하는 객체 단위 | `OrderService`, `OrderRepository`가 Bean으로 등록됨 |

## 정리

- 거대한 매니저인 컨테이너가 객체들을 빈이라는 관리 대상으로 묶어 통제하면서 일을 할 수 있게 필요한 관계를 DI로 엮어준다.
- Bean은 new로 만들어내는 일반 자바 객체랑 다르게 컨테이너에 의해서 생명주기가 철저히 관리되고 DI를 통해 조합되는 객체이다.
- Bean은 어노테이션을 이용한 컴포넌트 스캔으로 등록할 수 있고 외부 라이브러리처럼 내가 사용할 수 없는 경우에는 설정 클래스 기반으로 등록한다.
- Bean은 99%싱글톤으로 동작한다. → 하나의 객체를 수천, 수만명의 사용자가 동시에 요청하는 쓰레드가 공유하면서 사용하기 때문에 빈 내부에서는 요쳥별로 달라지는 상태값(인스턴스 변수)을 저장해서는 안되고 무상태로 설계해야한다.
