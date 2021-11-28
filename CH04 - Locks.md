# CH04 - Locks

- Dead Lock 예제 [StackOverflow](https://stackoverflow.com/questions/6470888/reentrant-lock-and-deadlock-with-java) 	
  - functionOne()에서 functionTwo()를 호출 순간 dead lock  
  - 이유: the thread would have to wait for the lock...which it holds itself.

```java
public synchronized void functionOne() {

    // do something

    functionTwo();

    // do something else

    // redundant, but permitted...
    synchronized(this) {
        // do more stuff
    }    
}

public synchronized void functionTwo() {
     // do even more stuff!
}
```

## Dead Lock 해결방법 - `ReentrantLock`

- Reentrant Mutex (a.k.a, Recursive Mutex)
  - 동일한 쓰레드에서 여러번 락을 걸 수 있음 
  - lock 개수만큼 unlock을 해야된다. 
    - 예제에서, `addPotato()` 에서 `addPotato()`으로 갈때 각 메소드에서 lock 2번과 unlock을 2번한다. 

- 자바의 Lock을 구현한 3개 클래스
  - `ReentrantLock`
  - `ReentrantReadWriteLock.ReadLock`
  - `ReentrantReadWriteLock.WriteLock`
  - 전부 이름에 ReentrantLock이 포함됨. 즉, 자바에선 non-ReentrantLock는 지원안함. 

- 코드해석
  - `addPotato()`에 동일 쓰레드에 있는 `addGarlic()`을 호출하여, 데드락 위험이 있지만, `ReentrantLock` 사용하여 해결

```java
/**
 * Two shoppers adding garlic and potatoes to a shared notepad
 */
import java.util.concurrent.locks.*;
class Shopper extends Thread {
    static int garlicCount, potatoCount = 0;
    static ReentrantLock pencil = new ReentrantLock();
    private void addGarlic() {  
        pencil.lock();
        System.out.println("Hold count: " + pencil.getHoldCount());
        garlicCount++;
        pencil.unlock();
    }

    private void addPotato() { 
        pencil.lock();
        potatoCount++;
        addGarlic(); // 마늘도 더해준다.
        pencil.unlock();
    }

    public void run() {
        for (int i=0; i<10_000; i++) {
            addGarlic();
            addPotato();
        }
    }
}

public class ReentrantLockDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread barron = new Shopper();
        Thread olivia = new Shopper();
        barron.start();
        olivia.start();
        barron.join();
        olivia.join();
        System.out.println("We should buy " + Shopper.garlicCount + " garlic.");
        System.out.println("We should buy " + Shopper.potatoCount + " potatoes.");
    }
}
// 결과
// Hold count: 1
// Hold count: 2 
// ...
// Hold count: 1
// Hold count: 2
// We should buy 40000 garlic.
// We should buy 20000 potatoes.
```

## Try Lock 사용 유무 비교 

- Mutex (pencil) 를 얻으려고 스탠다드 Lock 사용 경우 
- 아래 코드 "★★★★여기★★★★★" 를 보면, 다른 사람이 적는 시간 3초를 기다리게 된다. 
  - 효율적으로 가려면, Thinking 시간이 1초이기 때문에 3번 Thinking 해서 다른 사람의 Writing이 끝나면 적으면, 더 빨리 끝낼 수 있다. 

```java
import java.util.concurrent.locks.*;
class Shopper extends Thread {
    private int itemsToAdd = 0; // 각 Shopper가 더하려는 아이템수
    private static int itemsOnNotepad = 0; // 공유되는 노트로, 총 더해진 아이템 개수  
    private static Lock pencil = new ReentrantLock();
    public Shopper(String name) {
        this.setName(name);
    }

    public void run() {
        while (itemsOnNotepad <= 20){ // 총 아이템 개수 20개까지만 허용
            if (itemsToAdd > 0) { // <---- ★★★★여기★★★★★
                try {
                    pencil.lock();
                    itemsOnNotepad += itemsToAdd;
                    System.out.println(this.getName() + " added " + itemsToAdd + " item(s) to notepad.");
                    itemsToAdd = 0;
                    Thread.sleep(300); // 0.3초 (Writing 시간으로 가정)
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    pencil.unlock();
                }
            } else { // look for other things to buy
                try {
                    Thread.sleep(100); // 0.1초 (Thinking 시간으로 가정)
                    itemsToAdd++;
                    System.out.println(this.getName() + " found something to buy.");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}

public class TryLockDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread barron = new Shopper("Barron");
        Thread olivia = new Shopper("Olivia");
        long start = System.currentTimeMillis();
        barron.start();
        olivia.start();
        barron.join();
        olivia.join();
        long finish = System.currentTimeMillis();
        System.out.println("Elapsed Time: " + (float)(finish - start)/1000 + " seconds");
    }
}
// 결과
// Olivia added 1 item(s) to notepad. -> 1개씩 더한다.
// Barron found something to buy.
// ....
// Barron added 1 item(s) to notepad.
// Olivia found something to buy.
// Elapsed Time: 6.693 seconds -> 시간 오래 걸림. 
```

- Try Lock 사용한 경우 
  - `Lock`.`tryLock()`
  - 속도가 빠르다. (걸린시간 결과 확인하기) 
  - 이유
    - `pencil.tryLock()` 을 보면
    - try 해보고, 이미 lock이 걸려있으면, return false 한다. 
- 교훈
  - 다른 쓰레드가 사용중인 자원이 필요할 경우, 마냥 기다리지 말고, 다른일을 먼저 처리해도 되면 그걸 하고 와서 다시 기다려보자. 

```java
import java.util.concurrent.locks.*;
class Shopper extends Thread {
    private int itemsToAdd = 0; 
    private static int itemsOnNotepad = 0; 
    private static Lock pencil = new ReentrantLock();
    public Shopper(String name) {
        this.setName(name);
    }

    public void run() {
        while (itemsOnNotepad <= 20){
            if ((itemsToAdd > 0) && pencil.tryLock()) {  // <--- ★★★ tryLock() 사용 ★★★
                try {
                    itemsOnNotepad += itemsToAdd;
                    System.out.println(this.getName() + " added " + itemsToAdd + " item(s) to notepad.");
                    itemsToAdd = 0;
                    Thread.sleep(300); 
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    pencil.unlock();
                }
            } else { 
                try {
                    Thread.sleep(100); 
                    itemsToAdd++;
                    System.out.println(this.getName() + " found something to buy.");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
// ...
// Barron found something to buy.
// Barron found something to buy.
// Barron found something to buy.
// Barron added 3 item(s) to notepad.
// Olivia found something to buy. 
// Olivia found something to buy.
// Olivia found something to buy.
// Olivia found something to buy.
// Barron found something to buy.
// Olivia added 4 item(s) to notepad.
// Elapsed Time: 2.806 seconds -> 시간이 절반 넘게 단축됬다. 
```

## ReentrantReadWriteLock

- Reader-Writer Lock 특징
  - Shared Read: 여러 쓰레드가 같이 읽을 수 있음 
  - Exclusive Write: 한 쓰레드만이 쓸 수 있음 
- 보통은 일반적은 DB 처럼 Read가 많음. (Write많으면, Reader-Writer Lock 쓸 필요 없음.)
- 코드 해석
  - 10명의 Reader와 2명의 Writers가 캘린더를 공유한다. 
  - Reader-Writer Lock을 사용하지 않았을때 나타나는 문제점을 본다. 

```java
public class ReadWriteLockDemo {
    public static void main(String[] args) {
        for (int i=0; i<10; i++) // 10명의 Reader  
            new CalendarUser("Reader-"+i).start();

        for (int i=0; i<2; i++) // 2명의 Writers
            new CalendarUser("Writer-"+i).start();
    }
}

import java.util.concurrent.locks.*;

class CalendarUser extends Thread {

    private static final String[] WEEKDAYS = {"Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"};
    private static int today = 0;
    private static ReentrantLock marker = new ReentrantLock(); // 이건 Read Write Lock 아님 

    public CalendarUser(String name) {
        this.setName(name);
    }

    public void run() {
        while (today < WEEKDAYS.length-1){ 
            if (this.getName().contains("Writer")) { // Writer - update the shared calendar
                marker.lock();
                try {
                    today = (today+1) % 7;
                    System.out.println(this.getName() + " updated date to " + WEEKDAYS[today]);
                } catch (Exception e)
                    {e.printStackTrace(); }
                {
                    marker.unlock();
                }
            } else { // Reader - check to see what today is
                marker.lock();
                try {
                    System.out.println(this.getName() + " sees that today is " + WEEKDAYS[today]);
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    marker.unlock();
                }
            }
        }
    }
}
// 결과는 아래로 가서 확인하기. 
```

- Read와 Writer Lock을 분리한다. 

```java
// main은 위와 동일

import java.util.concurrent.locks.*;
class CalendarUser extends Thread {
    private static final String[] WEEKDAYS = {"Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"};
    private static int today = 0;
    private static ReentrantReadWriteLock marker = new ReentrantReadWriteLock();
    private static Lock readMarker = marker.readLock();
    private static Lock writeMarker = marker.writeLock(); 

    public CalendarUser(String name) {
        this.setName(name);
    }

    public void run() {
        while (today < WEEKDAYS.length-1){
            if (this.getName().contains("Writer")) { // update the shared calendar
                writeMarker.lock();
                try {
                    today = (today+1) % 7;
                    System.out.println(this.getName() + " updated date to " + WEEKDAYS[today]);
                } catch (Exception e)
                    {e.printStackTrace(); }
                {
                    writeMarker.unlock();
                }
            } else { // Reader - check to see what today is
                readMarker.lock();
                try {
                    System.out.println(this.getName() + " sees that today is " + WEEKDAYS[today] + "; total readers: " + marker.getReadLockCount());
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    readMarker.unlock();
                }
            }
        }
    }
}
```

- 위 두개 결과 비교 
- 첫번째 코드 결과
  - 문제점: 한 reader가 읽는 동안 다른 reader도 읽을 수 있어야 하는데, 쓸데없이 한 리더가 자원을 계속 점유한다. 
  - 궁금: 왜 7번보다 훨씬 많이 나오지...? 예를들어, 첫번째 "Reader-2 sees that today is Sun"이게 30번은 나옴..

```java
Reader-2 sees that today is Sun
//... 무수히 많은 Reader-2 생략 ...
Reader-2 sees that today is Sun
Reader-0 sees that today is Sun
//... 무수히 많은 Reader-0 생략 ...
Reader-0 sees that today is Sun
Reader-4 sees that today is Sun
//... 무수히 많은 Reader-4 생략 ...
Reader-4 sees that today is Sun
Reader-6 sees that today is Sun
//... 무수히 많은 Reader-6 생략 ...
Reader-6 sees that today is Sun
Reader-5 sees that today is Sun
//... 무수히 많은 Reader-5 생략 ...
Reader-5 sees that today is Sun
Reader-8 sees that today is Sun
//... 무수히 많은 Reader-8 생략 ...
Reader-8 sees that today is Sun
Reader-1 sees that today is Sun
//... 무수히 많은 Reader-1 생략 ...
Reader-1 sees that today is Sun
Reader-3 sees that today is Sun
//... 무수히 많은 Reader-3 생략 ...
Reader-3 sees that today is Sun
Reader-9 sees that today is Sun
//... 무수히 많은 Reader-9 생략 ...
Reader-9 sees that today is Sun
Reader-7 sees that today is Sun
//... 무수히 많은 Reader-7 생략 ...
Reader-7 sees that today is Sun
Reader-7 sees that today is Sun
Writer-0 updated date to Mon
Writer-0 updated date to Tue
Writer-0 updated date to Wed
Writer-0 updated date to Thu
Writer-0 updated date to Fri
Writer-0 updated date to Sat
Writer-1 updated date to Sun
Writer-1 updated date to Mon
Writer-1 updated date to Tue
Writer-1 updated date to Wed
Writer-1 updated date to Thu
Writer-1 updated date to Fri
Writer-1 updated date to Sat
Reader-2 sees that today is Sat
Reader-0 sees that today is Sat
Reader-4 sees that today is Sat
Reader-6 sees that today is Sat
Reader-5 sees that today is Sat
Reader-8 sees that today is Sat
Reader-1 sees that today is Sat
Reader-3 sees that today is Sat
Reader-9 sees that today is Sat
Reader-7 sees that today is Sat
```

- 두번째 코드 결과
  - 결과 값이 짧다. 
  - Reader가 고르게 자원을 공유하는 것을 확인할 수 있다.
  - `getReadLockCount()`를 통해 현재 같이 read 중인 (=같이 Lock를 holding 중인) 다른 쓰레드를 확인할 수 있다. 

```
Reader-9 sees that today is Sun; total readers: 2
Reader-4 sees that today is Sun; total readers: 8
Reader-1 sees that today is Sun; total readers: 6
Reader-3 sees that today is Sun; total readers: 9
Reader-6 sees that today is Sun; total readers: 3
Reader-5 sees that today is Sun; total readers: 4
Reader-7 sees that today is Sun; total readers: 10
Reader-2 sees that today is Sun; total readers: 5
Reader-8 sees that today is Sun; total readers: 7
Reader-0 sees that today is Sun; total readers: 1
Writer-0 updated date to Mon
Writer-0 updated date to Tue
Writer-0 updated date to Wed
Writer-0 updated date to Thu
Writer-0 updated date to Fri
Writer-0 updated date to Sat
Writer-1 updated date to Sun
Writer-1 updated date to Mon
Writer-1 updated date to Tue
Writer-1 updated date to Wed
Writer-1 updated date to Thu
Writer-1 updated date to Fri
Writer-1 updated date to Sat
Reader-9 sees that today is Sat; total readers: 1
Reader-1 sees that today is Sat; total readers: 3
Reader-6 sees that today is Sat; total readers: 4
Reader-4 sees that today is Sat; total readers: 2
Reader-2 sees that today is Sat; total readers: 4
Reader-7 sees that today is Sat; total readers: 4
Reader-5 sees that today is Sat; total readers: 4
Reader-3 sees that today is Sat; total readers: 3
Reader-0 sees that today is Sat; total readers: 6
Reader-8 sees that today is Sat; total readers: 5
```

# QUIZ

- How many threads can possess the ReadLock while another thread has a lock on the WriteLock?
  - 0
- What is the maximum number of threads that can possess the WriteLock of a ReentrantReadWriteLock at the same time?
  - 1
- What is the maximum number of threads that can possess the ReadLock of a ReentrantReadWriteLock at the same time?
  - no limit
- Which of these scenario describes the best use case for using a ReadWriteLock?
  - Lots of threads need to both and modify the value of a shared variable.
  - Only a few threads need to both and modify the value of a shared variable.
  - Lots of threads need to read the value of a shared variable, but only a few thread need to modify its value. (답)
  - Lots of threads need to modify the value of a shared variable, but only a few thread need to read its value.
  - What happens when a thread calls Java's tryLock() method on a Lock that is NOT currently locked by another thread?
    - The method immediately returns true
  - What is the difference between the tryLock() and the regular lock() method in Java? 다시
    - tryLock() includes built-in error handling so you do not need a separate try/catch statement
    - tryLock() will continuously try to acquire the lock if it is already taken by another thread
    - tryLock() is a non-blocking version of the lock() method (답)
  - Why is the tryLock() method useful?
    - It enables a thread to execute alternate operations if the lock it needs to acquire is already taken. (답)
    - If multiple threads try to acquire a lock simultaneously, the tryLock() method will randomly pick one to succeed.
    - It enforces fairness among multiple threads competing for ownership of the same lock.
    - It includes built-in protection against common locking errors.
  - Which statement describes the relationship between Lock and ReentrantLock in Java?
    - ReentrantLock is a class that implements the Lock interface.
  - How many times must a thread unlock a ReentrantLock before another thread can acquire it?
    - as many times as that thread locked it
  - A ReentrantLock can be locked `_____`. 다시
    - once by multiple threads at the same time
    - multiple times by different threads
    - multiple times by the same thread (답)



























