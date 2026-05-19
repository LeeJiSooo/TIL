## Runnable

스레드가 실행할 작업을 정의하기 위한 표준 인터페이스이다.

단 하나의 메서드 run()만을 가지고 있으며, 이 메서드 안에 스레드가 수행할 코드를 작성한다.

Thread객체와 실행할 코드를 분리함으로써 상속 제약 없이 실행 로직을 재사용할 수 있다.

## 사용 이유

Runnable은 실행 로직을 유연하게 정의하고 다양한 비동기 환경에서 재사용 가능하게 하기 위해서 사용한다.

자바는 Thread클래스를 상속하는 방법과 Runnable 인터페이스를 구현하는 방법으로 스레드를 사용할 수 있는데 실무에서는 Runnable을 사용한다.

- Thread를 상속 받으면 클래스 자체가 쓰레드가 된다.
- Runnable로 구현하면 클래스는 작업이 된다.

→ 작업과 실행 주체를 분리할 수 있다.

#### 실행 로직과 실행 제어를 분리할 수 있다.

Thread클래스를 상속하면 스레드 객체 자체가 실행로직을 포함하게 된다.

→ 재사용이나 테스트가 어렵고 실행 로직과 스레드 제어 코드가 한 클래스에 결합되어 유연성이 떨어지고 확장이어렵다.

반면에 Runnable은 실행할 작업을 객체로 전달하는 구조여서 스레드의 실행 제어와 실행 내용을 분리할 수 있다.

→ 다양한 곳에서 재사용하거나 테스트 코드에서도 독립적으로 실행할 수 있어서 구조가 유연해진다.


#### 실무에서는 대부분 비동기 도구가 Runnable 기반이다.

스레드를 직접 생성해서 사용하는 경우는 거의 없다.

실제로 일할 때는 Runnable로 정의하고 실행 관리는 Executor Service같은 도구에 맡기는 경우가 많다.

동시성과 병렬처리를 안정적으로 관리하기 위해 스레드 풀 기반의 도구를 사용한다.

<aside>
💡 스레드 풀은 필요할 때마다 스레드를 만들지 않고 미리 만들어둔 스레드를 재사용하는 구조이다.

동시에 실행되는 스레드 수를 제한해 과도한 리소스 사용을 방지한다.

</aside>

자바 표준 라이브러리에서는 ExecutorService, ThreadPoolExecutor, ScheduledExecutorService같은 실행 관리 도구들이 제공된다.

스프링 프레임워크에서는 @Async, @Scheduled 등의 어노테이션을 통해 비동기 작업을 처리한다.

→ 이러한 도구들은 내부적으로 작업 단위를 Runnable 또는 Callable 형태로 받아서 실행한다.


#### 상속 제한 없이 어떤 클래스와도 함께 사용할 수 있기 때문이다.

Thread를 상속하면 그 클래스는 더 이상 다른 클래스를 상속할 수 없다.

하지만 Runnable은 인터페이스이므로 다른 클래스를 상속하고 있어도 자유롭게 구현이 가능하다.

## 사용 방법

#### Runnable 인터페이스 구현

Runnable은 하나의 메서드 run()만 가진 함수형 인터페이스이다.

스레드는 이 run()메서드에 작성된 코드를 실행하며 이 메서드에는 수행할 작업 로직을 정의할 수 있다.

```jsx
public class Task implements Runnable {
    @Override
    public void run() {
        System.out.println("작업 실행 중: " + Thread.currentThread().getName());
    }
}
```

#### Thread에 Runnable 전달 후 실행

Runnable을 구현한 객체는 Thread 생성자의 인자로 전달하여 실행할 수 있다.

→ 스레드의 실행 흐름과 실행할 작업이 명확히 분리된다.

```jsx
public class Main {
    public static void main(String[] args) {
        Runnable task = new Task();
        Thread thread = new Thread(task);
        thread.start(); // 별도의 스레드에서 run() 실행
    }
}
```

## Runnable vs Thread

| **항목** | **Runnable** | **Thread** |
| --- | --- | --- |
| 역할 | 실행할 작업 정의(run 메서드) | 실제 실행 주체 (스레드 생성) |
| 관계 | 인터페이스 | 클래스 (Runnable 구현함) |
| 사용 방식 | Thread에 전달되어 실행됨 | 직접 start()로 실행됨 |
| 장점 | 코드와 실행 분리 가능 | 단순 테스트나 빠른 실행에 용이 |

## 정리

- Runnable은 실행할 작업을 독립된 객체로 정의하는 표준타입
- Runnable은 run()메서드에 실행로직을 담고 Thread나 ExcutorService같은 실행 주체가 그 작업을 호출한다.
- Runnable을 통해서 작업과 실행제어를 분리해야 한다.
