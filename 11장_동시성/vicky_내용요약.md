# 11장. 동시성

## 아이템 78. 공유 중인 가변 데이터는 동기화해 사용하라
### synchronized
+ 배타적 실행보장
    + 한 스레드가 변경하는 중이라서 상태가 일관되지 않은 순간의 객체를 다른 스레드가 보지 못하게 막는 용도
    + 한 객체가 일관된 상태를 가지고 생성되면 이 객체에 접근하는 메서드는 그 객체에 락(lock)을 건다. 락을 건 메서드는 객체의 상태를 확인하고 필요하면 수정한다. 즉, 객체를 하나의 일관된 상태에서 다른 일관된 상태로 변화시킨다. 동기화를 제대로 사용하면 어떤 메서드도 이 객체의 상태가 일관되지 않은 순간을 볼 수 없을 것이다.
+ 스레드 사이의 안정적인 통신을 보장
    + 동기화 없이는 한 스레드가 만든 변화를 다른 스레드에서 확인하지 못할 수 있다.
    + 동기화는 일관성이 깨진 상태를 볼 수 없게 하는 것은 물론, 동기화된 메서드나 블록에 들어간 스레드가 같은 락의 보호하에 수행된 모든 이전 수정의 최종 결과를 보게 해준다.
### 원자적 데이터와 동기화
+ 원자적
    + 여러 스레드가 같은 변수를 동기화 없이 수정하는 중이라도, <b>항상 어떤 스레드가 정상적으로 저장한 값을 온전히 읽어옴</b>을 보장한다는 뜻이다.
+ 자바 언어 명세에는 스레드가 필드를 읽을 때 항상 '수정이 완전히 반영된' 값을 얻는다고 보장하지만, 한 스레드가 저장한 값이 다른 스레드에게 '보이는가'는 보장하지 않는다.
> 동기화는 배타적 실행 뿐만 아니라 스레드 사이의 안정적인 통신에 꼭 필요하다.이는 한 스레드가 만든 변화가 다른 스레드에게 언제 어떻게 보이는지를 규정한 자바의 메모리 모델 때문이다.
### 다른 스레드를 멈추는 방법
+ Thread.stop 메서드 사용 제제
#### 1. boolean 필드의 이용
+ 스레드는 자신의 boolean 필드를 폴링하면서 그 값이 true가 되면 멈춘다. 
+ 쓰기 메서드(requestStop)와 읽기 메서드(stopReqeust) 모두를 동기화했음에 주목하자.
```java
public class SyncStopThread {    
  private static boolean stopRequested;     
  private static synchronized void requestStop(){        
    stopRequested = true;    
  }     
  private static synchronized boolean stopRequested(){        
    return stopRequested;    
  }    
  public static void main(String[] args) throws InterruptedException {        
    Thread backgroundThread = new Thread(()-> {            
      int i = 0;            
      while(!stopRequested()) i++;
    });        
    backgroundThread.start();         
    TimeUnit.SECONDS.sleep(1);        
    requestStop();    
  }}
```
### volatile 한정자
+ stopRequested 필드를 volatile으로 선언하면 동기화를 생략해도 된다.
```java
public class VolatileStopThread {     
  private static volatile boolean stopRequested;     
  public static void main(String[] args) throws InterruptedException {        
    Thread backgroundThread = new Thread(()-> {            
      int i = 0;           
      while (!stopRequested)                
        i++;        
    });        
    backgroundThread.start();         
    TimeUnit.SECONDS.sleep(1);        
    stopRequested = true;    
  }}
```
#### volatile 사용시 주의점
+ 안전실패 발생
    + generateSerialNumber() 메서드에서 증가 연산자(++)다. 
    + 이 연산자는 코드상으로는 하나지만 실제로는 nextSerialNumber 필드에 두 번 접근한다. 
    + 먼저 값을 읽고, 그런 다음 (1 증가한) 새로운 값을 저장하는 것이다. 
    + 만약 두 번째 스레드가 이 두 접근 사이를 비집고 들어와 값을 읽어가면 첫 번째 스레드와 똑같은 값을 돌려받게 된다.     
```java
private static volatile int nextSerialNumber = 0; 
public static int generateSerialNumber() {    
  return nextSerialNumber++;
}
// 해결책
private static long nextSerialNumber = 0;
public static synchronized long generateSerialNumber() {
  return nextSerialNumber++;
}
```
### java.util.concurrent.atomic - AtomicLong
+ volatile은 동기화의 두 효과 중 통신 쪽만 지원하지만 이 패키지는 원자성(배타적 실행)까지 지원한다.
```java
private static final AtomicLong nextSerialNum = new AtomicLong(); 
public static long generateSerialNumber() {    
  return nextSerialNum.getAndIncrement();
}
```
### 객체의 안전발행
1. 정적 필드
2. volatile 필드
3. final 필드
4. 락을 통해 접근을 하는 필드
5. 동시성 컬렉션에 저장하는 방법

<br><hr><br>

## 아이템 79. 과도한 동기화는 피하라
+ 과도한 동기화는 성능을 떨어뜨리고 교착상태에 빠트리고, 예측할 수 없는 결과를 낳는다.
+ 동기화 메서드나 동기화 블럭에서 제어를 클라이언트에 양도하면 안된다.
### 외계인 메서드
+ 무슨 일을 할지 알지 못하며 통제할 수 없는 메서드
+ 1. 예외를 일으키거나 2. 교착상태에 빠지거나 3. 데이터를 훼손 할 수 있다.
```java
public class ObservableSet<E> extends ForwardingSet<E> {
    public ObservableSet(Set<E> set) {
        super(set);
    }

    private final List<SetObserver<E>> observers = new ArrayList<>();

    public void addObserver(SetObserver<E> observer) {
        synchronized(observers) {
            observers.add(observer);
        }
    }

    public boolean removeObserver(SetObserver<E> observer) {
        synchronized(observers) {
            return observers.remove(observer);
        }
    }

    private void notifyElementAdded(E element) {
        synchronized(observers) {
            for (SetObserver<E> observer : observers)
                observer.added(this, element);
        }
    }

    @Override
    public boolean add(E element) {
        boolean added = super.add(element);
        if (added)
            notifyElementAdded(element);
        return added;
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        boolean result = false;
        for (E element : c)
            result |= add(element); // notifyElementAdded 호출
        return result;
    }
}
```
#### 외계인 메서드 해결방법
+ 동시성 컬렉션 라이브러리의 CopyWriteArrayList 사용
+ 내부의 배열은 수정되지 않아 순회할 때 락이 필요 없어 매우 빠르다.
```java
private final List<SetObserver<E>> observers = new CopyOnWriteArrayList<>();

public void addObserver(SetObserver<E> observer) {
    observers.add(observer);
}

public boolean removeObserver(SetObserver<E> observer) {
    return observers.remove(observer);
}

private void notifyElementAdded(E element){
    for(SetObserver<E> observer:observers)
    observer.added(this,element);
}
```

## 아이템 80. 스레드보다는 실행자, 태스크, 스트림을 애용하라
### 실행자 프레임워크
+ java.util.concurrent 패키지의 실행자, 태스크, 스트림을 이용하는 편이 더 낫다.
```java
// 큐를 생성한다.
ExecutorService exec = Executors.newSingleThreadExecutor();

// 태스크 실행
exec.execute(runnable);

// 실행자 종료
exec.shutdown();
```
#### 실행자 프레임워크 주요기능
+ submit().get() : 특정 태스크가 완료되기를 기다릴 수 있다. 
+ 태스크 모음 중에서 어느 하나(invokeAny) 혹은 모든 태스크(invokeAll)가 완료되는 것을 기다릴 수 있다.
+ awaitTermination : 실행자 서비스가 종료하기를 기다린다. 
+ ExecutorCompletionService : 완료된 태스크들의 결과를 차례로 받는다. 
+ ScheduledThreadPoolExecutor : 태스크를 특정 시간에 혹은 주기적으로 실행하게 한다. 
### ThreadPool 종류
+ Executors.newCachedThreadPool은 가벼운 프로그램을 실행하는 서버에 적합하다.
+ 무거운 서버에는 스레드 개수를 고정해서 사용하는 것이 좋다.

## 아이템 81. wait와 notify보다는 동시성 유틸리티를 애용하라
+ wait()는 스레드가 일시정지 상태로 돌아가도록
+ notify()는 일시정지 상태인 스레드 중 하나를 실행대기 상태로
+ notifyAll() 은 일시정지 상태인 스레드를 모두 실행대기 상태로 
### 동시성 컬렉션
+ List, Queue, Map 같은 표준 컬렉션 인터페이스에 동시성을 가미해 구현한 고성능 컬렉션이다.
+ 동시성 컬렉션에서 동시성을 무력화하는 건 불가능하며, 외부에서 락을 추가로 사용하면 오히려 속도가 느려진다.
+ 동기화 된 맵을 동시성 맵으로 교체하는 것만으로 동시성 애플리케이션의 성능은 극적으로 개선된다.
```java
public static String intern(String s) {
    String result = map.get(s);
    if ( result == null) {
        result = map.putIfAbsent(s, s);
        if (result == null)
            result = s;
    }
    return result;
}
```




## 아이템 82. 스레드 안전성 수준을 문서화하라

## 아이템 83. 지연 초기화는 신중히 사용하라

## 아이템 84. 프로그램의 동작을 스레드 스케줄러에 기대지 말라
