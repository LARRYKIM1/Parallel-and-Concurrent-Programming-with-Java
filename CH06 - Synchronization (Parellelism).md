여기서 부터는 Parellel programing 에 대해서 배운다. 

**목표: Using a condition variable**

# CH06 - Synchronization 

# 1 Conditional variable

- Condition variable: 동기화를 도와주기 위고, 쓰레드들을 block하기 위해 사용. 
- Monitor
  - Mutex로 코드의 섹션들을 보호
  - condition이 발생할때까지 쓰레드들을 기다릴 수 있게 해준다. 
- 3 operation of Condition variable
  - wait
    - 자원 사용에 내 차례가 아니면, mutex에 lock을 릴리즈한다.
    - sleep된 상태로 waiting queue에서 기다린다. 
    - 깨어졌을때 lock을 다시 요청한다.
  - signal (=notify or wake)
    - condition variable 큐의 한 쓰레드를 깨운다
  - broadcast

- 코드 설명: 국을 11번먹을 수 있고, 번갈아가면서 먹는다는 가정.
- 문제점: 돌려보면, 번갈아서 안먹고 한쪽이 계속 먹는 것을 볼 수 있음.

```java
class HungryPerson extends Thread {

    private int personID;
    private static Lock slowCookerLid = new ReentrantLock();
    private static int servings = 11;

    public HungryPerson(int personID) {
        this.personID = personID;
    }

    public void run() {
        while (servings > 0) {
            slowCookerLid.lock();
            try {
                if ((personID != servings % 2) && servings > 0) { // 본인순서인지 확인
                    servings--; // it's your turn - take some soup!
                    System.out.format("Person %d took some soup! Servings left: %d\n", personID, servings);
                } else { // not your turn - put the lid back
                    System.out.format("Person %d checked... then put the lid back.\n", personID);
                }
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                    slowCookerLid.unlock();
            }
        }
    }
}

public class ConditionVariableDemo {
    public static void main(String args[]) {
        for (int i=0; i<2; i++)
            new HungryPerson(i).start();
    }
}
```

- 해결 방법: Condition 사용하기 
  - `Condition soupTaken = slowCookerLid.newCondition();`
  - `soupTaken.await()` - 해당 쓰레드는 큐에 들어가게 됨. 
  - `soupTaken.signalAll()` -> signal()쓰면 안됨. 프로그램이 멈춰버림.
- 5개 쓰레드(HungryPerson)이 (꽤나 공정하겐) 순서대로 먹는 것을 볼 수 있따.

```java
class HungryPerson extends Thread {
    private int personID;
    private static Lock slowCookerLid = new ReentrantLock();
    private static int servings = 11;
    private static Condition soupTaken = slowCookerLid.newCondition(); // 추가

    public HungryPerson(int personID) {
        this.personID = personID;
    }

    public void run() {
        while (servings > 0) {
            slowCookerLid.lock();
            try {
                while ((personID != servings % 5) && servings > 0) { 
                    System.out.format("Person %d checked... then put the lid back.\n", personID);
                    soupTaken.await(); // 추가
                }
                if (servings > 0) {
                    servings--;
                    System.out.format("Person %d took some soup! Servings left: %d\n", personID, servings);
                    soupTaken.signalAll(); // 추가
                }
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                    slowCookerLid.unlock();
            }
        }
    }
}
```

# 2 Producer-Consumer pattern

- Producer-Consumer pattern
  - Producer: 공유된 자료구조에 요소를 더한다
  - Consumer: 공유된 자료구조에 요소를 삭제한다

- 동기화 어려운점
  - P와 C사이의 mutex 강제하는 것.
  - 꽉찬 큐에 데이터를 추가하는 것을 행하는 P를 저지시키는 것.
- Pipeline: 더많은 프로세스가 들어가기 위해 사용
- 코드 설명: soup 생산자와 소비자가 있다. 
- 문제점: 생산자가 더 빨라서 큐가 꽉차버려, IllegalStateException 이 난다. 
  - 생산자는 0.2초, 소비자는 0.3초 걸림.

```java
class SoupProducer extends Thread {

    private BlockingQueue servingLine;

    public SoupProducer(BlockingQueue servingLine) {
        this.servingLine = servingLine;
    }

    public void run() {
        for (int i=0; i<20; i++) { // serve 20 bowls of soup
            try {
                servingLine.add("Bowl #" + i);
                System.out.format("Served Bowl #%d - remaining capacity: %d\n", i, servingLine.remainingCapacity());
                Thread.sleep(200); 
            } catch (Exception e) { e.printStackTrace(); }
        }
    }
}

class SoupConsumer extends Thread {

    private BlockingQueue servingLine;

    public SoupConsumer(BlockingQueue servingLine) {
        this.servingLine = servingLine;
    }

    public void run() {
        while (true) {
            try {
                String bowl = (String)servingLine.take();
                System.out.format("Ate %s\n", bowl);
                Thread.sleep(300); // time to eat a bowl of soup
             } catch (Exception e) { e.printStackTrace(); }
        }
    }
}

public class ProducerConsumerDemo {
    public static void main(String args[]) {
        BlockingQueue servingLine = new ArrayBlockingQueue<String>(5);
        new SoupConsumer(servingLine).start();
        new SoupProducer(servingLine).start();
    }
}
```

- 해결방법: 소비자를 더 만든다. 

```java
public static void main(String args[]) {
    BlockingQueue servingLine = new ArrayBlockingQueue<String>(5);
    new SoupConsumer(servingLine).start();
    new SoupConsumer(servingLine).start();
    new SoupProducer(servingLine).start();
}
```

- 하지만, consumer는 계속 soup을 기다리고 있기때문에, 다음과 같이 바꿔준다.

```java
class SoupProducer extends Thread {
	//...
    public void run() {
        for (int i=0; i<20; i++) {
            //...
        }
        servingLine.add("no soup for you!"); //1번째 소비자를 위해 추가
        servingLine.add("no soup for you!"); //2번째 소비자를 위해 추가
    }
}

class SoupConsumer extends Thread {
	//...
    public void run() {
        while (true) {
            try {
                String bowl = (String)servingLine.take();
                if (bowl == "no soup for you!") // 추가
                    break;
                System.out.format("Ate %s\n", bowl);
                Thread.sleep(300);
             } catch (Exception e) { e.printStackTrace(); }
        }
    }
}
```

# 3 Semaphore

- Semaphore: 동기화 방법 중 하나
  - Mutex와 비슷하지만, 여러 쓰레드에 의해 동시에 사용될 수 있따는 점이 다르다 
  - avaiability 확인을 위해 counter 사용. 
- 메서드
  - acquire(): 카운터가 양수면 내리고, 0이면 기다린다.
  - release(): 카운터를 올리고, 기다리는 쓰레드에게 신호를 보낸다. 
- 종류
  - couting semaphore
    - value는 0보다 크거나 같다.
    - 제한된 자원 트래킹하기 위해 사용. (e.g. 커넥션풀, 큐안에 아이템들)
  - binary semaphore
    - value는 0과 1만 있따. 0은 locked 1은 unlocked
- Mutex와 Semaphore 차이는?
  - Mutex는 오직 동일한 쓰레드에 의해 acquired랑 release가 가능한데, Semaphore는 다른 쓰레드들에 의해서도 가능

- 코드설명: 폰 충전기 포트가 제한되어 있어서, 포트 개수보다 이상의 기기를 충전할 수 없다. 

```java

class CellPhone extends Thread {
    private static Semaphore charger = new Semaphore(4);
    public CellPhone(String name) {
        this.setName(name);
    }
    public void run() {
        try {
            charger.acquire();
            System.out.println(this.getName() + " is charging...");
            Thread.sleep(ThreadLocalRandom.current().nextInt(1000, 2000));
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            System.out.println(this.getName() + " is DONE charging!");
            charger.release();
        }
    }
}

public class SemaphoreDemo {
    public static void main(String args[]) {
        for (int i =0; i < 10; i++)
            new CellPhone("Phone-"+i).start();
    }
}
```

- 다음과 같이 바꾸면, binary semphore로 작동한다

```java
private static Semaphore charger = new Semaphore(1); // 개수를 1개로
```

- 하나씩 접근해서 처리함으로 mutex 같이 작동한다고 볼수 있다. 

# QUIZ

- What is a common use case for a counting semaphore?
  - Track the availability of a limited resource.
- In addition to modifying the counter value, what else does calling the semaphore's release() method do?
  - Signal another thread waiting to acquire the semaphore.
- What does the semaphore's release() method do to the counter value?
  - Always increment the counter's value.
- What does the semaphore's acquire() method do to the counter value?
  - If the counter is positive, decrement its value.
- What is the difference between a binary semaphore and a mutex?
  - The binary semaphore can be acquired and released by different threads.
- What happens if the producer puts elements into a fixed-length queue faster than the consumer removes them? 
  - The queue will fill up and cause an error.
- Which architecture consists of a chained-together series of producer-consumer pairs?
  - pipeline
- How should the average rates of production and consumption be related in a producer-consumer architecture?
  - The consumption rate should be greater than or equal to the production rate.
- When should a thread typically signal a condition variable?
  - after doing something to change the state associated with the condition variable but before unlocking the associated mutex
- Why would you use the condition variable's signal() method instead of signalAll()?
  - You only need to wake up one waiting thread and it does not matter which one.
- Condition variables serve as a `_____` for threads to `_____`.
  - holding place; wait for a certain condition before continuing execution
- Condition variables work together with which other mechanism serving as a monitor?
  - mutex













# Preventing a race condition







