# CH03 - Mutual Exclusion

## 1.1 (Data Race)

- 코드 설명: 배런과 올리비아가 공유된 노트를 업데이트해 문제가 생김. 
  - 사야되는 마늘은 10_000_000개 인데... 결과를 보면...

```java
/**
 * Two shoppers adding items to a shared notepad
 */
class Shopper extends Thread {
    static int garlicCount = 0; 
    public void run() {
        for (int i=0; i<10_000_000; i++)
            garlicCount++;
    }
}

public class DataRaceDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread barron = new Shopper();
        Thread olivia = new Shopper();
        barron.start();
        olivia.start();
        barron.join();
        olivia.join();
        System.out.println("We should buy " + Shopper.garlicCount + " garlic.");
    }
}

// 결과: We should buy 10_856_309 garlic.
```

- 해결 방법: Lock 사용
- 위에서 펜이 2개 였기 때문에 문제가 생긴걸로 가정했음. 그래서 펜을 1개만 써서 해결한다는 컨셉 (Lock 구현 비유). 

```java
import java.util.concurrent.locks.*;
class Shopper extends Thread {
    static int garlicCount = 0;
    static Lock pencil = new ReentrantLock();  // Lock 인터페이스의 concrete class

    public void run() {
        pencil.lock(); // 공유자원에 락 걸기 
        for (int i=0; i<10_000_000; i++) 
            garlicCount++;
        pencil.unlock();
    }
}
```

- 문제점
  - 만약 다음과 같이 for문 안에 시간이 걸리는 작업이 있다면, 다른 쓰레드는 현재 쓰레드의 for문이 끝날때까지 오래 기다려야한다. 

```java
public void run() {
    pencil.lock();
    for (int i=0; i<10_000_000; i++) {
        garlicCount++;
        System.out.println(Thread.currentThread().getName() + " is thinking.");
        try {
            Thread.sleep(500); // 시간 걸리는 작업 
        } catch (InterruptedException e) { e.printStackTrace(); }
    }
    pencil.unlock();
}
```

- 해결법
  - garlicCount 부부만이 실제 락을 걸어야 되는 부분임으로 개선을 위해 다음과 같이 바꾼다. 
  - 자원을 나눠쓰게되어, 쓰레드 이름을 교차로 가져오는 것 확인할 수 있다. 

```java
public void run() {
    for (int i=0; i<10_000_000; i++) {
        pencil.lock(); 
        garlicCount++;
        System.out.println(Thread.currentThread().getName() + " is thinking."); // 현재 쓰레드 출력
        try {
            Thread.sleep(500); // 시간 걸리는 작업 
        } catch (InterruptedException e) { e.printStackTrace(); }
        pencil.unlock();
    }
}
```

- 간단히 변수에 1씩 더하는 거라면,
  - 자바의 [java.util.concurrent.atomic](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/concurrent/atomic/package-summary.html) 사용 
  - 여기선 `AtomicInteger` 의 `incrementAndGet()`

```java
import java.util.concurrent.atomic.*;

class Shopper extends Thread {
    static AtomicInteger garlicCount = new AtomicInteger(0); // AtomicInteger
    public void run() {
        for (int i=0; i<10_000_000; i++)
            garlicCount.incrementAndGet();
    }
}

public class AtomicVariableDemo { // 여긴 똑같다. 
    public static void main(String[] args) throws InterruptedException {
        Thread barron = new Shopper();
        Thread olivia = new Shopper();
        barron.start();
        olivia.start();
        barron.join();
        olivia.join();
        System.out.println("We should buy " + Shopper.garlicCount + " garlic.");
    }
}
// 결과: We should buy 20000000 garlic.
```

## 2.1 (Intrinsic Locks)

- synchronized

```java
class Shopper extends Thread {
    static int garlicCount = 0;
    private static synchronized void addGarlic() {
        garlicCount++; 
    }

    public void run() {
        for (int i=0; i<10_000_000; i++)
            addGarlic(); // synchronized 메소드 호출 
    }
}
// 결과: We should buy 20000000 garlic.
```

- 만약 `addGarlic()` 메소드에 static이 없었다면? 
  - 결과가 We should buy 15517241 garlic 이렇게 된다. 
  - 이유는?
  - 저 메소드가 모든 쇼퍼 객체에서 공유되는게 아닌, 객체당 하나씩 되어, `garlicCount++` 을 뒤죽박죽 호출한다. 

## 2.2

- 아래 두개의 차이는 무엇일까?
  - 결과 확인 

```java
class Shopper extends Thread {
    static int garlicCount = 0;
    public void run() {
        for (int i=0; i<10_000_000; i++){
            synchronized (Shopper.class) { // 이 부분만 다르다.
                garlicCount++;
            }
        }
    }
}

class Shopper extends Thread {
    static int garlicCount = 0;
    public void run() {
        for (int i=0; i<10_000_000; i++){
            synchronized (this) { // 이 부분만 다르다.
                garlicCount++;
            }
        }
    }
}

// 첫번째 결과: We should buy 20000000 garlic.
// 두번째 결과: We should buy 12561299 garlic.
```

- 이건 가능할까?

```java
synchronized (garlicCount) { 
    garlicCount++;
}
```

- 객체만 가능하기에 안된다. 
- 그럼 이거는 가능할까?

```java
class Shopper extends Thread {
    static Integer garlicCount = 0; // int에서 Integer 객체로 바꿧다
    public void run() {
        for (int i=0; i<10_000_000; i++){
            synchronized (garlicCount) { 
                garlicCount++;
            }
        }
    }
}
```

- 결과: We should buy 14842266 garlic.
  - 자바의 Integer는 mutable이다. `garlicCount++` 할때마다 새로운 Integer 객체를 생성한다. 
  - 즉, synchronized에서 계속 다른 객체를 사용하는 격. 

- 요약
  - 살펴본 자바에서 mutual exclusion 구현 방식들 4가지 
    - Locks (특정 알고리즘 구현에 flexible)
    - Atomic variables
    - synchronized (구현쉽고, 락에서 생기는 문제점을 예방)
      - synchronized method ()
      - synchronized statement 

# QUIZ

- Why should you avoid using Java's synchronized statement on an immutable object such as an Integer?
  - If you change that variable's value you will be synchronized to a different object.
- What is an advantage of using the "synchronized" methods in Java instead of creating explicit Locks?
  - It is easier to implement and can prevent many common pitfalls of using Locks.
- What happens if Thread A calls the lock() method on a Lock that is already possessed by Thread B?
  - Thread A will block and wait until Thread B calls the unlock() method.
- How many threads can possess a Lock at the same time?
  - 1
- What does it mean to protect a critical section of code with mutual exclusion?
  - Prevent multiple threads from concurrently executing in the critical section.
- Using the ++ operator to increment a variable in Java executes as `_____` at the lowest level.
  - multiple instructions
- In the Java program to demonstrate a data race, why did the data race only occur when each of the threads were incrementing a shared variable a large number of time?
  - The large number of write operations on the shared variable provided more opportunities for the data race to occur.
- Why can potential data races be hard to identify?
  - The data race may not always occur during execution to cause a problem.
- Which of these scenarios does NOT have the potential for a data race?
  - Two threads are both reading and writing the same shared variable.
  - Two threads are both reading the same shared variable. (답)
  - Two threads are both writing to the same shared variable.
  - One thread is reading a shared variable while another thread writes to it.















