## throw vs throws

두 키워드는 이름이 비슷하지만 메서드 내부에서 예외를 ‘발생’시키는 능동적인 행위와 메서드 외부 선언부에 ‘경고창’을 다는 행위라는 명확한 차이가 있다.

| **구분** | **throw (예외 발생)** | **throws (예외 위임)** |
| --- | --- | --- |
| **성격** | **능동적인 동사** | **경고 표지판, 책임 전가** |
| **위치** | 메서드 내부 코드 블록 안 | 메서드 이름 옆 (선언부) |
| **역할** | 의도적으로 예외 객체를 생성하여 **예외 상황의 시작점**을 만드는 것 | 이 메서드 실행 중 특정 예외가 발생할 수 있으니, **호출하는 쪽(상위 로직)에서 알아서 처리하라고 경고하고 위임을 선언**하는 것 |
| **대상** | `throw new XxxException("메시지");` (예외 객체) | `void method() throws XxxException` (예외 클래스 이름) |

```jsx
public void withdraw(int amount) throws InsufficientBalanceException { // 2. throws: 발생할 수 있다고 상위에 경고 및 위임
    if (this.balance < amount) {
        // 1. throw: 개발자가 의도적으로 예외 상황의 시작점을 만듦 
        throw new InsufficientBalanceException("잔액이 부족합니다."); 
    }
    this.balance -= amount;
}
```

## 예외를 상위로 던지는 이유

스프링 부트의 전형적인 3계층(Controller → Service → Repository)구조에서 예외를 직접 처리하지 않고 위임해야 하는 이유

Repository계층에서 DB를 조회했는데 사용자가 찾는 데이터가 없는 예외 상황이 발생했다. 

이때 Repository가 굳이 try-catch를 써서 에러를 삼키고 콘솔에 단순히 System.out.println("데이터 없음");만 찍은 뒤 null을 반환하면 어떻게 될까?

1. Service계층은 데이터가 없는 줄도 모르고 null을 가지고 다음 비즈니스 로직을 수행하다가 NPE를 터트리며 비정상 종료된다.
2. Controller계층은 사용자에게 “데이터를 찾을 수 없습니다”라는 명확한 404 Not Found 응답을 줘야 하는데, 서버가 뻗어버렸기 때문에 서버 에러인 500 Internal Server Error를 던지게 된다.

#### 해결책 : 예외는 상위로 던지고 한 곳에서 일괄 처리(Spring의 방식)

Repository나 Service계층은 문제가 생기면 절대로 자체적으로 해결하려 하지 말고, thorw로 예외를 발생시키고 throws를 통해 상위 계층으로 책임을 전가한다.

“나는 DB에서 값을 가져오는 역할이지, 에러가 났을 때 사용자에게 어떤 화면이나 메시지를 보여줄지는 내 알 바가 아니다” → 역할과 책임을 철저히 분리

예외가 최종적으로 Controller까지 올라오면 스프링이 제공하는 글로벌 예외 처리기인  @RestControllerAdvice가 이를 한곳에서 가로챈다.

→ 클라이언트에게 HTTP 상태코드 400이랑 ‘잔액을 확인해주세요’라는 JSON메시지를 만들어 반환해준다.

## Checked Exception vs Unchecked Exception

자바에서 예외는 크게 RuntimeException을 상속받았는가, 받지 않았는가에 따라 두 갈래로 나뉜다.

#### Checked Exception

- Exception클래스의 자식 중 RuntimeException을 상속받지 않은 예외들이다.
- IOException(파일 입출력 오류), SQLException(DB 통신 오류)
- 컴파일러가 “너 이 예외 처리 코드 안 짜면 컴파일 안 해줄거야”하고 처리를 강제한다.
- 메서드 내부에서 발생할 가능성이 있으면 무조건 try-catch로 직접 잡거나 메서드 옆에 throws를 반드시 명시해야한다. 처리하지 않으면 IDE에서 컴파일 에러가 난다.

#### Unchecked Exception

- RuntimeException을 상속받은 예외들이다.
- NullPointerException, IndexOutOfBoundsException
- 주로 잘못된 파라미터를 전달하거나 코드 설계 오류 등 개발자의 실수로 인해 프로그램 실행 중에 발생하는 예외이다.
- 컴파일러가 예외 처리를 강제하지 않는다. 즉, 메서드 내부에서 throw로 이 예외를 던지더라도 메서드 선언부에 throws를 적는 행위를 생략해도 무방하여 컴파일 에러가 나지 않는다.
