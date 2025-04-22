## 3.1 가시성

가시성
- 한 스레드에서 변경한 값이 다른 스레드에게 즉시 보이지 않는 현상
- CPU L1, L2 캐시는 코어마다 독립적이기 때문에 발생하는 현상

재배치 현상
- JVM 또는 CPU가 성능 최적화를 위해 코드 실행 순서를 작성된 소스 코드와 다르게 바꾸는 것
- 이로 인해 멀티스레드 환경에서는 가시성, 동시성 문제가 발생할 수 있음

stale data
- 더 이상 최신이 아닌 데이터
- 조회수, 댓글수 같은 데이터는 최신 데이터가 아니어도 큰 문제 발생하지 않음
- 같이 고민해볼만한 기능: 프로필 변경, 내 댓글 바로 보기 / 오래된 데이터를 계속 잡고 있는 경우

64비트 연산
- volatile로 지정되지 않은 long이나 double형 64비트 값은 메모리에 쓰거나 읽을 때 2번의 32비트 연산 발생
- 이로 인해 읽은 값의 일부는 최신 값을 나머지는 이전 값을 읽어 아예 다른 값을 읽어올 수 있음
- 최신 JVM에서도 시스템적으로 원자적 연산을 보장하도록 변경된 것이 아니라, volatile로 지정된 경우만 원자적 연산 가능
- 즉, 아직도 기본적으로 원자성이 보장되지 않는다...?
- Write and reads of volatile long and double values are always atomic. Writes to and reads of references are always atomic, regardless of whether they are implemented as 32 or 64 bit values. (JSR-133: Java Memory Model and Thread Specification - 10 Treatment of Double and Long Variable 참고)

위 현상은 volatile, synchronized, 락 같은 기술로 해결 가능
- volatile: CPU 캐시가 아닌 메모리에 직접 접근해 작업하며, 가시성만 해결할 뿐 동시성 문제 발생할 수 있음
- synchronized, 락 기술 사용하면 요청이 직렬화 되어 이상 현상이 발생하지 않음

<br>

## 3.2 공개와 유출

어떤 객체든 유출되고 나면 반드시 잘못 사용할 수 있다고 가정해야 하며 이를 위해 객체 내부는 캡슐화가 필요하다.
- 객체 내부에서 사용하는 값이 적절하게 캡슐화되어 있다면 프로그램이 정상적으로 동작할 것이라고 쉽계 예측 가능하며
- 예상치 못한 상황에서 원래 설계했던 동작을 벗어나지 않도록 제한할 수 있음

에일리언 메서드는 피해라
- 클래스에 정의는 되어 있지만, 그 기능이 만들어져 있지 않은 메서드
- 클래스를 상속받으면서 오버라이드할 수 있는 메서드라고 책에서 설명하는데... 
- gpt의 도움을 빌려 다음과 같이 해석했음
- 상위 클래스가 자식 클래스의 구현에 의존적인 코드는 피해라
- 더 쉽게 설명하면 '호출자 입장에서 정확히 어떤 구현이 호출되는지 예측이 불가능한 구조'는 피해라
- 추가로 '내부 락을 잡은 상태로 얼마나 오래 걸릴지도 모르는 외부 메서드를 호출하는 메서드'도 에일리언 메서드라고 함

### 생성 메서드 안전성

객체의 this 변수는 생성 메서드가 완전하게 종료되기 전까지는 외부에 공개하면 안된다.
- 정상적이지 않은 상태를 외부에 불러 사용할 가능성이 있기 때문
- this escape: 생성자에서 this를 외부로 넘겨 객체 참조가 빠져가는 것

```java
public class A {

    private int value;

    public A() {
        new Thread(() -> {
            System.out.println("value=" + value);
        }).start();
        
        // Thread.sleep

        value = 1;
    }
}
```
위 코드처럼 객체가 완전히 초기화되지 않았는데 새로운 스레드를 생성해 this에 접근하면
- 의도하지 않은 값이 노출되거나, 정상적인 값이 노출되지 못하는 상황 발생
- 이처럼 생성 메서드 실행 도중에 this 변수가 외부에 공개된다면 이론적으로 해당 객체는 정상적으로 생성되지 않았다고 정의

이를 해결하려면 초기화 과정이 완료될 때까지 this를 외부로 공개하면 안 됨
- 생성자 메서드와 this를 사용하는 메서드를 분리

<br>

## 3.3 스레드 한정

가변 객체를 공유해서 사용한다면 동기화는 필수다. <br>
JDBC 표준에 따르면 JDBC의 커넥션 객체가 반드시 스레드 안전성을 확보하고 있어야 하는 건 아니다.
- 애플리케이션 서버에서 DB 연결을 다루는 기능으로 제공하는 풀은 스레드 안전
- 연결 풀은 거의 모든 경우에 여러 스레드에서 동시에 사용하기 때문에 스레드 안전성을 확보하지 못하면 아무런 의미가 없다.
- 라고 설명하는데 이것도 gpt의 도움을 받아 해석해보면
- 커넥션 자체를 여러 스레드가 동시에 같이 사용하면 이상 현상이 발생하니 커넥션은 스레드 안전하지 않음
- 그래서 커넥션은 꼭 한 스레드에게만 할당되어야 함
- 커넥션 풀에 여러 스레드가 동시 접근해도 커넥션을 한 스레드에만 할당되는 작업이 안전하게 처리되므로 스레드 안전

### 스레드 한정

특정 모듈의 기능을 단일 스레드로 동작하도록 구현하면 데드락을 미연에 방지할 수 있다는 장점이 있다.<br>
volatile 변수를 공유해서 사용할 때는 단일 스레드에서만 쓰기 작업을 하도록 동기화를 구현해야 안전하다
- volatile은 그 작업을 CPU 캐시가 아닌 메인 메모리에서 진행해 가시성을 해결하는 것이 목적이지
- 동시성 문제를 해결해주지 않음

### 스택 한정 작업

스택 한정 기법이란 변수를 스레드의 호출 스택에 한정시켜서 해당 스레드만 그 변수에 접근하도록 만드는 기법
- 로컬 변수는 스레드의 스택에만 존재하기 때문에 스레드 안전
- 하지만 참조 객체는 스레드 안전하지 않음
- 이를 해결하려면 ThreadLocal<가변, 참조 객체> 을 통해 스레드 내부에서만 사용하도록 설정해야 함

### ThreadLocal

멀티스레드 환경에서 각 스레드에게 별도의 저장 공간을 할당하여 별도의 상태를 가질 수 있게 함
- 스레드 안전함 -> 경쟁 조건 없음 (경합 회피 가능)
- 로컬 변수, 불변 객체에는 사용할 필요 없음 -> 사용 시 메모리 사용량만 증가

단점
- Thread와 ThreadLocal은 강결합
- 이후 로직이 비동기이거나 병렬 처리할 떄는 ThreadLocal에 있는 정보를 다시 가져와야 한다는 번거로움 발생

ThreadLocal 초기화
- 재사용 패턴이 있다면 스레드 반납 전 remove() 호출로 ThreadLocalMap을 초기화해야 함
- 그렇지 않으면 이후 다른 요청에서 이전 요청의 Entry 값이 노출될 수 있고
- Entry가 WeakReference<ThreadLocal<T>> + value 구조이므로 memory leak으로 이어질 수 있음

```java
public class ThreadLocal<T> {

    private final int threadLocalHashCode = nextHashCode();

    static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal<?>> {
        
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }

            private Entry[] table;
            ... 생략
}
```
```java
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i]; 
         e != null; 
         e = tab[i = nextIndex(i, len)]) {
        if (e.refersTo(key)) {
            e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}

private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;
```

Spring 내에서 ThreadLocal을 사용 중이고 알아서 잘 컨트롤해주고 있고 있음
- Transactional
- SecurityContextHolder

### @Transactional

트랜잭션도 ThreadLocal을 통해 관리된다.

```java
public abstract class TransactionSynchronizationManager { 
    private static final ThreadLocal<Map<Object, Object>> resources =
            new NamedThreadLocal<>("Transactional resources");
    
    private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations = 
            new NamedThreadLocal<>("Transaction synchronizations");

    private static final ThreadLocal<Integer> currentTransactionIsolationLevel = 
            new NamedThreadLocal<>("Current transaction isolation level");

    private static final ThreadLocal<Boolean> actualTransactionActive = 
            new NamedThreadLocal<>("Actual transaction active");
```

TransactionSynchronizationManager가 내부에서 ThreadLocal 사용해 트랜잭션 리소스를 현재 스레드에 국한시켜 관리
- 같은 스레드가 트랜잭션 안에서 여러 다른 빈의 메서드에 접근해도
- 트랜잭션 정보가 ThreadLocal로 관리되고 있어 여러 빈에서 트랜잭션 정보 공유
- 같은 트랜잭션에서 같은 커넥션을 계속 사용
- 결국 이 스레드가 스레드 풀에서 가져왔다면 이 정보도 정리해야 함

### 멀티 스레드 환경에서 ThreadLocal

멀티 스레드 환경에서 ThreadLocal을 사용하면 
- 경쟁 회피 가능: 스레드별 캐시 분리
- CPU 캐시 일관성 비용 절감
  - 같은 캐시에 여러 스레드 접근하면 캐시 간 동기화 비용 발생
  - 반면 스레드별 데이터는 자신의 값을 갖기 떄문에 동기화 비용이 발생하지 않음
  - 완전히 발생하지 않는다고 말하긴 어렵다고 개인적으로 생각 (false sharing)

## 3.4 불변성

불변 객체란 맨 처음 생성되는 시점을 제외하고는 그 값이 전혀 바뀌지 않는 객체를 의미하며, 스레드 안전하다.
하지만 final 설정만으로 불변을 보장할 수 없음
- final 변수면 불변 객체 가능
- private final List<T> list와 같이 final로 설정했지만, 안에서 관리하는 값이 참조 객체라면 변경될 수 있음