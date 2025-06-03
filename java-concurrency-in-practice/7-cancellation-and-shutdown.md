## 7.1 작업 중단

실제 상황에서 특정 스레드나 서비스를 즉시 멈춰야 할 경우는 거의 없다.
- Thread.stop, Thread.suspend 메서드는 사용하면 안됨
- interrupt 메서드로 특정 스레드에게 작업 중단을 요청해야 함

interrupt
- 특정 스레드에게 인터럽트 요청이 있었다는 것만 전달할 뿐
- 실제로 스레드가 중단되는 것은 아님
- 실제로 멈출지는 호출 스레드의 구현 방식에 달려있음

인터럽트 정책 및 전달
- 인터럽트 처리 정책이란 인터럽트 요청이 들어왔을 때 해당 스레드가 인터럽트를 어떻게 처리해야 하는지에 대한 지침
- 작업을 실행하는 스레드에서 인터럽트가 발생했다면 인터럽트 사실을 상위로 전달해야 함
- 작업을 실행하는 스레드에서 인터럽트를 무시하면 호출 스레드(또는 메인 스레드)는 인터럽트 발생 사실을 파악하지 못함
- InterruptedException을 던지거나, 그에 따른 결과 값을 반환해야 함
- 또는 Thread.currentThread().interrupt() 로 실행 스레드 상태를 유지해야 함

Thread.currentThread().interrupt() 상태 유지
```java
Runnable task = () -> {
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt(); // 상태 복구
    }

    if (Thread.currentThread().isInterrupted()) {
        // 인터럽트 상태 감지 가능
        return; // 또는 처리 중단
    }

    doSomething();
};
```
- Thread.currentThread().interrupt() 상태 유지만으로는 인터럽트 사실이 상위로 전달되지 않음
- 그럼 throw e 대신 Thread.currentThread().interrupt() 를 하는 경우는 어떤 경우냐면 
- 위 코드와 같이 인터럽트가 발생헀을 때 어떤 작업을 추가로 해야할 떄 (상태 유지를 위해 interrupt() 호출)
- 아래 코드와 같이 Runnable에서는 InterruptedException throw 불가능

```java
Runnable r = () -> {
   try {
       Thread.sleep(1000);
   } catch (InterruptedException e) {
       Thread.currentThread().interrupt();
       // throw e; // CE
   }
   if (Thread.currentThread().isInterrupted()) return;
};
```

### 인터럽트 패턴

```java
public Task getNextTask(BlockingQueue<Task> queue) {
     boolean interrupted = false;
     try {
          while(true) {
               try {
                    return queue.take();
	        } catch (InterruptedException e) { interrupted = true; }
          }
     } finally {
          if (interrupted) Thread.currentThread().interrupt();
     }
}
```
- 인터럽트는 무시하지 않겠지만, 작업은 무조건 처리하고 나간다는 코드 (1개의 스레드가 처리)
- catch 부분에서 상태유지를 위해 interrupt()를 호출하는 것이 아니라 flag를 설정하는 이유는 무한 루프가 발생할 수 있기 때문
- 인터럽트를 걸 수 있는 블로킹 메서드는 대부분 실행되자마자 가장 먼저 인터럽트 상태를 확인하며 인터럽트가 걸린 상태가면 즉시 InterruptedException을 던지는 경우가 많아 인터럽트 상태를 너무 일찍 설정하면 무한 반복에 빠질 수 있음

### 인터럽트 상태 확인은 수시로
인터럽트가 걸릴 수 있는 블로킹 메서드를 전혀 사용하지 않는다해도 작업이 진행되는 과정 곳곳에서 현재 스레드의 인터럽트 상태를 확인해준다면 인터럽트에 대한 응답 속도를 높일 수 있다.
- 그래야 오래 블록되는 현상에 들어가기 전 구해줄 수 있음
- 인터럽트 상태 확인 주기는 응답 속도와 효율성 측면에서 적절한 타협점을 찾아야 함
- 만약 응답 속도가 가장 주요한 애플리케이션이라면 **인터럽트에 재빠르게 응답하지 않으면서** 오래 실행되는 메서드를 호출하지 않는 것이 좋음

### 여러 상태와 연관
작업 중단 기능은 인터럽트 상태 뿐만 아니라 여러 다른 상태와 관련이 있을 수 있다. 예를 들어 ThreadPoolExecutor 내부의 풀에 등록되어 있는 스레드에 인터럽트가 걸렸다면
- 인터럽트가 걸린 스레드는 전체 스레드 풀이 종료되는 상태인지 먼저 확인
- 스레드 풀 자체가 종료되는 상태였다면 스레드를 종료하기 전에 스레드 풀을 정리하는 작업을 실행하고
- 스레드 풀이 종료되는 상태가 아니라면 스레드 풀에서 동작하는 스레드의 수를 그대로 유지시킬 수 있도록 새로운 스레드를 하나 생성해 풀에 등록시킨다.
- newFixedThreadPool 은 ThreadPoolExecutor의 runWorker(), processWorkerExit() 메서드를 통해 스레드 수 유지

### Future 작업 중단

```java
public static void timedRun(Runnable r, long timeout, TimeUnit unit) throws InterruptedException {
	Future<?> task = taskExec.submit(r);
	try {
        task.get(timeout, unit);
    } catch (TimeoutException e) {
        // finally 에서 interrupt() 호출
    } catch (ExecutionException e) {
	    throw launderThrowable(e.getCause());
    } finally {
	    task.cancel(true); 
    }
}
```
ExecutorService.submit 메서드를 실행하면 작업을 나타내는 Future 인스턴스를 리턴받는다.
- Future에는 cancel 메서드가 있는데 boolean 값을 넘겨 받으며 취소 요청에 따른 작업 중단 시도가 성공적이었는지를 알려주는 결과를 리턴받을 수 있음
- 작업 자체가 실행 전이라면 boolean 값 상관 없이 큐에서 제거 및 true
- true 설정: 작업이 실행 중이라면 interrupt() 호출 / false 면 아무것도 안함
- 이미 완료된 작업이라면 무시

### 인터럽트에 응답하지 않는 블로킹 작업 다루기

자바 라이브러리에 포함된 여러 블로킹 메서드의 대부분은 인터럽트가 발생하는 즉시 멈추면서 InterruptedException을 띄우도록 되어 있어 작업 중단 요청에 적절하게 대응하는 작업을 쉽게 구현할 수 있다. 
하지만 그렇지 않은 경우도 있다.

```java
public class ReaderThread extends Thread {
	private final Socket socket;
	private final InputStream in;

	@Override
	public void interrupt() {
		try { socket.close(); } 
		catch (IOException ignored) {}
		finally { super.interrupt(); } 
    }
}
```
InputStream.read() 같은 소켓 읽기 메서드는 블로킹 메서드
- Thread.interrupt()는 read() 같은 소켓 블로킹 메서드에는 효과 없음
- Socket을 강제로 닫아야 read()가 예외를 던지면서 깨어남 (read 중단)

super.interrupt() 로 인터럽트 플래그를 설정
- socket.close()는 read() 같은 블로킹 IO를 IOException으로 깨울 뿐 
- 해당 스레드의 인터럽트 상태는 설정되지 않기 때문에 super.interrupt() 호출해 인터럽트 상태 유지

<br>

## 7.2 스레드 기반 서비스 중단

애플리케이션을 깔끔하게 종료하려면 스레드 기반의 서비스 내부에 생성되어 있는 스레드를 안전하게 종료시킬 필요가 있다. 
- 그런데 스레드를 선점적인 방법으로 강제로 종료시킬 수는 없기 때문에 스레드에게 알아서 종료해달라고 부탁할 수밖에 없음
- 스레드 풀에 들어 있는 모든 작업 스레드는 스레드 풀이 소유한다고 볼 수 있고 개별 스레드에 인터럽트를 걸어야 하는 상황이 된다면 그 작업은 스레드를 소유한 스레드 풀에서 책임을 져야 함

### 종료 플래그
로그를 출력할 때 프로듀서-컨슈머 패턴을 적용해 로그 생성 및 출력 작업을 분리
- 로그는 BlockingQueue에 쌓아둠
- 로그 출력 전담 스레드가 BlockingQueue에 쌓인 로그를 계속 실행하느라 JVM이 정상적으로 멈추지 않는 현상이 발생하면 안됨
- BlockingQueue.take 메서드는 인터럽트에 대응하도록 만들어져 있기 때문에 컨슈머를 종료하는 것이 쉬움
- 하지만 컨슈머가 그냥 멈춰 버리도록 구현하는 것은 그다지 만족스러운 방법이 아님

```java
private final BlockingQueue<String> queue;
public void log(String msg) throws InterruptedException {
	if (!shutdownRequested) queue.put(msg);
	else throw new IllegalStateException(“logger is shut down”);
}
```
다음과 같이 종료 플래그를 사용하면 특정 상태부터는 더 이상 로그 메시지를 큐에 넣지 않을 수 있음
- 그러면 컨슈머는 종료 요청이 왔을 때 큐에 있는 모든 메시지를 소비하고 정상 종료할 수 있음
- 또한 로그 메시지를 생성하기 위해 대기 상태로 머무르는 일도 없어짐
- 하지만 위 코드에서도 경쟁 조건 발생할 수 있음 (책에서는 synchronized 사용해 상태 확인 + 로그 생성 작업을 단일 연산으로 처리하라고 함)


### ExecutorService 종료
shutdown 메서드
- 안전하게 종료하는 방법
- 종료 속도는 느리지만 큐에 등록된 모든 작업을 처리할 때까지 스레드를 종료시키지 않고 놔두기 때문에 작업을 잃을 가능성이 없어 안전

shutdownNow 메서드
- 강제로 종료하는 방법
- 실행 중인 모든 작업을 중단하도록 한 다음 아직 시작하지 않은 작업의 목록을 그 결과로 리턴시킴
- 응답이 훨씬 빠르지만
- 실행 도중에 스레드에 인터럽트를 걸어야 하기 때문에 작업이 중단되는 과정에서 여러 문제가 발생할 가능성 있음

shutdownNow 메서드 단점
- 강제 종료하면서 실행되지 않았던 모든 작업 목록을 리턴
- 그러면 ExecutorService를 사용했던 클래스는 리턴받은 작업을 보관할 수 있음
- 하지만 실행했지만 완료되지 않은 작업이 어떤 것인지 알 수 있는 방법은 없음
- 238p 7.21 코드처럼 작성하면 중단 시점에 실행 중이던 스레드를 알 수 있음

```java
try { 
    runnable.run(); 
} finally {
    if (isShutdown() && Thread.currentThread.isInterrupted()) {
	tasksCancelledAtShutdown.add(runnable);
    }
}
```
238p 7.21 코드도 경쟁 조건이 발생할 수 있음
- 실제로는 작업이 취소되었지만 겉으로는 해당 작업이 완료되었다고 잘못 판단할 가능성 있음
- 실행 중이던 작업의 마지막 명령어를 실행하는 시점과 해당 작업이 완료되었다고 기록해두는 시점의 가운데에서 스레드 풀을 종료시키도록 하면 발생하게 됨

```java
// 233p 7.16
public Class A {
	private final ExecutorService exec = newSingleThreadExecutor();
	public void stop() throws InterruptedException {
		try {
            exec.shutdown();
            exec.awaitTermination(TIMEOUT, UNIT);
        } finally {
            writer.close();
        }
    }
}
```
위 A 클래스는 스레드를 사용해야 하는 부분에서 스레드를 직접 갖다 쓰기 보다는 ExecutorService를 사용해 스레드의 기능을 활용한다.
- ExecutorService를 특정 클래스의 내부에 캡슐화하면 애플리케이션에서 서비스와 스레드로 이어지는 소유 관계에 한 단계를 더 추라하는 셈이고
- 각 단계에 해당하는 클래스는 모두 자신이 소유한 서비스나 스레드의 시작과 종료에 관련된 기능을 관리

### Poison pill
프로듀서-컨슈머 패턴으로 구성된 서비스를 종료하는 방법 중 하나는 poison pill 독약이다. 
- 특정 객체를 큐에 쌓고, ‘이 객체를 받았다면 종료해야 한다‘ 라는 의미를 부여한다.
- FIFO 유형이라면 이 객체를 만나기 전까지는 쌓인 작업을 처리할 수 있고 이후는 처리하지 않음

<br>

## 7.3 비정상적인 스레드 종료 상황 처리

스레드를 예상치 못하게 종료시키는 가장 큰 원인은 바로 RuntimeException
- RuntimeException은 대부분 프로그램이 잘못 구현되어 있어 발생하거나 기타 회복 불능의 문제점을 나타내는 경우가 많아 try-catch 구문으로 잡지 못하는 경우가 많다.
- RuntimeException은 호출 스택에 따라 상위로 전달되기 보다는 현재 실행되는 시점에서 콘솔에 스택 호출 추적 내용을 출력하고 해당 스레드를 종료시키도록 되어 있음

### UncaughtExceptionHandler
UncaughtExceptionHandler 기능을 사용하면 처리하지 못한 예외 상황으로 인해 특정 스레드가 종료되는 시점을 정확하게 알 수 있다.
- 스레드별 UncaughtExceptionHandler 지정 가능
- 자바에서 기본적으로 제공하는 스레드 풀에서는 예상치 못한 예외가 발생했을 때 해당 스레드가 종료되도록 하면서 try-finally 구문을 사용해 스레드가 종료되기 전에 스레드 풀에 종료된다는 사실을 알려 다른 스레드를 대체해 실행할 수 있도록 함
- 이런 곳에 UncaughtExceptionHandler를 지정하지 않거나 기타 다른 오류 확인 방법을 전혀 사용하지 않는다면 오류가 생긴 작업이 아무 소리 없이 종료되어 개발자나 운영자가 혼란스러울 수 있음

UncaughtExceptionHandler 가 호출되도록 하려면 반드시 execute 메서드를 통해서 작업 실행해야 함
- submit 메서드로 등록된 작업에서 예외가 발생하면 Future.get 메서드에서 해당 예외가 ExecutionException에 감싸진 상태로 넘어옴

<br>

## JVM 종료
### shutdown hook
예정된 절차대로 종료되는 경우에 JVM은 가장 먼저 등록되어 있는 모든 종료 훅을 실행시킨다. 
- 종료 훅은 Runtime.addShutdownHook 메서드를 사용해 등록되었지만 아직 시작하지 않은 스레드를 의미
- 하나의 JVM에 2개 이상의 종료 훅이 등록되어 있는 경우 어떤 순서로 훅이 실행될지는 모름 (순서 규칙 없음)
- JVM은 종료 과정에서 계속해서 실행되고 있는 애플리케이션 내부의 스레드에 대해 중단 절차를 진행하거나 인터럽트를 걸지 않음
- 계속해서 실행되던 스레드는 결국 종료 절차가 끝나는 시점에 강제로 종료

여러 종료 훅이 있는 경우
- 한 종료 훅은 A 서비스를 종료하고
- 다른 종료 훅에서 A 서비스를 통해 특정 작업을 하고 종료해야 한다면 그 과정이 실패할 수 있음
- 그러므로 서비스별 각자 종료 훅을 만들어 등록하기 보다는
- 모든 서비스를 정리할 수 있는 하나의 종료 훅을 사용해 각 서비스를 의존성에 맞춰 순서대로 정리하는 것도 하나의 방법
- 정리된 여러 종료 훅을 단일 스레드에서 순차 처리하면 경쟁 조건이나 데드락 상황을 방지할 수 있음

### 데몬 스레드
JVM이 처음 시작할 때 main 스레드를 제외하고 JVM 내부적으로 사용하기 위해 실행하는 스레드는 모두 데몬 스레드다.
- GC나 기타 여러 가지 부수적인 스레드
- 일반 스레드와 데몬 스레드는 종료될 때 처리 방법이 약간 다를 뿐 그 외에는 모든 것이 완전히 동일

JVM은 남아있는 스레드 가운데 일반 스레드가 있는지 확인하고
- 일반 스레드는 모두 종료되고
- 남아있는 스레드가 모두 데몬 스레드라면 즉시 JVM 종료 절차 진행
- 즉. JVM이 중단될 때는 모든 데몬 스레드가 버려지는 셈
- finally 블록도 실행되지 않으며 호출 스택도 원상 복구되지 않음
- 이런 특성 때문에 데몬 스레드는 보통 부수적인 용도로 사용하는 경우가 많음
  - 메모리 내부에 관리하고 있는 캐시에서 기한이 만료된 항목을 주기적으로 제거하는 등의 부수적인 단순 작업

### finalize 메서드
결론부터 얘기하면 finalize 메서드는 사용하지 말고 다른 방법으로 처리하라 <br>
finalize 메서드가 정의되어 있는 객체는 명시적으로 풀어줘야 하는 자원을 정리할 수 있도록 GC에 수집될 때 finalize 메서드를 호출해 실행시킨다.
- 이 메서드는 JVM이 관리하는 스레드에서 직접 호출하기 때문에 finalize 메서드에서 사용하는 모든 애플리케이션 상태 변수를 다른 스레드에서 사용할 수 있음
- 따라서 동기화 작업 필수
- 하지만 이 메서드의 실행 시점은 보장되지 않음
- 그러므로 try-finally 구문에서 각종 close 메서드를 적절하게 호출하는 것만으로 finalized 메서드에서 해야할 일을 훨씬 더 잘 처리할 수 있음
- finalize 메서드에가 더 나을 수 있는 유일한 예는 바로 native 메서드에서 확보했던 자원을 사용하는 객체 정도 뿐


