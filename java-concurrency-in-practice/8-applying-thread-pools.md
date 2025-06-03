## 8.1 작업과 실행 정책 간 연결 관계

Executors 프레임워크 나름대로 실행 정책을 정하거나 변경하는데 있어어 어느 정도의 유연성을 가지고 있지만, 특정 형태의 실행 정책에서는 실행할 수 없는 작업이 있기도 하다.
- 의존성이 있는 작업 (데드락 발생할 수 있음, 물론 스레드 풀의 크기가 크면 데드락 피할 수 있지만, 여전히 가능성 있음)
- 스레드 한정 기법을 사용하는 작업
- 응답 시간에 민감한 작업 (작업을 무제한 대기시키지 말고, timeout을 설정해 일정 주기로 깨어나 다음 선택지를 선택할 수 있는 환경을 만들어라)
- ThreadLocal 사용하는 작업

결론: 스레드 풀에선 서로 독립적인 다수의 작업을 실행할 떄 가장 효과적이다. 그리고 작업 시간이 길지 않은 작업일수록 좋다.

<br>

## 8.2 스레드 풀 크기 조절

스레드 풀의 크기는 작업의 종류와 스레드 풀을 활용할 애플리케이션 특성에 따라 결정
- 스레드 풀의 크기는 하드 코딩하는 것은 그다지 좋은 방법이 아님
- 스레드 풀의 크키가 너무 크면 CPU 자원 경쟁 증가
- 반대로 너무 작으면 작업이 계속 쌓여 작업 속도 떨어짐
- CPU를 많이 사용하는 경우 N개의 CPU를 탑재한 하드웨어에서는 스레드를 N + 1 개로 맞추면 최적의 성능 발휘 (스레드가 block 되어 CPU를 양보하는 경우를 고려)

N(thread) = N(cpu) * U(cpu) * (1 + W/C)
- CPU가 원하는 활용도를 유지할 수 있는 스레드 풀의 크기를 위 수식으로 구할 수 있음
- N(cpu): CPU 개수
- U(cpu): 목표로 하는 CPU 활용도 (0보다 크거나 같고, 1보다 작거나 같음)
- W/C: 작업 시간 대비 대기 시간의 비율

스레드 풀을 적용하면 메모리, 파일 핸들, 소켓 핸들, 데이터베이스 연결과 같은 자원의 사용량도 적절하게 조절할 수 있다.
- 각 작업에서 실제로 필요한 자원의 양을 모두 더한 값을 자원의 전체 개수로 나눠주면 됨
- 이 값이 바로 스레드 풀의 최대 크기에 해당
- 각 작업 하나가 데이터베이스 연결 하나를 사용한다고 가정하면 스레드 풀의 실제 크기는 데이터베이스 연결 풀의 크기로 제한됨

<br>

## 8.3 ThreadPoolExecutor 설정

스레드 풀 클래스는 실행할 작업이 없다 하더라도 스레드의 개수를 최대한 코어 크기에 맞추도록 되어 있음
- 최초 ThreadPoolExecutor를 생성한 이후에도 prestartAllCoreThreads 메서드를 호출하지 않는 한 코어 크기만큼의 스레드가 미리 만들어지지 않음
- 작업이 실행되면서 코어 크기까지의 스레드가 차례로 생성됨

newCachedThreadPool 팩토리 메서드는 스레드 풀의 최대 크기를 Integer.MAX_VALUE 값으로 지정하고 코어 크기를 0으로, 스레드 유지 시간을 1분으로 지정한다.
- 따라서 해당 스레드 풀은 끝없이 크기가 늘어날 수 있으며, 사용량이 줄어들면 스레드 개수가 적당히 줄어드는 효과가 있음

### 큐에 쌓인 작업 관리
newFixedThreadPool 메서드와 newSingleThreadExecutor 메서드에서 생성하는 풀은 
- 기본 설정으로 크기가 제한되지 않은 LinkedBlockingQueue를 사용한다.
- 물론 자원 관리 측면에서 ArrayBlockingQueue 또는 new LinkedBlockingQueue<>(capacity) 처럼 크기 제한된 큐를 사용하거나
- PriorityBlockingQueue와 같이 큐의 크기를 제한시켜 사용하는 방법이 훨씬 안정적이다.

스레드의 개수가 굉장히 많거나 제한이 거의 없는 경우에는 작업을 큐에 쌓는 절차를 생략할 수도 있을텐데 
- 이럴 때는 SynchronousQueue를 사용해 프로듀서에서 생성한 작업을 컨슈머인 스레드에게 직접 전달 가능
- SynchronousQueue는 따지고 보면 큐가 아니고 단지 스레드 간 작업을 넘겨주는 기능을 담당한다고 할 수 있으며
- SynchronousQueue에 작업을 추가하려면 컨슈머 스레드가 이미 작업을 받기 위해 대기하고 있어야 함

### 집중 대응 정책

Abort Policy
- 기본적으로 사용하는 집중 대응 정책
- execute 메서드에서 RuntimeException을 상속받은 RejectedExecutionException 던짐
- execute 메서드를 호출하는 스레드는 RejectedExecutionException 을 잡아서 작업을 더 이상 추가할 수 없는 상황에 직접 대응해야 함

Discard Policy
- 제거 정책은 큐에 작업을 더 이상 쌓을 수 없다면 방금 추가하려던 작업을 아무 반응 없이 제거
- DiscardOldestPolicy는 오래된 항목을 제거하고 새로운 작업을 추가
- 작업 큐가 우선순위에 따라 동작한다면 우선 순위가 가장 높은 항목을 제거

CallerRunsPolicy
- 호출자 실행 정책은 작업을 제거하거나 예외를 던지지 않으면서
- 큐의 크기를 초과하는 작업을 프로듀서에게 다시 넘겨 작업 추가 속도를 늦출 수 있도록 일종의 속도 조절 방법으로 사용
- 메인 스레드가 이 작업을 처리한다면 애플리케이션 전체 성능에 영향을 줄 수 있음

### BoundedExecutor + Semaphore 조합

```java
public class BoundedExecutor {
    private final Executor exec;
    private final Semaphore semaphore;

    public void submitTask(final Runnable runnable) {
        semaphore.acquire();
        try {
            exec.execute((Runnable) () -> {   // run()
                try {
                    command.run();
                } finally {
                    semaphore.release();
                }
            });
        } catch (RejectedExecutionException e) {
            semaphore.release();
        }
    }
}
```
AbortPolicy 를 사용하면 작업 큐가 다 차면 예외를 발생해 바로 실패 처리되었지만, 위 코드처럼 semaphore를 사용하면 실패가 아닌 일단 대기시킨다.
- 스레드 풀의 스레드 개수와 큐에서 대기하도록 허용하고자 하는 최대 작업 개수를 더한 값을 세마포어의 크기로 지정
- 작업을 submit 할 때마다 semaphore를 acquire() 해서 정해둔 총량을 넘는 작업은 submitTask() 에서 block 대기하게 함

AbortPolicy 를 사용해 바로 실패처리한다면 큐의 상태를 감지할 수 있는데
- 세마포어를 사용해 무한 대기를 한다면 대기 스레드 포화, 응답 지연으로 이어질 수 있지 않을까 생각
- 응답 속도와는 상관없이 무조건 처리만 되면 되는 상황이라면 세마포어도 사용해볼만할 것 같은데 (카프카, 레디스 등 외부 기술을 사용할 수 없는 환경에서)


### privilegedThreadFactory

privilegedThreadFactory 를 사용하면 스레드들이 ‘내가 지정한 권한, 컨텍스트’로만 동작하게 강제해서 여러 사용자가 서로 다른 권한으로 작업을 제출해도 보안 혼란/우회 위험을 방지할 수 있다.
- privilegedThreadFactory는 자바 17 부터는 @Deprecated
- privilegedThreadFactory 은 SecurityManager 을 함께 사용할 때 유용한데
- 자바 17부터는 보안 모델이 바뀌고 Spring, Tomcat 등도 이제는 SecurityManager 방식을 권장하지 않음

SecurityManager 사라진 이후 JVM 자체 보호보다는 OS, 컨테이너, 네트워크, 프레임워크, 코드 레벨 등 다양한 층에서 보안을 계층적으로 적용하는 것이 표준
- Docker로 /etc, /var 등 파일 시스템 읽기 불가, 8080 외 포트 봉인, CPU & 메모리 제한
- 프레임워크 수준에서는 Spring Security, OAuth2, JWT 등 활용
- API Gateway 에서 인증/권한 검증, 요청 필터링

<br>

## 8.4 ThreadPoolExecutor 상속

```java
public class A extends ThreadPoolExecutor {
  
   private final ThreadLocal<Long> startTime = new ThreadLocal<>();


   public A(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue) {
       super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue);
   }
  
   protected void beforeExecute(Thread t, Runnable r) {
       super.beforeExecute(t, r);
       startTime.set(System.nanoTime());
   }
  
   protected void afterExecute(Runnable r, Throwable t) {
       try {
           long taskTime = System.nanoTime() - startTime.get();
           // 로그 출력
       } finally {
           super.afterExecute(r, t);
       }
   }
  
   protected void terminated() {
       try {
           // terminated 관련 작업
       } finally {
           super.terminated();
       }
   }
}
```

ThreadPoolExecutor는 상속받아 기능을 추가할 수 있도록 만들어졌다.
- afterExecute 메서드는 run 메서드가 정상적으로 종료되거나 아니면 예외가 발생해 Exception을 던지고 종료되는 등의 어떤 상황에서도 항상 호출됨
- Exception보다 심각한 오류인 Error 때문에 작업이 중단되면 afterExecute 메서드가 실행되지 않을 수 있음
- beforeExecute 메서드에서 RuntimeException이 발생하면 해당 작업도, afterExecute 메서드도 실행되지 않음
- 스레드 풀이 종료 절차를 마무리한 후(모든 작업과 모든 스레드가 종료) terminated 메서드 호출 

