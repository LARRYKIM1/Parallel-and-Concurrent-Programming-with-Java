# CH05 - Liveness

- Deadlock: 서로 자원을 쥔 상태에서 서로의 자원이 release되기를 기다리는 상황
- Liveness: 시스템이 계속 작동하기 위해 필요한 속성

## Deadlock 사례

- 코드 해석
  - 3명의 철학자가 스시를 먹는데, 젓가락이 3개(3쌍아님)가 있고 2개를 먼저 집어야 스시를 먹을 수 있다. 
  - 데드락이 걸릴 수 있는 상황인데, 여러번 돌려도 잘 돌아갈 수 도 있다. (어느 순간 서버가...)
  - sushiCount의 숫자를 500_000으로 크게 바꾸면, 
    - 1차: 작동안됨
    - 2차: 작동안됨
    - 3차: 10개 output 나오고 정지. 
- 이 코드에서 데드락 발생하는 이유
  - 만약 3명 철학자의 각 첫번째 젓가락인 chopstickA, chopstickB, chopstickC 가 동시에 lock을 걸게되면 데드락이 걸린다. 예를 들어, A는 B가 필요한데 다른 쓰레드에서 락을 걸어뒀고 B는 C가 필요한데 다른 쓰레드에서 락을 걸뒀고, C는 A가 필요한데 또 락이 걸려있으니, 데드락 상태가 된다. 

```java
/**
 * Three philosophers, thinking and eating sushi
 */

import java.util.concurrent.locks.*;

class Philosopher extends Thread {
    private Lock firstChopstick, secondChopstick;
    private static int sushiCount = 5; // 
    public Philosopher(String name, Lock firstChopstick, Lock secondChopstick) {
        this.setName(name);
        this.firstChopstick = firstChopstick;
        this.secondChopstick = secondChopstick;
    }

    public void run() {
        while(sushiCount > 0) { // eat sushi until it's all gone

            // pick up chopsticks
            firstChopstick.lock();
            secondChopstick.lock();

            // take a piece of sushi
            if (sushiCount > 0) {
                sushiCount--;
                System.out.println(this.getName() + " took a piece! Sushi remaining: " + sushiCount);
            }

            // put down chopsticks
            secondChopstick.unlock();
            firstChopstick.unlock();
        }
    }
}

public class DeadlockDemo {
    public static void main(String[] args) {
        Lock chopstickA = new ReentrantLock();
        Lock chopstickB = new ReentrantLock();
        Lock chopstickC = new ReentrantLock();
        new Philosopher("Barron", chopstickA, chopstickB).start();
        new Philosopher("Olivia", chopstickB, chopstickC).start();
        new Philosopher("Steve", chopstickC, chopstickA).start();
    }
}

// 1번째 결과
Steve took a piece! Sushi remaining: 4
Steve took a piece! Sushi remaining: 3
Steve took a piece! Sushi remaining: 2
Steve took a piece! Sushi remaining: 1
Steve took a piece! Sushi remaining: 0
// 2번째 결과
Olivia took a piece! Sushi remaining: 4
Olivia took a piece! Sushi remaining: 3
Olivia took a piece! Sushi remaining: 2
Barron took a piece! Sushi remaining: 1
Barron took a piece! Sushi remaining: 0
```

- 해결방법 
  - Steve의 젓가락의 순서를 chopstickC, chopstickA에서 chopstickA, chopstickC로 바꾼다. 
  - 그럼, Steve가 A를 기다리다리는 동안 올리비아의 B와 C를 통해 스시를 먹고 릴리즈를 하면, 배런의 A와 (릴리즈된) B를 가지고 스시를 먹고 릴리즈하면, 스티브도 (릴리즈된) A와 B를 가지고 밥을 먹을 수 있다. 
  - 즉, 데드락이 걸리지 않고 돌아가면서 처리하게 된다.

```java
new Philosopher("Steve", chopstickA, chopstickC).start();
```

- 결과를 보면, 안멈추고 50만개 스시를 잘 처리하는 것을 볼 수 있다. 

- ```
  ...
  Olivia took a piece! Sushi remaining: 24130
  Olivia took a piece! Sushi remaining: 24129
  Olivia took a piece! Sushi remaining: 24128
  ...
  Olivia took a piece! Sushi remaining: 3
  Olivia took a piece! Sushi remaining: 2
  Olivia took a piece! Sushi remaining: 1
  Olivia took a piece! Sushi remaining: 0
  ```

- 데드락 해결방법 
  - Lock Ordering 
    - 모든 락들이 어느 쓰레드에서든 항상 동일한 순서로 발생되게 한다. 하지만 이 방법은 feasible 하진 않다.
  - Lock Timeoout 
    - 시간 제한

## 데드락과 비슷한 상황

- 한 쓰레드에서 에러가 나서, unlock을 못하는 상황 
- 0으로 나누게 해서 에러 발생시키면, 다른 철학자들도 스시를 못먹는 상황이 발생한다. 

```java
public void run() {
    while(sushiCount > 0) {
        firstChopstick.lock();
        secondChopstick.lock();

        if (sushiCount > 0) {
            sushiCount--;
            System.out.println(this.getName() + " took a piece! Sushi remaining: " + sushiCount);
        }

        if ( sushiCount == 10 ) System.out.print(10/0); // 0으로 나누니 ArithmaticException 

        secondChopstick.unlock();
        firstChopstick.unlock();
    }
}
```

- 해결방법
  - try catch -> 크리티컬 섹션은 항상 try 블록에 넣기!

```java
public void run() {
    while(sushiCount > 0) {
        firstChopstick.lock();
        secondChopstick.lock();

        try {
            if (sushiCount > 0) {
                sushiCount--;
                System.out.println(this.getName() + " took a piece! Sushi remaining: " + sushiCount);
            }

            if ( sushiCount == 10 ) System.out.print(10/0); // 0으로 나누니 ArithmaticException 
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            secondChopstick.unlock();
            firstChopstick.unlock();
        }
    }
}
```

## Starvation

- 쓰레드가 자원을 계속 얻지 못하고 다른쓰레드에게 뺏기는 상황
- 아래 코드는 위의 코드와 동일한데, sushiEaten을 추가하여, 먹은 스시 개수를 확인한다. 
  - 셋다 먹긴 하지만, 공정하지 않다.   

```java
import java.util.concurrent.locks.*;

class Philosopher extends Thread {

    private Lock firstChopstick, secondChopstick;
    private static int sushiCount = 500_000;

    public Philosopher(String name, Lock firstChopstick, Lock secondChopstick) {
        this.setName(name);
        this.firstChopstick = firstChopstick;
        this.secondChopstick = secondChopstick;
    }

    public void run() {
        int sushiEaten = 0;
        while(sushiCount > 0) { 
            firstChopstick.lock();
            secondChopstick.lock();

            try {
                if (sushiCount > 0) {
                    sushiCount--;
                    sushiEaten++;
                    System.out.println(this.getName() + " took a piece! Sushi remaining: " + sushiCount);
                }
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                secondChopstick.unlock();
                firstChopstick.unlock();
            }
        }
        System.out.println(this.getName() + " took " + sushiEaten);
    }
}

public class StarvationDemo {
    public static void main(String[] args) {
        Lock chopstickA = new ReentrantLock();
        Lock chopstickB = new ReentrantLock();
        Lock chopstickC = new ReentrantLock();
        new Philosopher("Barron", chopstickA, chopstickB).start();
        new Philosopher("Olivia", chopstickB, chopstickC).start();
        new Philosopher("Steve", chopstickA, chopstickC).start();
    }
}
```

- 해결 방법
  - 모두의 젓가락을 동일한 순서로 한다.  (Lock Ordering)
  - 결과를 본다면, 비슷한 숫자로 먹은 것을 볼 수 있다. 

```java
new Philosopher("Barron", chopstickA, chopstickB).start();
new Philosopher("Olivia", chopstickA, chopstickB).start();
new Philosopher("Steve", chopstickA, chopstickB).start();
```

- 그런데 만약에 다음과 같이 for문으로 여러 쓰레드의 상황이 놓이면, 몇몇은 또 먹지 못하는 상황이 생긴다. 
  - 참 복잡하다...

```java
public static void main(String[] args) {
    Lock chopstickA = new ReentrantLock();
    Lock chopstickB = new ReentrantLock();
    Lock chopstickC = new ReentrantLock();
    for (int i=0; i<5000; i++) {
        new Philosopher("Barron", chopstickA, chopstickB).start();
        new Philosopher("Olivia", chopstickA, chopstickB).start();
        new Philosopher("Steve", chopstickA, chopstickB).start();
    }
}
```

## Livelock

- 서로 자원 양보하는 상황
- 데드락을 피하기 위해서 만든 방안이 때로는 이 상황을 만들기도 한다.
- 아래코드가 완전히 livelock 상황은 아니지만, 흘끗 보여준다. 
- 코드 실행하면, 끝나지가 않는다. 

```java
import java.util.concurrent.locks.*;
class Philosopher extends Thread {
    private Lock firstChopstick, secondChopstick;
    private static int sushiCount = 500_000;
    public Philosopher(String name, Lock firstChopstick, Lock secondChopstick) {
        this.setName(name);
        this.firstChopstick = firstChopstick;
        this.secondChopstick = secondChopstick;
    }

    public void run() {
        while(sushiCount > 0) {
            firstChopstick.lock();
            if (! secondChopstick.tryLock()) {
                System.out.println(this.getName() + " released their first chopstick.");
                firstChopstick.unlock();
            } else {
                try {
                    if (sushiCount > 0) {
                        sushiCount--;
                        System.out.println(this.getName() + " took a piece! Sushi remaining: " + sushiCount);
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    secondChopstick.unlock();
                    firstChopstick.unlock();
                }
            }
        }
    }
}

public class LivelockDemo {
    public static void main(String[] args) {
        Lock chopstickA = new ReentrantLock();
        Lock chopstickB = new ReentrantLock();
        Lock chopstickC = new ReentrantLock();
        new Philosopher("Barron", chopstickA, chopstickB).start();
        new Philosopher("Olivia", chopstickB, chopstickC).start();
        new Philosopher("Steve", chopstickC, chopstickA).start();
    }
}
```

- 해결 방법은 랜덤 사용. 
  - firstChopstick을 집기 전에 1~3초간 잠들어 있게 한다. 
  - 실행해보면, 철학자들이 스시가 0이 될때까지 잘 돌아가는 것을 확인할 수 있다.  

```java
import java.util.concurrent.locks.*;
import java.util.Random; 

class Philosopher extends Thread {

    private Lock firstChopstick, secondChopstick;
    private static int sushiCount = 500_000;
    private Random rps = new Random(); // 랜덤 추가

    public Philosopher(String name, Lock firstChopstick, Lock secondChopstick) {
        this.setName(name);
        this.firstChopstick = firstChopstick;
        this.secondChopstick = secondChopstick;
    }

    public void run() {
        while(sushiCount > 0) {
            firstChopstick.lock();
            if (! secondChopstick.tryLock()) {
                System.out.println(this.getName() + " released their first chopstick.");
                firstChopstick.unlock();
                try {
                    Thread.sleep(rps.nextInt(3)); // 랜덤 sleep 사용
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            } else {
                try {
                    if (sushiCount > 0) {
                        sushiCount--;
                        System.out.println(this.getName() + " took a piece! Sushi remaining: " + sushiCount);
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    secondChopstick.unlock();
                    firstChopstick.unlock();
                }
            }
        }
    }
}

// 메인은 동일
```















