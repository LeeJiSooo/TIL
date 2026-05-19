## ExecutorService

ExecutorService는 스레드 풀 기반의 비동기 작업 실행을 관리하는 스레드 실행 서비스 인터페이스이다.

기존 ) Thread를 직접 생성해 실행 로직 관리 → ExecutorService는 작업 실행, 스레드 생성, 재사용, 종료까지 자동으로 관리하는 실행 환경을 제공

내부적으로 작업 대기열(BlockingQueue)과 스레드 풀(Thread Pool)을 함께 관리하며, 외부에서 전달받은 작업(Runnable 또는 Callable)을 받아 스레드 풀 내에서 효과적으로 처리하는 구조로 동작

<img width="1316" height="742" alt="image" src="https://github.com/user-attachments/assets/bf5d8878-4da2-4660-8e85-0d1a770502b8" />


1. 작업 제출
    
    개발자가 excutor.sumit() 또는 excutor.excute()메서드를 통해 작업을 제출하면 내부 큐로 전달된다.
    
    → Runnable 또는 Callable 인터페이스를 구현한 형태
    
2. BlockingQueue
    
    전달된 작업은 BlockingQueue에 저장되며 들어온 순서대로 큐에 쌓인다. → 스레드 수보다 많은 작업이 들어올 경우 작업을 대기시키는 역할
    
3. Thread Pool
    
    미리 생성되어 있는 스레드 풀의 스레드들은 큐에 작업이 있으면 하나씩 꺼내서 실행한다.
    
    작업이 완료된 스레드는 사라지지 않고 재사용을 위해서 다시 풀에 반환된다.
    
    → 매번 새로 생성하는 비용 절감, 자원을 안정적으로 관리
    

## 사용 이유

스레드를 직접 관리하지 않고도 대량 작업을 안전하게 병렬처리하고 시스템 자원을 예측 가능하게 제어하기 위해서이다.

멀티 스레딩이 필요한 경우 Thread 클래스를 직접 생성해 사용하는 방식이 단순하고 직관적이지만 실무 환경에서는 동시성 처리의 안정성과 예측가능성, 운영 효율성을 고려해야 하므로 `ExecutorService` 를 사용한다.

`ExecutorService` 는 내부에 스레드 풀을 유지하면서 한 번 생성한 스레드를 재사용함 → 불필요한 자원 낭비 차단 한다.

Thread를 매번 직접 생성하면 객체 생성/파괴 비용이 반복되며, 과도한 스레드로 인해 시스템 자원이 고갈되거나 OOM이 발생할 수 있다.

<aside>
💡 OOM(Out Of Memory)는 자바 프로그램이 사용할 수 있는 메모리를 모두 소진해서 더 이상 객체를 생성할 수 없을 때 발생하는 오류이다.

주로 스레드를 과도하게 생성하거나, 작업이 무한히 쌓일 때 발생한다.

java.lang.OutOfMemoryError 예외가 발생한다.

</aside>

## 사용 방법

#### ExecutorService 생성

다양한 Executors 유틸리티 클래스를 통해 다양한 종류의 ExecutorService를 만들 수 있다.

```jsx
ExecutorService executor = Executors.newFixedThreadPool(3); // 스레드 3개로 구성된 풀 생성
```

#### 작업 제출(excute() vs submit())

ExecutorService는 Runnable 또는 Callable을 작업 단위로 받아서 실행한다.

1. execute(Runnable) - 결과 반환 없음

```java
executor.execute(() -> {
    System.out.println("비동기 작업 실행 중: " + Thread.currentThread().getName());
});
```

오직 작업만 실행하고 실행 결과나 예외 정보는 호출자에게 전달하지 않는다.

실행 중 예외가 발생하더라도 잡아내기 어렵고 작업이 정상적으로 끝났는지 여부도 확인할 수 없다.

결과가 중요하지 않은 단순한 비동기 작업(ex. 로그 기록, 알림 전송…)에 사용된다.

1. submit(Runnable or Callable) - 결과 추적 가능

```jsx
Future<String> future = executor.submit(() -> {
    Thread.sleep(1000);
    return "작업 완료";
});

String result = future.get(); // 블로킹 방식으로 결과 대기
System.out.println(result);
```

작업 실행과 함께 Future 객체를 반환한다.

Future는 아직 끝나지 않았지만 나중에 결과를 받을 수 있는 객체이다.

이 객체를 통해 결과를 비동기적으로 조회하거나, 예외 여부를 확인할 수 있다.

작업의 결과를 확인하거나 예외를 추적해야하는 경우에 적합하다.

ex. 데이터 처리, 외부 api 응답 대기, 비즈니스 로직 실행 결과 반환 등

#### 실행 종료 처리(shutdown() vs shutdownNow())

ExecutorService는 명시적으로 종료되지 않으면 JVM이 종료되지 않는다.

따라서 모든 작업이 끝난 후에 반드시 자원을 해제해야 한다.

```jsx
executor.shutdown(); // 남은 작업 완료 후 종료
```

shutdown() : Executor의 종료를 요청하고 현재 큐에 있는 작업은 모두 완료한 후 종료한다.

shutdownNow() : 현재 대기 중인 작업은 취소하고 실행 중인 스레드에는 인터럽트를 요청하여 가능한 빠르게 종료한다.
