

# 1 Race Condition

- Data Race와 Race Condition 차이는?
  - 실전에서, 많이 data race 때문에 Race Condition 가 발생하곤 한다.
  - 근데 서로 dependent 하지 않다. 
- 계산 순서가 뒤바뀌면, 은 전형 다른 값이나온다. 
  - 예를들어, 올리비아는 파티를 위해 기존에 1개있던 chips에 3개를 더사자고 더하고 배런은 거기에 넉넉히 먹기 위해 2배를 더한다. 근데, 순서가 바뀌게 되면, (1+3)x2 와 (1x2)+3
- 불행히도, Race Condition 완전히 감지하기 위한 방법은 없다.
- 대응 방법: use sleep statement to modify timing and execution order 
- 코드설명: 각 5개의 쓰레드를 가진 올리비아와 배런은 파티를 위해 chips를 사려고 한다. 현재 집에 1개가 있다. 배런은 2개씩 곱해주고, 올리비아는 3개씩 더한다. 
  - 특징: ReentrantLock을 사용하여 data race는 발생하지 않지만, 배런과 올리비아의 상대적인 실행 순서 변경으로 인해 race condition은 발생하는 상황

```java
class Shopper extends Thread {
    public static int bagsOfChips = 1; // start with one on the list
    private static Lock pencil = new ReentrantLock();
    public Shopper(String name) {
        this.setName(name);
    }

    public void run() {
        if (this.getName().contains("Olivia")) {
            pencil.lock();
            try {
                bagsOfChips += 3;
                System.out.println(this.getName() + " ADDED three bags of chips.");
            } finally {
                pencil.unlock();
            }
        } else { // "Barron"
            pencil.lock();
            try {
                bagsOfChips *= 2;
                System.out.println(this.getName() + " DOUBLED the bags of chips.");
            } finally {
                pencil.unlock();
            }
        }
    }
}

public class RaceConditionDemo {
    public static void main(String args[]) throws InterruptedException {
        // create 10 shoppers: Barron-0...4 and Olivia-0...4
        Shopper[] shoppers = new Shopper[10];
        for (int i = 0; i < shoppers.length / 2; i++) {
            shoppers[2 * i] = new Shopper("Barron-" + i); // 5명 배런 생성
            shoppers[2 * i + 1] = new Shopper("Olivia-" + i); // 5명 올리비아 생성
        }
        for (Shopper s : shoppers)
            s.start();
        for (Shopper s : shoppers)
            s.join();
        System.out.println("We need to buy " + Shopper.bagsOfChips + " bags of chips.");
    }
}
```

- 해결방법: Barrier

# 2 Barrier

- Barrier란?
- 코드설명: 결과값이 계속 다르게 나오는 위와 다르게 bagsOfChips가 계속 512개로 나오는 것 확인가능. 
  - `CyclicBarrier.await()` 사용
    - 다른메서드
      - int getParties(): barrier를 넘어야되는 모든 쓰레드들의 개수
      - int getNumberWaiting(): barrier에서 현재 기다리는 쓰레드 개수
      - void reset(): barrier를 초기 상태로 만든다
      - boolean isBroken(): broken 스레드가 있는지

```java
class Shopper extends Thread {
    public static int bagsOfChips = 1; 
    private static Lock pencil = new ReentrantLock();
    private static CyclicBarrier fistBump = new CyclicBarrier(10); // Barrier 추가 

    public Shopper(String name) {
        this.setName(name);
    }

    public void run() {
        if (this.getName().contains("Olivia")) {
            pencil.lock();
            try {
                bagsOfChips += 3;
                System.out.println(this.getName() + " ADDED three bags of chips.");
            } finally {
                pencil.unlock();
            }
            try {
                fistBump.await(); // 추가
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
        } else { // "Barron"
            try {
                fistBump.await(); // 추가
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
            pencil.lock();
            try {
                bagsOfChips *= 2;
                System.out.println(this.getName() + " DOUBLED the bags of chips.");
            } finally {
                pencil.unlock();
            }
        }
    }
}

// main 동일 
```

- 

# 3 CountDownLatch

- 멀티 쓰레드에서 모든 동작이 끝나기를 기다려야 하는 상황이라면 join 을 사용하기 조금 난감하다. 이럴 땐 CountDownLatch 를 사용하면 편리하다.


- 메서드
  - CountDownLatch(value): count값 초기화.
  - await(): count 값이 0이 될때까지 기달.
  - countDown(): count 값 내리기.
- CyclicBarrier와 CountDownLatch 차이
  - CyclicBarrier: 특정 스레드 수가 기다리고 있을때 릴리즈
  - CountDownLatch: count 값이 0에 도달했을때 릴리즈

```java
class Shopper extends Thread {

    public static int bagsOfChips = 1; // start with one on the list
    private static Lock pencil = new ReentrantLock();
    private static CountDownLatch fistBump = new CountDownLatch(5);

    public Shopper(String name) {
        this.setName(name);
    }

    public void run() {
        if (this.getName().contains("Olivia")) {
            pencil.lock();
            try {
                bagsOfChips += 3;
                System.out.println(this.getName() + " ADDED three bags of chips.");
            } finally {
                pencil.unlock();
            }
            fistBump.countDown(); // 추가
        } else { // "Barron"
            try {
                fistBump.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            pencil.lock();
            try {
                bagsOfChips *= 2;
                System.out.println(this.getName() + " DOUBLED the bags of chips.");
            } finally {
                pencil.unlock();
            }
        }
    }
}

// 메인 동일
```

- 주의할점: 만약 위의 코드 CountDownLatch에 capacity를 10을 줬다하면 프로그램이 멈춰버린다. 이유 생각해보기.
- 흔한 사용 방법  CountDownLatch 
  - count를 1로 초기화
    - 간단한 on/off 게이트
  - count를 N으로 초기화
    - N 쓰레드가 특정 업무를 끝날때 까지 기다린다.
    - or 특정 업무가 N번 진행될때까지 기다린다. 

# QUIZ

- Which scenario describes a potential use case for a CountDownLatch initialized to a count value of 1?
  - Multiple threads need to wait until after one other thread completes some action to continue
- What is true about Java's CountDownLatch?
  - It releases when its counter value reaches zero.
- A thread that needs to execute a section of code before the barrier should call the CyclicBarrier's `_____` method `_____` executing the code. 
  - await(); after
- Barriers can be used to control `_____`.
  - the relative order in which threads execute certain operations
- Which of these is responsible for causing a race condition?
  - the OS execution scheduler (The order in which the OS schedules threads to execute is non-deterministic)
- A race condition `_____` a data race.
  - can occur independently of
- Which scenario creates the potential for a race condition to occur?
  - A single-threaded program is competing with other processes for execution time on the CPU.
  - Two threads are concurrently reading and writing the same shared variable.
  - One thread is modifying a shared variable while another thread concurrently reads its value.
  - The order in which two threads execute their respective operations will change the output. (픽)

