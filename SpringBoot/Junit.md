## 가져가야 할 핵심

- Junit의 본질은 자동으로 테스트 결과 판정 → 테스트를 통과 했는지 안 했는지를 사람이 콘솔로 보는 게 아니라 코드가 스스로 판정하게 만들어주는 도구
- Junit은 어노테이션이랑 Assertions 두 축으로 돌아간다.
    - 어노테이션은 ‘이건 테스트다’, ‘이건 매 테스트 전에 실행한다’처럼 테스트의 구조와 흐름을 정의
    - Assertion은 ‘기댓값과 실제 값이 같은가’처럼 결과를 검증
    - 모든 도구는 준비 - 실행 - 검증 어딘가에 들어간다. 앞서 배웠던 given - when - then의 흐름 위에 JUnit이라고 하는 기능을 얹는다고 생각

## JUnit의 정의

- 자바 프로그래밍 언어용 표준 단위 테스트 프레임워크
    
    → 테스트를 작성하는 규칙, 테스트를 찾아서 실행하는 엔진, 결과를 판정하는 도구를 한 세트로 묶어둠
    
- 개발자가 콘솔 창을 눈으로 쳐다보며 성공 여부를 확인하는 비효율을 제거하고, 코드가 스스로 통과 여부를 자동 판정하게 만드는 엔진이다. JUnit 규칙에 맞춰 작성해 두면 기계가 알아서 실행하고 결과를 리포팅해준다.

## 사용 방법

#### 어노테이션 → 테스트의 구조와 흐름 정의

“이것은 테스트 메서드다”, “테스트 시작 전에 이걸 실행해라”처럼 테스트의 라이프사이클(실행 순서와 환경)을 제어하는 뼈대이다.

#### 단언문(Assertions) → 결과의 자동 검증

“내가 기대한 값과 실제 실행 결과가 똑같은가?”를 자가 검증하는 실제 판정관 역할을 한다.

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.assertEquals;

class CalculatorTest {

    @Test
    void 덧셈_테스트() { // 테스트 코드가 문서와 같은 역할을 한다라고 했는데 이렇게 한글로 작성해서 나타내기도 합니다.
	    
        Calculator calculator = new Calculator();
        
        int result = calculator.add(2, 3);
        
        assertEquals(5, result); // 기댓값: 5, 실제값: result
        // 지금처럼 검증 부분에 값을 하드코딩 하는것과 expectedResult 변수 선언하고 거기에 5를 담아서 처리하는것. 두가지 방법이 있습니다. 둘 중 어떤게 더 좋을까요?
    }
    
}
```

`@Test`  → 어노테이션이 붙으면 junit에게 이 메서드는 테스트 메서드다라고 알려주는 표시

→ JUnit은 클래스를 안 훑어보다가 테스트 어노테이션 붙은 메서드를 찾아서 실행한다.

→ 메서드 이름은 자유롭게 지을 수 있고 한글로도 작성 가능하다.

```java
public class Calculator {
    public int add(int a, int b) {
        return a + b;
    }

    public int subtract(int a, int b) {
        return a - b;
    }

    public int divide(int a, int b) {
        if (b == 0) {
            throw new IllegalArgumentException("0으로 나눌 수 없습니다.");
        }
        return a / b;
    }
}
```

→ 테스트 대상 클래스

→ 덧셈 뺄셈이랑 0으로 나누면 예외를 던지는 메서드가 있다.

```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

@DisplayName("계산기 클래스 테스트")
class CalculatorTest {

    private Calculator calculator;

    @BeforeEach
    // 아래 각각의 Test 메소드 실행되기 전에 매번 실행되는 역할 
    // 독립성 때문에..
    void setUp() {
        // 각 테스트가 실행되기 전에 새로운 Calculator 인스턴스를 생성
        // 왜 매번 새로 만드는가?
        calculator = new Calculator();
    }

    @Test
    @DisplayName("덧셈 기능 테스트")
    void testAddition() {
        // given (준비)
        int a = 5;
        int b = 3;
        
        // when (실행)
        int result = calculator.add(a, b);
        
        // then (검증)
        // 테스트 실패했을때 출력되는 설명. 실패한 케이스가 어떤 상황인지 명확하게 보여주고 싶으면 쓸수는 있다.
        assertEquals(8, result, "5 + 3은 8이어야 합니다.");
       
    }

    @Test
    @DisplayName("뺄셈 기능 테스트")
    void testSubtraction() {
        int result = calculator.subtract(10, 4);
        assertEquals(6, result);
    }

    @Test
    @DisplayName("0으로 나눌 때 예외 발생 테스트")
    void testDivisionByZero() {
		    // assertThrows는 발생한 예외 객체를 돌려준다. 그래서 그걸 받아서 assertEquals로 예외 메시지까지 검증한다.
        IllegalArgumentException exception = assertThrows(IllegalArgumentException.class, () -> {
            calculator.divide(10, 0);
        });
        
        // 단순히 예외가 났다를 넘어서 '내가의도한 예외가 내가 의도한 메시지와 함께 발생했는가?'를 확인
        // 이 테스트가 실패했다면 어떤 의미인가? divide 메소드 두번째 인자에 0을 넣어도 예외를 안던지고 그냥 계산을 시도한다면 이 테스트 실패한다. 의도한대로 예외를 던지지 않은것이니까.
        // 그럼 우리가 의도한 약속이 코드에서 깨졌다는 신호
        // 성공했다면 적어도 0으로 나누는 상황에서는 우리가 정한 규칙대로 코드가 막아주고 있구나 라고하는걸 믿을수가 있다.
        assertEquals("0으로 나눌 수 없습니다.", exception.getMessage());
    }
}
```

- `@DisplayName("설명")` : 테스트 결과 창에 복잡한 메서드 이름 대신 인간이 읽기 좋은 깔끔한 한글 설명으로 테스트 제목을 띄워줌
- `@BeforeEach` : 각각의 @Test메서드가 실행되기 바로 직전에 매번 실행되는 준비 영역이다.
    - 왜 매번 새로 인스턴스를 만들까? 테스트의 절대 원칙인 I(독립성) 때문
    - 이전 테스트가 Calculator 객체의 상태를 더렵혔어도, 다음 테스트는 완전히 새롭게 생성된 깨끗한 객체로 시작해야 서로 영향을 안 받는다.
- `@AfterEach` : 각각의 테스트 메서드가 끝난 직후 매번 실행되어, 테스트가 사용했던 무거운 자원(임시 파일, 메모리 데이터 등)을 청소하는 데 쓰인다.
- `@BeforeAll` / `@AfterAll` : 개별 메서드가 아니라 클래스 전체 테스트를 통틀어 시작 전/후에 딱 한 번만 실행된다. DB연결이나 스프링 컨테이너 초기화처럼 매번 띄우기엔 너무 무거운 작업을 단 한 번만 수행하고 공유할 때 사용한다.

Given-When-Then 흐름 중에서 **Then(검증)** 단계에 얹어서 성공/실패 신호를 발생시키는 핵심 메서드들

- `assertEquals(expected, actual, "실패 메시지")`**:** 기대값과 실제값이 같은지 비교해. 세 번째 인자에 실패 시 출력될 구체적인 설명 메시지를 적어두면, 테스트가 왜 깨졌는지 시나리오를 빠르게 파악할 수 있어서 디버깅에 큰 도움이 된다.
- `assertNotNull(object)`**:** 해당 객체가 정상적으로 생성되어 `null`이 아닌지 검증한다.

## 정리

- JUnit은 테스트의 성공, 실패를 사람이 콘솔창을 보고 판단하는 게 아니라, 도구가 판단해준다.
- `@Test`, `@BeforeEach` 같은 **어노테이션**을 이용해 Given-When-Then 흐름 속에서 테스트 객체들을 격리하고 구조를 짠다.
- `assertEquals`나 `assertThrows` 같은 **Assertions 단언문**을 사용해 실제 연산 상태와 예외 메시지까지 자가 검증한다.
- `assertThrows` 를 통해서 예외를 던지고 그에 대해서 테스트하는 상황 → 테스트 코드를 처음 작성하면 성공 케이스만 작성하고 끝나는 경우가 많은데, 좋은 테스트 코드 베이스는 예외에 대해서 얼마나 꼼꼼하게 작성했는지가 중요하다.
