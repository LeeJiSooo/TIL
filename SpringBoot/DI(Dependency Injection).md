## DI 의 필요성

```jsx
[Client] ── 요청(HTTP) ──> Controller ──> Service ──> Repository ──> DB
         <── 응답(JSON) ───     │          │             │
                               └─(의존)──> └────(의존)───>┘
```

의존이란? OrderService가 주문을 처리하기 위해 OrderRepository의 기능을 빌려 쓰는 것처럼 한 클래스가 자기 책임을 다하기 위해 다른 클래스를 필요로 하는 논리적 협력 관계를 의미한다.

주입이란? 내가 필요한 도구(객체)를 내부에서 직접 생성(new)하지 않고, 외부(스프링 컨테이너)가 완성된 객체 내 안으로 밀어 넣어주는 것을 의미한다.

CLI프로그램을 만들 때는 Scanner sc = new Scanner(System.in)처럼 개발자가 조립 과정을 전부 통제했다. 하지만 실무 환경에서는 클래스 수백, 수천 개이다. 이 거대한 의존성을 개발자가 직접 new로 생성하다가 순서가 하나만 틀려도 시스템 전체가 무너진다. 

→ 따라서 프레임워크가 조립과 생성을 전담하고 개발자가 핵심 비즈니스 로직에만 집중하는 것이 생산성 측면에서 훨씬 유리햐다.

## DI를 사용하는 이유

#### 컴파일 타임 vs 런타임 의존성 분리(결합도 완화)

- 직접 객체를 생성할 때(new) : NotificationService 내부 코드에 new EmailSender()를 적는 순간 컴파일 시점(바이트코드로 변환되는 시점)에 두 클래스가 시멘트처럼 굳어진다. 컴파일 타임 의존성과 런타임 의존성이 동일하기 때문에 KakaoSender로 바꾸려면 소스코드를 직접 열어서 수정해야 하는 강한 결합 상태가 된다.
- DI를 적용할 때 : 내부 코드는 MessageSender라는 인터페이스(역할)에만 의존하도록 선언한다. 컴파일러는 “어떤 구현체인지 모르겠지만 일단 패스”한다. 이후 프로그램이 켜지는 런타임 시점에 스프링의 ApplicationContext(IoC컨테이너)가 창고에서 알맞은 객체(EmailSender 또는 KakaoSender)를 찾아 생성자에 넣어준다. 컴파일 타임과 런타임 의존성을 다르게 가져감으로써 코드 수정 없이 부품을 갈아 끼우는 유연함이 생긴다.

#### 단위 테스트(Unit Test)의 용이성

만약 서비스 내부에 new EmailSender()가 박혀있다면 할인 로직 하나를 테스트할 때마다 진짜 외부 API가 호출되어 비용이 발생하고 불필요한 메일이 발생된다.

인터페이스와 DI구조로 설계해두면 테스트 코드 작성 시 실제 외부 연동 객체 대신 가짜 객체를 생성자에 주입하여 네트워크 연결 없이 순수 자바 로직만 안전하게 검증할 수 있다.

## 의존성 주입 방식

1. 필드 주입 

```jsx
@Service
public class CarService {
    @Autowired
    private Engine engine; // 리플렉션 기술로 private 벽을 뚫고 강제 주입
}
```

외부에서 접근할 수 있는 통로(생성자나 Setter)가 아예 없다. 스프링 도움 없이 순수 자바 코드로 단위 테스트를 만들 때 가짜 객체를 꽂아 넣을 방법이 없어 테스트가 불가능해진다.

1. 세터 주입

```jsx
@Service
public class CarService {
    private Engine engine;

    @Autowired
    public void setEngine(Engine engine) { this.engine = engine; }
}
```

CarService라는 빈 껍데기 객체를 먼저 만들고 나중에 Setter로 주입을 한다. 만약 Setter가 호출 되기 전에 누군가 drive()메서드를 실행하면 NPE가 터진다. 주입이 필요한 객체가 누락되어도 컴파일 시점에 알 수 없어 안전하지 못하다.

1. 생성자 주입 - 절대적 표준

```jsx
@Service
public class CarService {
    private final Engine engine; // 1. final 키워드 사용 가능

    // 2. 생성자를 통한 필수 주입 강제
    public CarService(Engine engine) {
        this.engine = engine;
    }
}
```

왜 생성자 주입만 써야 하나?

1. 객체의 불변성
    
    필드에 final 키워드를 붙일 수 있다. 자바 언어 스펙상 final은 생성자 호출 시 딱 한번만 값을 넣을 수 있고 이후 절대 변경 불가능하다. 서버가 켜질때 중간에 다른 것으로 바뀌는 것을 원천 차단하여 애플리케이션을 안전하고 견고하게 유지한다.
    
2. 필수 의존성 누락 방지(컴파일 시점 검증)
    
    자바에서 생성자는 객체가 태어날 때 무조건 호출해야 하는 제약이다. 만약 개발자가 테스트 코드를 짜다가 Engine을 안 넣고 new CarService()를 하려고 하면 자바 컴파일러가 에러를 내며 빌드를 막아준다. 가장 좋은 에러는 실행 전에 컴파일러가 잡아주는 에러이다.
    
3. 순환참조의 즉각적인 차단
    
    A클래스가 B를 원하고, B클래스가 A를 원하는 상황일 때 생성자 주입을 쓰면 애플리케이션이 구동되는 시점에 BeanCurrentlyInCreationException을 터뜨리며 서버 구동을 중단시킨다. 실행 중에 무한 루프를 돌며 서버가 다운되는 상황을 방지하고 개발 단계에서 문제를 바로 인지하게 해준다.
