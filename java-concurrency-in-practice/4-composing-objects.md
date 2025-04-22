## 4.1 스레드 안전한 클래스 설계

객체가 갖고 있는 여러 정보를 객체 내부에 숨겨두면 (캡슐화) 전체 프로그램을 다 뒤져볼 필요 없이 객체 단위로 스레드 안전성이 확보되어 있는지 확인할 수 있다. 
클래스가 스레드 안전성을 확보하도록 설계하고자 할 때는 다음을 고려한다.
- 객체의 상태를 보관하는 변수가 어떤 것인지
- 객체의 상태를 보관하는 변수가 가질 수 있는 값이 어떤 종류 및 범위에 해당되는지
- 객체 내부의 값을 동시에 사용할 때 그 과정에서 관리할 수 있는 정책

### 동기화 요구사항 정리

상태 범위란 객체와 변수가 가질 수 있는 가능한 값의 범위를 의미한다.
- 상태 범위가 좁을수록 객체의 논리적인 상태 파악이 쉬움 (예를 들어 final)
- 특정 상태가 올바른지 그렇지 않은지 확인하는 마지노선이 있음
  - Long.MIN_VALUE, Long.MAX_VALUE
  - 초기화 값이 0 이고 ++ 연산만 있다면 해당 변수는 음수가 될 수 없음 (이전 값과 상관관계가 있는 경우)
  - 상관관계를 지켜야만 올바른 상태 범위에 있다고 말할 순 없음. 예를 들어 온도는 현재 온도와 다음 온도의 상관관계가 없음

클래스가 특정 상태를 가질 수 없도록 구현해야 한다면 꼭 캡슐화 
- 해당 변수를 클래스 내부에 숨겨(캡슐화) 보호 
- 그렇지 않고 공개하면 외부에서 ‘올바르지 않다’고 정의한 값이 저장될 수 있음 
- 또한 여러 연산을 실행했을 때 올바르지 않은 상태를 가질 가능성이 있다면 해당 연산은 단일 연산으로 구현해야 함
  - 원자적 연산, 같은 락으로 여러 연산을 원자적 처리

별다른 제약 조건을 두지 않는다면 클래스의 유연성과 실행 성능을 높인다는 측면에서 이와 같은 동기화나 캡슐화 기법을 사용하지 않아도 됨
- 이 주제와 궁금한 것이 @Builder를 클래스 레벨에서 사용하는 경우
- 클래스 레벨에 설정하면 객체 생성의 자유를 주는 대신, 의도하지 않은 객체가 생성될 가능성 있음
- 클래스 레벨에 설정하지 않으면, 무결성 지킬 수 있음. 하지만 생성자 메서드가 계속 증가한다는 단점

### 상태 의존 연산
현재 조건에 따라 동작 여부가 결정되는 연산을 상태 의존 state dependent 연산이라고 한다.
- 예를 들어 큐에 값이 없으면 pop 불가

wait, notify 명령은 특정 상태가 원하는 조건에 다다를 때까지 기다릴 수 있음
- 내부에서 모니터 락 사용
- 하지만 올바르게 사용하기 쉽지 않아 세마포어나 블로킹 큐 등 다른 라이브러리를 사용한 편이 안전

### 상태 소유권
변수를 통해 객체 상태를 정의하고자 할 때 해당 객체가 실제로 소유하는 데이터만을 기준으로 삼아야 한다.
- HashMap 사용 시 내부에서 기능을 구현하는 데 사용할 여러 Map.Entry 객체와 기타 다양한 객체의 인스턴스가 만들어진다.
- 따라서 HashMap 객체의 논리적인 상태를 살펴보고자 한다면 HashMap 내부에 있는 모든 Map.Entry 객체의 상태와 기타 여러 객체의 상태를 한번에 다뤄야 함

가비지 컬렉션 기능까지 생각하면 객체의 소유권 개념이 어렵다.
- C++은 const으로 넘겨주면 읽기 전용으로 값을 넘기고, * pointer로 넘겨주면 소유권도 넘겨주는 것처럼 소유권을 명확하게 정의 가능
- 자바에서도 객체 소유권 문제를 대부분 조절할 수 있지만, 객체를 공유하는 데 있어 오류가 발생하기 쉬운 부분을 가비지 컬렉터가 대부분 알아서 조절해주기 때문에 소유권 개념이 훨씬 불명확한 경우가 많음


```java
public class DocumentManager { 
    private final List<Document> docs = new ArrayList<>();
    
    public void addDocument(Document doc) {
        docs.add(doc);
    }
}
```
클래스는 일반 메서드나 생성 메서드로 넘겨받은 객체에 대한 소유권을 갖지 않는다가 일반적인 모양이지만,
위와 같이 구현하면 넘겨받은 객체의 소유권을 확보할 수도 있다.

동기화된 컬렉션 라이브러리에서도...

```java
List<String> list = Collections.synchronizedList(new ArrayList<>());
```

Collections.synchronizedList를 사용해 
- 넘겨받은 리스트의 소유권을 가져가 내부적으로 동기화를 걸고 
- 외부로 thread-safe wrapper 객체를 넘겨줌
- 단순히 list를 받아서 쓰는 것이 아닌 list 통제권을 가져와 감싸고 새로운 성질(thread-safe)을 부여한 상태로 관리
- 이런 경우 넘겨받은 객체의 소유권을 확보한다고 표현

### ServletContext, HttpSession 내용 추가 필요..

<br>

## 4.2 인스턴스 한정
인스턴스 한정 기법이란 객체가 내부적으로만 사용되고, 외부에서 그 상태에 직접 접근하지 못하게 막음
- 즉, 객체를 적절하게 캡슐화하는 것으로도 스레드 안전성을 확보 가능
- 외부는 꼭 간접적인 경로(wrapper)를 통해서 접근 가능
- 이 기법과 락을 적절하게 활용하면 스레드 안전성이 검증되지 않은 객체도 안전하게 사용 가능

```java
public class A { 
    private final Set<?> s = new HashSet<>();

	public synchronized void addValue(...) { s.add(...); }
	public synchronized void contains(...) { s.contains(...); }
}
```
HashSet 자체는 스레드 안전한 객체가 아니다.
- 하지만 private 으로 설정해 외부에 공개하지 않았으며
- set에 접근하는 방식 모두 synchronized를 사용 -> 즉 락을 통해 접근해 동기화 제공
- 객체 자체를 반환하고 있지 않아 클래스가 모든 원소의 소유권을 가지고 있음

Set으로 관리하는 객체도 스레드에 안전해야 한다.
- 참조 객체를 Set으로 관리한다면 참조 객체 내부의 여러 데이터 동기화 기법이 필요하다.
- 가장 좋은 방법은 참조 객체 자체에서 스레드 안전성을 확보하는 것이고
- 참조 객체를 사용할 때마다 여러 동기화 기법을 사용하는 방법도 있지만 그다지 추천하는 방법은 아님 (참조 객체를 가지고 있는 클래스에서 추가 동기화 작업으로 참조 객체에 접근해라.. 이런 의미인 듯)

```java
List<String> list = Collections.synchronizedList(new ArrayList<>());
```

ArrayList나 HashMap 같은 클래스는 스레드 안전하지 않지만, 자바 플랫폼 라이브러리에서 Collections.synchronizedList 같은 팩토리 메서드를 제공해 멀티스레드 환경에서 안전하게 사용할 수 있도록 한다.
- 이런 팩토리 메서드의 결과로 만들어진 래퍼 클래스는 기본 클래스의 메서드를 호출하는 연동 역할뿐만 아니라 모든 메서드의 동기화를 제공한다.
- 즉 래퍼 클래스를 거쳐야만 원래 컬렉션 클래스에 접근할 수 있기 때문에 래퍼 클래스를 통해 스레드 안전성 확보 가능. 이것이 인스턴스 한정 기법
- wrapper에 넣었더라도 외부에서 원래 컬렉션에 다이렉트 접근이 가능하다면 스레드 안전 깨짐
- 반드시 Collections.synchronizedList 같은 래퍼 인스턴스를 통해서만 접근해야 스레드 안전 보장

### 자바 모니터 패턴
자바 모니터 패턴에 따르는 객체는 변경 가능한 데이터를 모두 객체 내부에 숨긴 다음 객체의 암묵적인 락으로 데이터에 대한 동시 접근을 막는다.
- synchronized 키워드 사용하며 내부에서 모니터 락 자동 사용
- synchronized 블록을 통해서만 변수에 접근할 수 있다면 그것은 변수가 클래스 내부에 숨겨진 것
- synchronized 키워드 사용의 장점은 간결함

```java
public class A { 
    private final Object myLock = new Object();
    B b;
    
    void someMethod() {
        synchronized(myLock) { b 객체 작업 }
    }
}

```

synchronized 블록 진입을 위한 락이 private final 로 설정했기 때문에 내부에서만 사용 가능
- 만약 락이 외부에 공개되었다면 다른 객체도 같은 락으로 동기화 가능
- 외부에 공개되었기 때문에 의도와 다르게 사용될 수 있음을 인지해야 함

### Collections.unmodifiableMap
Collections.unmodifiableMap 이란 말 그대로 수정 불가능한 뷰를 반환한다.
- 단순하게 Map 인스턴스를 Collections 클래스의 unmodifiableMap 메서드로 감사는 것으로는 deppCopy의 기능을 다하지 못하는데 
- unmodifiableMap은 컬렉션 자체만 변경할 수 없게 막아주며 그 안에 보관하고 있는 객체의 내용을 수정하는 것은 막지 못함
- 즉, 반환된 맵을 직접 수정하거나 컬렉션 뷰를 통해 수정하려고 시도하면 UnsupportedOperationException 발생
- 그렇다고 완전한 read-only를 제공하는 것은 아님

```java
public static void main(String[] args) {
    
    Map<String, Person> m = new HashMap<>();
    m.put("kim", new Person("Kim", 25));
    
    Map<String, Person> m2 = Collections.unmodifiableMap(m);
    m2.put("lee", new Person("Lee", 30)); // UnsupportedOperationException!

    m2.get("kim").setAge(30); 

    m1.get("kim").printAge(); // 30 출력
    m2.get("kim").printAge(); // 30 출력
}
```
위 코드에서 볼 수 있듯이 객체에 대한 변경 가능
- unmodifiableMap은 Map의 구조적인 변경인 put, remove 등은 막지만,
- Map 안에 들어있는 객체 참조는 여전히 열려 있음
- 즉, unmodifiableMap으로 객체를 감싸는 것만으로는 불변성 보장이 힘듦
- 때문에 깊은 복사를 해야 완전 보호가 가능함

### DeepCopy 할 것인가 말 것인가
```java
public class A { 
    private final Map<String, B> m;
    public synchronized Map<String, B> getValue() {
        return deepCopy(m);
	}
}

public class B { 
    private int x, y;
}
```

B 객체는 x, y 변수를 private으로 가지고 있고 A 객체도 private으로 Map을 가지고 있지만,
- B 객체에 대한 참조를 가지고 있기 때문에 그대로 반환하면 참조를 반환하게 되어 여러 곳에서 공유가 가능하다.
- 즉 스레드 안전하지 못함
- 스레드 안전하려면 키와 값을 모두 복사해 새로운 인스턴스를 반환하는 깊은 복사 필요

하지만 깊은 복사를 제공하는 getValue() 호출이 증가하면 성능 문제가 발생할 수 있다.
- 깊은 복사를 제공하는 메서드 자체가 synchronized 키워드를 사용하고 있어서 깊은 복사가 진행될 때까지 다른 스레드는 처리하지 못함
- 또한 실시간성이 중요하다면 깊은 복사를 통해 만들어진 값과 실제 값 간의 차이가 발생해 실시간이 떨어질 수 있음
- 그러므로 해당 기능이 어떤 특성을 가졌는지도 파악하며 구현해야 함 (기능 요구사항에 따라 장점이 될 수도, 단점이 될 수도)

<br>

## 4.3 스레드 안전성 위임

### CopyOnWriteArrayList
책에서는 CopyOnWriteArrayList을 사용해 스레드 안전성을 확보하고 있다고 하는데
- 리스트 자체의 구조적 변경에 대해 스레드 안전을 보장하는 것이지
- 그 안의 원소가 불변 객체가 아닌 참조 객체이면 결국 공유되어 스레드 안전이라 말할 수 없지 않나.. 개인적인 생각

참고로
- fork() 로 COW 방식

### 모든 값 한 번에 반환

```java
public class SafePoint { 
    private int x, y;
    
    public synchronized int[] get() {
        return new int[] {x, y};
    }
}
```

getX(), getY()를 따로 구현하는 것이 아니라 모든 값을 한 번에 내보내는 메서드만 제공
- 따로 제공하면 x값 가져오고, y 값 가져오는 사이에 값이 변경될 수 있기 때문

<br>

## 4.4 스레드 안전하게 구현된 클래스에 기능 추가

기존 라이브러리에 새로운 동기화 기능을 추가하고 싶다면 기존 클래스에 다이렉트 추가 or 상속받아 기능을 추가하는 방식이 있다.
- 기존 클래스에 다이렉트 추가하는 것이 안전하지만, 라이브러리라면 수정이 불가능할 수 있음
- 기존 클래스를 상속받아서 하는 추가하는 경우, 동기화를 맞춰야 할 대상이 2개 이상의 클래스에 걸쳐 분산되기 때문에 위험
  - 동기화 기법을 약간이라도 수정한다면 그 하위 클래스는 본의 아니게 적절한 락을 필요한 부분에 적용하지 못할 가능성이 높아 동기화가 깨질 가능성이 있음을 주의해야 함

### 호출하는 측의 동기화

다음 코드의 목적은 contains 와 add 연산을 원자적으로 처리하고 싶은 것이다.
```java
public class A<E> { 
    public List<E> list = Collections.synchronizedList(new ArrayList<E>());
    
    public synchronized boolean putIfAbsent(E x) {
        boolean absent = !list.contains(x);
        if(absent) list.add(x);
		return absent;
    }
}
```

위 코드는 synchronized 키워드를 추가해 putIfAbsent 메서드를 구현했지만,
- 문제는 아무 의미 없는 락을 대상으로 동기화가 진행되면 스레드 안전성을 확보했다고 말하기 어렵다
- list는 Collections.synchronizedList를 통해 내부적으로 자체 락을 이용해서 스레드 안전을 구현
- 제 3의 클래스에서 구현한 putIfAbsent 메서드는 A 클래스의 this 객체를 락으로 사용
- list와 putIfAbsent() 는 서로 다른 락을 사용하는 구조라 list 입장에서 이는 단일 연산이라 보기 힘듦 (동기화되었다고 착각할 뿐)
- 결론은 원자성을 완벽하게 보장하려면 같은 락을 사용해야 함

### Client side lock
위와 같은 상황을 해결하려면 클라이언트 측 락이나 외부 락을 사용해 list가 사용하는 것과 동일한 락을 사용해야 한다. 

```java
public class A<E> { 
    public List<E> list = Collections.synchronizedList(new ArrayList<E>());
    
    public boolean putIfAbsent(E x) {
        synchronized (list) {
            boolean absent = !list.contains(x);
			if(absent) list.add(x);
			return absent;
        }
    }
}
```
클라이언트 측 락은 X 객체를 사용할 때 X 객체가 사용하는 것과 동일한 락을 사용해 스레드 안전성을 확보하는 방법이다.
- 하지만 위 코드터럼 제 3의 클래스를 만들어 클라이언트 측 락 방법으로 단일 연산을 구현하는 방법은
- 특정 클래스(list) 내부에서 사용하는 락을 전혀 관계없는 제 3의 클래스(A)에서 갖다 쓰기 때문에 훨씬 위험해 보인다는 관점도 있음

```java
public class B {
	A<String> a = new A<>();
	synchronized (a.list) {
		if(!a.list.contains(“x”)) { a.list.add(“x”;) }
	}
}
```

이 경우 클라이언트 측 락을 외부에서 사용하는 방식인데
- 어떤 클래스가 내부적으로 사용하는 락 객체를
- 외부에 노출시키고, 외부에서 그걸 synchronized(lock) 같은 방식으로 걸어버리면
- 락의 캡슐화가 깨져서 나중에 클래스 내부 구현이 바뀌었을 때 예기치 못한 동기화 버그가 생길 수 있음 (락 제어가 힘듦 & 캡슐화가 깨져 외부 코드와 충돌 위험 증가)

A 클래스의 list 접근 제어자는 public
- 물론 list가 private이면 외부로 노출되지 않음
- list를 사용하는 클래스가 클래스 A만 존재하는 상황이라면 락 사용을 A 클래스 안에서만 하고 있기 때문에 외부로 노출된 상황은 아님

### 클래스 재구성

```java
public class AList<T> implements List<T> { 
    private final List<T> list;
    public AList(List<T> list) {
		this.list = list;
    }
    
    public synchronized boolean putIfAbsent(T x) {
	    boolean contains = list.contains(x);
	    if (!contains) list.add(x);
	    return !contains;
    }
}
```
위 코드는 AList 클래스 자체를 락으로 사용해 그 안에 포함되어 있는 List와는 다른 수준에서 락을 활용하고 있다.
- AList 클래스가 별도의 스레드 안전성을 부여하고 있다는 의미

AList 클래스를 락으로 사용해 동기화하기 때문에 List 클래스가 내부적으로 동기화 정책을 바꾼다해도 신경 쓸 필요가 없다.
- 내부 list가 ArrayList, synchronizedList, CopyOnWriteArrayList 구현이든 AList의 putIfAbsent()는 오직 AList 클래스의 락만 신경쓰니깐, 외부 구현체의 동기화 변경은 영향을 주지 못함
- AList 클래스가 직접 락을 관리하기 때문에 외부 구현의 동기화 정책 변화에 영향을 받지 않음
- 이는 락 캡슐화 원칙 잘 지킨 사례

```java
public class B<E> {
	private List<E> list = Collections.synchronizedList(new ArrayList<E>());

	public synchronized boolean putIfAbsent(E x) {
		boolean absent = !list.contains(x);
		if(absent) list.add(x);
		return absent;
}
}
```
두 코드의 차이
락 정책
- AList 클래스는 동기화가 외부(AList 클래스)에서 진행
- B 클래스는 내부적(Collections.synchronized)으로 자체 락 보유

동기화
- AList 클래스는 AList 클래스 인스턴스 락 1개로 동기화 진행
- B 클래스는 2개의 락으로 동기화 진행
  - 만약 A 클래스의 list.add가 putIfAbsent() 에서만 진행되면 동기화는 문제되지 않음
  - 하지만 list가 public 이거나 다른 메서드에서 list.add()로 다이렉트 접근하는 코드가 있다면 동기화 깨짐
