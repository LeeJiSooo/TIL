## Synchronize 정의

Synchronized는 여러 스레드가 동시에 공유 자원에 접근하지 못하도록 일시적으로 해당 자원에 대한 접근을 하나로 제한하는 동기화 키워드이다.

여러 스레드가 동시에 같은 메모리 공간(공유 자원)에 접근하면 데이터 충돌이나 예기치 않은 결과가 발생할 수 있다. → Synchronized는 한 번에 하나의 스레드만 해당 자원에 접근할 수 있도록 제한

Synchronized 해당 코드 블록 또는 메서드에 대해 하나의 스레드만 접근할 수 있는 임계영역을 정의한다.

<aside>
💡 임계영역은 공유 자원에 접근하는 프로그램의 일부분이며 여러 스레드가 동시에 실행해서는 안 되는 코드 영역이다.

</aside>

스레드가 이 영역에 접근하면 자동으로 락(lock)을 획득하고 해당 코드 실행이 끝나면 락을 반납하여 다른 스레드가 대기 후에 진입할 수 있게 만든다.

자바에서는 객체나 클래스 단위로 락을 설정할 수 있다.

<aside>
💡 락(lock)은 동시에 여러 스레드가 하나의 공유 자원에 접근하지 못하도록 막는 장치이다.

스레드가 임계 영역에 진입할 때 락을 획득하고 작업이 끝날 때까지 다른 스레드는 그 영역에 들어가지 못하게 대기한다.

</aside>

Spring Boot로 개발하면 service클래스나 controller클래스는 메모리에 딱 하나만 생성되는 싱글톤 객체이다. → 수천명의 사용자가 동시에 접속하면 톰캣서버는 수천 개의 스레드를 생성해서 단 하나의 service에 동시에 접근한다. → 그런데 이 service 객체 안에 상태를 저장하고 다루는 멤버변수가 있다면 수천개의 스레드가 동시에 그 변수를 읽고 쓰려고 한다. → service클래스나 객체 안에는 변경이 가능한 멤버변수가 있어서는 안된다.

## 사용 이유

멀티스레드 환경에서 공유 자원의 충돌을 방지하고, 실행 흐름의 안정성과 데이터의 일관성을 보장하기 위해서이다.

Synchronized를 사용하면 스레드 간의 충돌을 예방하고, 데이터 상태를 안정적으로 유지하며, 실행 순서를 명확하게 제어할 수 있다는 점에서 정확성이 높은 비즈니스 로직이나 공유 객체를 다루는 상황에서 유용하게 작동한다.

반대로, 이런 동기화 제어가 없는 경우에는 여러 스레드가 동시에 같은 자원(메모리, 객체, 컬렉션 등)에 접근하면 데이터가 꼬이거나, 중간에 값이 덮어씌워지거나, 연산 결과가 틀리는 등의 문제가 발생할 수 있다. → 이런 문제는 예외가 발생하지 않기 때문에 “문제는 발생했지만 감지되지 않은 상태” → 무증상 논리 오류(logic bug)로 이어지기 쉬워 실무에서는 가장 위험한 장애 유형 중 하나이다.

#### 무증상 논리 오류(logic bug)

→ 이커머스 백엔드 개발자라고 가정. 운동화 한정판 100켤레 선착순으로 판다.

→ 재고가 1개 남아있을 때 A사용자랑 B사용자가 정확히 같은 0.001초에 구매버튼 눌렀다.

→ A스레드가 DB에서 재고 읽어보니까 1개 B도 1개

→ A스레드가 재고가 1개가 있으니까 1을 빼서 0으로 재고 업데이트 한다.

→ B스레드가 재고가 1개가 있으니까 1을 빼서 0으로 재고 업데이트 한다.

→ 운동화는 2명에게팔렸는데 재고는 -1이 아니라 0으로 기록된다.

—> Race Condition

<aside>
💡 Race Condition(경쟁 상태)란 두 개 이상의 스레드가 하나의 공유 자원(데이터베이스 메모리 변수 등)을 두고 서로 먼저 수정하려고 경쟁하는 상태를 의미한다.

</aside>

이 문제가 발생하는 핵심 메커니즘은 데이터 처리가 ‘원자적(Atomic)이지 않기 때문’이다. 

우리가 코드로 재고감소()를 호출하면 한번에 실행되는 것 같지만 실제 시스템 내부(CPU/DB)에서는 다음 3단계로 쪼개져서 실행된다.

1. Read(읽기) : DB에서 현재 재고를 가져온다. (재고=1)
2. Modify(수정) : 가져온 값에서 1을 뺀다. (1 - 1 = 0)
3. Write(쓰기) : 계산된 결과를 DB에 다시 저장한다. (재고=0)

A스레드가 1,2,3 단계를 순서대로 끝내고 나서 B스레드가 시작했다면 아무 문제가 없지만 0.0001초 차이로 동시에 들어오면 실행 순서가 아래처럼 엉켜버린다.

1. A스레드 : DB에서 읽어서 재고 1개 (Read)
2. B스레드 : 나도 DB에서 읽어서 재고 1개(Read - A가 아직 안 줄어서 1로 보임)
3. A스레드 : 1에서 1빼서 0 저장 (Modify & Write)
4. B스레드 : 나도 아까 1로 읽었으니까 1 빼서 0으로 저장(Modify & Write)

→ B가 A의 수정 사항을 완전히 덮어써버림.. → 갱신 손실 발생

## 사용 방법

Synchronized를 메서드 전체에 붙일 수도 있고, 필요한 코드 블록 일부에만 적용할 수도 있다.

동기화 대상이 되는 객체(락의 기준)는 this(인스턴스), 클래스 객체, 또는 명시적으로 지정한 객체 중 하나이다.

```jsx
// 메서드 전체에 동기화 적용
public synchronized void method() { ... }

// 블록 단위로 동기화 적용
synchronized (lockObject) {
    // 임계 영역
}
```

메서드에 Synchronized를 선언하면 해당 메서드 전체가 임계 영역으로 간주된다.

이 경우 락은 인스턴스(this) 또는 클래스(Class 객체)를 기준으로 자동 적용된다.

블록 단위의 synchronized(객체)는 지정한 객체를 기준으로 락을 설정한다.

→ 여러 개의 임계 영역을 구분하거나 특정 범위에만 동기화를 적용하고 싶을 때 사용한다.

### 인스턴스 메서드에 적용

대상 : 메서드 전체 

락 기준 : this 객체

```jsx
public class BankAccount {
    private int balance = 10000;

    // 1번 형태: 인스턴스 메서드 전체 동기화
    public synchronized void withdraw(int amount) {
        balance -= amount;
    }

    public synchronized void deposit(int amount) {
        balance += amount;
    }
}
```

- 철수와 영희가 '동일한 계좌 객체'(`Account_A`)를 공유하고 있다.
- 철수가 `Account_A.withdraw()`를 호출해 돈을 뽑는 순간, 철수는 `Account_A`라는 **`this` 객체 자체의 열쇠**를 쥐고 문을 잠근다.
- 바로 그 직후에 영희가 같은 계좌에 돈을 넣으려고 `Account_A.deposit()`을 호출한다.
- 하지만 `deposit()` 역시 `synchronized` 메서드라 `Account_A`의 열쇠가 필요하다. 열쇠는 철수가 갖고 있으므로, **영희는 철수가 출금을 끝마치고 열쇠를 돌려줄 때까지 꼼짝없이 기다려야한다.**

### static 메서드에 적용

대상 : 메서드 전체

락 기준 : Class 전체

```jsx
public class BankAccount {
    private static int totalBankFunds = 100000000; // 은행 전체 총 자산

    // 2번 형태: static 메서드 전체 동기화
    public static synchronized void auditBank() {
        // 은행 전체 자산 감사 중... 다른 스레드가 건들면 안 됨
    }
}
```

- 철수는 `Account_A(강남점 계좌)`를 쓰고 있고, 민수는 `Account_B(부산점 계좌)`를 쓰고 있다. 둘은 완전히 다른 객체(인스턴스)이다.
- 이때 은행 본사 스레드가 `BankAccount.auditBank()`를 호출해 은행 총자산 감사를 시작한다.
- 이 메서드는 `static synchronized`이기 때문에 개별 계좌가 아니라 **`BankAccount.class`라는 클래스 자체에 락**을 걸어버린다.
- 이 순간, 철수가 `Account_A`를 쓰든 민수가 `Account_B`를 쓰든 상관없이, 이 클래스로 만들어진 **모든 객체의 `static synchronized` 메서드는 전면 마비**되어 대기 상태가 된다. (단, 일반 인스턴스 메서드는 영향을 받지 않는다.)

### 코드 블록 일부를 동기화하는 방법(**synchronized (this))**

대상 : 블록 일부

락 기준 : this 객체

```jsx
public class BankAccount {
    private int balance = 10000;

    public void transfer(int amount) {
        // 1단계: 영수증 프린트하기 (동기화 필요 없음, 누구나 동시 가능)
        System.out.println("이체 처리를 시작합니다..."); 

        // 3번 형태: 블록 일부만 this(내 계좌) 기준으로 동기화
        synchronized (this) {
            // 2단계: 실제 돈 차감 (위험한 임계 영역!)
            balance -= amount;
        }

        // 3단계: 로그 기록하기 (동기화 필요 없음)
        System.out.println("이체 완료 로그를 전송합니다.");
    }
}
```

- 1번 형태처럼 여전히 `this`(내 계좌)를 열쇠로 쓰기 때문에, 다른 스레드가 내 계좌의 다른 `synchronized` 공간에 들어오는 걸 막는 효과는 같다.
- 하지만 결정적인 차이는 '시간'이다. 철수가 `transfer()`를 호출했을 때, 1단계 영수증 프린트를 하는 동안에는 열쇠를 잠그지 않는다.
- 딱 2단계 **`synchronized(this)` 블록에 진입하는 순간에만 잠깐 내 계좌를 잠그고**, 돈 빼자마자 바로 열쇠를 반납한다.
- 덕분에 다른 스레드(영희)가 기다리는 시간이 훨씬 줄어들어 프로그램 효율이 좋아진다.

### 명시적 객체에 락을 거는 방법 **(synchronized (lock))**

대상 : 블록 일부

락 기준 : 지정된 객체

```jsx
public class BankAccount {
    private int balance = 10000;
    private String password = "1234";

    // 각각의 목적에 맞는 전용 자물쇠(객체)를 명시적으로 생성
    private final Object moneyLock = new Object();
    private final Object passwordLock = new Object();

    public void deposit(int amount) {
        // 4번 형태: 명시적으로 지정한 객체(moneyLock)로 락 설정
        synchronized (moneyLock) {
            balance += amount; // 입금 처리
        }
    }

    public void changePassword(String newPassword) {
        // 4번 형태: 명시적으로 지정한 객체(passwordLock)로 락 설정
        synchronized (passwordLock) {
            password = newPassword; // 비밀번호 변경
        }
    }
}
```

- 만약 1번이나 3번 형태처럼 `this`를 열쇠로 썼다면, 철수가 `deposit()`으로 입금하는 동안 영희는 비밀번호를 바꾸고 싶어도(`changePassword`) 무조건 대기해야 했다. 입금과 비번 변경은 아무 상관이 없다.
- 하지만 여기서는 **`moneyLock` 자물쇠**와 **`passwordLock` 자물쇠**를 따로 만들었다.
- 이제 철수가 입금을 하느라 `moneyLock`을 잠그고 들어가도, 영희는 아무런 방해를 받지 않고 `passwordLock`을 열고 들어가서 **동시에 비밀번호를 바꿀 수 있다.** 멀티스레드의 장점을 극대화하는 가장 정교하고 효율적인 동기화 방식이다.

| **형태** | **대상** | **락 기준** |
| --- | --- | --- |
| synchronized 인스턴스 메서드 | 메서드 전체 | this 객체 |
| synchronized static 메서드 | 메서드 전체 | Class 객체 |
| synchronized (this) | 블록 일부 | this 객체 |
| synchronized (객체) | 블록 일부 | 지정된 객체 |

## 정리

- 여러 스레드가 공유 자원에 동시에 접근할 때 발생하는 문제를 막기 위해 단 한번에 하나의 스레드만 접근할 수 있는 임계영역을 설정해야 한다.
- synchronized 역할은 자바에서 임계 영역을 설정하고 자물쇠 lock를 걸어주는 가장 기본적인 키워드이고 무증상논리오류를 막고 데이터의 일관성을 지킨다.
- 락의 범위 최적화 → 메서드 전체에 락을 걸어줄 수도 있는데 성능을 위해서 블록을 나누거나 명시적 락 객체를 사용할 수 있다.
