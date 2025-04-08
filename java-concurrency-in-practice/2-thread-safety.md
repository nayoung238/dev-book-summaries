## 스레드 안전성

스레드에 안전한 코드란
- 여러 스레드가 클래스에 접근할 때 계속 정확하게 동작하면 해당 클래스는 스레드 안전
- 여기서 정확성이란 클래스가 정의한 명세를 항상 만족해야 한다는 것
- 잘 작성된 클래스 명세는 객체 상태를 제약하는 불변조건과 연산 수행 후 효과를 기술하는 후조건을 정의함

객체 지향 프로그래밍 기법에서 사용하는 캡슐화나 데이터 은닉이 스레드에 안전한 코드를 작성하는 데 도움이 됨
- 접근 범위가 좁아야 상대적으로 쉽게 파악 가능
- 프로그램 상태를 잘 캡슐화할수록 스레드 안전하게 만들기 쉬움 (관리)

### 경쟁 조건

경쟁조건은 상대적인 시점이나 JVM이 여러 스레드를 교차해서 실행하는 상황에 따라 계산의 정확성이 달라질 때를 의미한다.
- 어떤 작업이 먼저 처리되는가에 따라 그 결과가 달라짐을 의미

가장 일반적인 경쟁 조건 형태는 check-then-act 형태 구문
- 어떤 사실을 확인하고 그 관찰에 기반에 행동했지만, 그 관찰이 더 이상 유효하지 않을 때
- Optimistic Lock 사용했을 때

### 지연 초기화
```java
public class SingletonService {
    private static SingletonService instance;

    public static SingletonService getInstance() {
        if (instance == null) {
            instance = new SingletonService();
        }
        return instance;
    }
}
```
두 스레드가 특정 객체가 null 임을 확인하고 객체를 생성하면 서로 다른 인스턴스를 가져갈 수 있다.
- 싱글톤 깨짐
- 이런 지연 초기화는 비용이 큰 객체를 꼭 필요할 때만 초기화하고 싶을 때 사용한다고 함
- 스프링에선 @Component 추가하면 싱글톤으로 관리

### java.util.concurrent.atomic 패키지

java.util.concurrent.atomic 패키지에는 숫자나 객체 참조 값에 대해 상태를 단일 연산으로 변경할 수 있도록 단일 연산 변수 클래스가 준비되어 있다고 함
- 여기서 단일 연산으로 상태 변경이란 스레드 간 충돌 없이 안전하게 값을 바꾼다는 의미
- CAS 연산 + volatile 사용해 CPU 수준에서 원자적 연산 제공

### synchronized

모든 자바 객체는 락으로 사용 가능하며 내장된 락을 암묵적인 락 또는 모니터 락이라고 함
- synchronized 블록 진입 전 락을 획득해야 하며, 블록 탈출 시 락 해제됨

```java
public class Service {
	
    public synchronized void methodA() {...}
    public synchronized void methodB() {...}
}
```

synchronized 잘못 사용하면 응답성 떨어짐
- 위 코드처럼 2개의 프로시저가 있어도 한 스레드만 접근 가능
- 특정 락으로 보호된 코드 블록은 한 번에 한 스레드만 실행할 수 있기 때문에 같은 락으로 보호되는 여러 synchronized 블록은 절대로 동시에 실행되지 않음

### 재진입성

```java
public class A {
	public synchronized void doSomething() {...}
}

public class B extends A {
	public synchronized void doSomething() {
		// 작업
		super.doSomething();
	}
}

B b = new B();
b.doSomething();
```
재진입 특성이 없었다면 위 코드는 데드락 현상이 발생한다.
- 락 획득 후 B.doSomething() 진입
- A.doSomething() 진입하기 위해 락 획득 시도
- 이미 누군가(자신) 락을 획득한 상태라 영원히 대기하는 데드락 현상 발생

### 락으로 상태 보호하기
락을 사용하면 여러 스레드가 순차 접근하기 때문에 동시성 문제를 해결할 수 있음
- DB 격리 수준을 올릴수록 많은 락을 사용해 DB 이상 현상을 제거하지만, 성능이 떨어질 수 있다는 점

특정 변수에 대한 접근을 조율하기 위해 동기화할 때는 해당 변수에 접근하는 모든 부분 동기화 및 같은 락 사용 유지
- 메일이 오면 메일 개수도 같이 카운트
- 이때 메일 저장 작업과 카운팅하는 두 작업에 대한 락은 같아야 함


### 활동성과 성능
동기화 영역이 너무 넓으면 직렬화되어 병렬 처리 능력 떨어짐
- 꼭 필요한 부분만 동기화
- 계산량이 많거나 잠재적으로 대기 상태에 들어갈 수 있는 작업이 락을 잡고 있으면 성능이 떨어질 수 있음
- 그렇다고 원자적 연산이 필요한 부분을 여러 synchronized 블록으로 나누는 것은 추천하지 않음
