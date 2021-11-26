

# CH02 - Threads and Processes

## CH02-1

- 작업관리자 켜서, CPU 점유 확인하기. `CPUWaster`에 의해 낭비 확인.
- java.exe를 보면, OS에서 허용된 쓰레드 보다 더 많이 도는데 -> JVM 때문

```java
class CPUWaster extends Thread {
    public void run() {
        while (true) {}
    }
}

public class ThreadProcessDemo {
    public static void main(String args[]) throws InterruptedException {

        // display current information about this process
        Runtime rt = Runtime.getRuntime();
        long usedKB = (rt.totalMemory() - rt.freeMemory()) / 1024 ;
        System.out.format("  Process ID: %d\n", ProcessHandle.current().pid());
        System.out.format("Thread Count: %d\n", Thread.activeCount());
        System.out.format("Memory Usage: %d KB\n", usedKB);

        // start 6 new threads
        System.out.println("\nStarting 6 CPUWaster threads...\n");
        for (int i=0; i<6; i++)
            new CPUWaster().start();

        // display current information about this process
        usedKB = (rt.totalMemory() - rt.freeMemory()) / 1024 ;
        System.out.format("  Process ID: %d\n", ProcessHandle.current().pid());
        System.out.format("Thread Count: %d\n", Thread.activeCount());
        System.out.format("Memory Usage: %d KB\n", usedKB);
    }
}
```

## CH02-2

- 포인트: `join()`를 사용해, 다른 쓰레드에게 자원 쓸 수 있게 내어준다. (기존은 blocked 상태로)
- 자바 쓰레드 상태 6가지: new, runnable, blocked, waiting, timed_waiting, terminated
- priorty 
  - 1: lowest priorty 
  - 10: highest priority

```java
/**
 * Two threads chopping vegetables
 */

class VegetableChopper extends Thread{
    public int vegetable_count = 0;
    public static boolean chopping = true;
    
    public VegetableChopper(String name) {
        this.setName(name);
    }

    public void run() {
        while(chopping) {
            System.out.println(this.getName() + " chopped a vegetable!");
            vegetable_count++;
        }
    }
}

public class ExecutionSchedulingDemo {
    public static void main(String args[]) throws InterruptedException {
        VegetableChopper barron = new VegetableChopper("Barron");
        VegetableChopper olivia = new VegetableChopper("Olivia");

        barron.start();                    // Barron start chopping
        olivia.start();                    // Olivia start chopping
        Thread.sleep(1000);          // continue chopping for 1 second
        VegetableChopper.chopping = false; // stop chopping

        barron.join();
        olivia.join();
        System.out.format("Barron chopped %d vegetables.\n", barron.vegetable_count);
        System.out.format("Olivia chopped %d vegetables.\n", olivia.vegetable_count);
    }
}
```

## CH02-3

- 코드 간단 설명
  - 배런, 올리비아가 같이 떡국 만드는 중. 
  - 배런이 1초 동안 저어주면 배런은 임무 끝. 
  - 그러나 마지막 고명을 준비하는 올리비아는 3초 걸린다고 함. 
  - `olivia.join()`을 해줌으로써 배런(메인쓰레드)가 올리비아를 기다리게 함 (안그럼 고명없는 떡국 먹어야됨!)
- `olivia.getState()`로 쓰레드 상태 확인 

```java
/**
 * Two threads cooking soup
 */

class ChefOlivia extends Thread {
    public void run() {
        System.out.println("Olivia started & waiting for sausage to thaw...");
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Olivia is done cutting sausage.");
    }
}

public class ThreadLifecycleDemo {
    public static void main(String args[]) throws InterruptedException {
         System.out.println("Barron started & requesting Olivia's help.");
        Thread olivia = new ChefOlivia();
        System.out.println("  Olivia state: " + olivia.getState()); // NEW

        System.out.println("Barron tells Olivia to start.");
        olivia.start();
        System.out.println("  Olivia state: " + olivia.getState()); // RUNNABLE

        System.out.println("Barron continues cooking soup.");
        Thread.sleep(1000);
        System.out.println("  Olivia state: " + olivia.getState()); // TIMED_WAITING

        System.out.println("Barron patiently waits for Olivia to finish and join...");
        olivia.join();
        System.out.println("  Olivia state: " + olivia.getState()); // TERMINATED

        System.out.println("Barron and Olivia are both done!");
    }
}
```

## CH02-4

- Runnable 사용시 코드
- 차이는?
  - Thread는 클래스이고, Runnable은 인터페이스
  - 자바 다중상속 안됨으로, Thread 상속 받으면 추가 상속 불가.
  - Runnable 사용이 국룰

```java
/**
 * Two threads cooking soup
 */

class ChefOlivia implements Runnable {
    public void run() {
        System.out.println("Olivia started & waiting for sausage to thaw...");
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Olivia is done cutting sausage.");
    }
}

public class RunnableDemo {
    public static void main(String args[]) throws InterruptedException {
        System.out.println("Barron started & requesting Olivia's help.");
        Thread olivia = new Thread(new ChefOlivia());

        System.out.println("Barron tells Olivia to start.");
        olivia.start();

        System.out.println("Barron continues cooking soup.");
        Thread.sleep(500);

        System.out.println("Barron patiently waits for Olivia to finish and join...");
        olivia.join();

        System.out.println("Barron and Olivia are both done!");
    }
}
```

## CH02-5

-  `olivia.setDaemon(true)` : Deamon Thread 사용 
  - 데몬쓰레드 = 메인의 보조 쓰레드 
- 코드 간단 설명: 배런이 요리를 하는 동안 올리비아는 청소를 하는데, 배런이 요리를 끝내면, 올리비아(데몬쓰레드)도 종료 시켜버린다. (같이 밥먹어야 되니까!)

```java
/**
 * Barron finishes cooking while Olivia cleans
 */

class KitchenCleaner extends Thread {
    public void run() {
        while (true) {
            System.out.println("Olivia cleaned the kitchen.");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
public class DaemonThreadDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread olivia = new KitchenCleaner();
        olivia.setDaemon(true);
        olivia.start();

        System.out.println("Barron is cooking...");
        Thread.sleep(600);
        System.out.println("Barron is cooking...");
        Thread.sleep(600);
        System.out.println("Barron is cooking...");
        Thread.sleep(600);
        System.out.println("Barron is done!");
    }
}
```

## QUIZ

- 1 In most modern multi-core CPUs, cache coherency is usually handled by the `_____` 
  - processor hardware
- 2 A key advantage of distributed memory architectures is that they are `_____` than shared memory systems.
  - scalable
- 3 Modern multi-core PCs fall into the `_____` classification of Flynn's Taxonomy.
  - MIMD
- 4 The four classifications of Flynn's Taxonomy are based on the number of concurrent `_____` streams and `_____` streams available in the architecture.
  - instruction; data
- 5 A Java thread can be turned into a daemon thread after it has been started.
  - True
  - False (답)
- 6 If a daemon thread in Java creates another thread, that child thread will `_____`.
  - decided whether or not it will be a daemon thread on its own
  - always be a daemon thread because all new children are automatically daemons
  - inherit the daemon status of its parent thread (답)
  - only be a daemon thread if its parent calls the setDaemon() method before starting it
- 7 Why would you use daemon threads to handle continuous background tasks?
  - Daemon threads will automatically spawn additional helper threads as needed when their workload increases.
  - Daemon threads always execute at the lowest priority which makes them ideal for background tasks.
  - The daemon thread will not prevent the program from terminating when the main thread is finished. (답)
  - You should never trust a daemon. They're mischievous!
- 8 Why can it be risky to use a daemon thread to perform a task that involves writing data to a log file?
  - Daemons only write their own name, so the log will just say "Daemon, Daemon, Daemon, etc."
  - Daemon threads cannot read or write files.
  - The log file could end up with multiple, duplicate entries.
  - The log file could be corrupted. (답)
    - 설명: A daemon thread will be abruptly terminated when the main thread finishes. If that occurs during a write operation the file could be corrupted.
- 9 Why would ThreadA call the `ThreadB.join()` method?
  - ThreadB is blocked so ThreadA needs to tell it to continue executing.
  - ThreadB needs to wait until after ThreadA has terminated to continue.
  - ThreadA needs to wait until after ThreadB has terminated to continue. (답)
  - ThreadA needs to terminate ThreadB immediately.
- 10 thread that calls the join method on another thread will enter the `_____` state until the other thread finishes executing.
  - Blocked
- 11 Why do you have to start a thread after creating it?
  - Threads do not automatically run when instantiated. (답)
- 12 You can safely expect threads to execute in the same relative order that you create them.
  - TRUE
  - FALSE (답)
- 13 It is possible for two tasks to execute `_____` using a single-core processor.
  - in parallel
  - concurrently or in parallel
  - concurrently (답)
- 14 Which of these applications would benefit the most from parallel execution? 어렵다
  - math library for processing large matrices (답)
    - 설명: Mathematical operations on matrices are computationally intensive and therefore well suited to benefit from parallel execution.
  - graphical user interface (GUI) for an accounting application. 
    - 설명: The GUI should be structured for concurrency, but as an I/O bound task, it will generally run as well with parallel execution as without.
  - tool for downloading multiple files from the Internet at the same time. 
    - 설명: The rate at which files can be downloaded over the network will act as the limiting factor in this application. It is I/O bound and therefore will not benefit significantly from parallel execution.
  - system logging application that frequently writes to a database
- 15 A hyperthreaded processor with eight logical cores will usually provide `_____` performance compared to a regular processor with eight physical cores.
  - lower (답)
  - equivalent
  - higher
- If you run multiple Java applications at the same time, they will execute in `_____`.
  - the same JVM thread
  - separate JVM threads
  - the same JVM process
  - separate JVM processes (답)
    - When you run a Java application, it executes within its own instance of the Java Virtual Machine (or JVM), and the operating system treats that instance of the JVM as its own independent process.
- Processes `_____` than threads.
  - require more overhead to create (답)
    - Rule of thumb: 멀티프로세스보단 멀티쓰레드 사용 
  - are faster to switch between
  - are considered more "lightweight"
  - are simpler to communicate between
- A `_____` contains one or more `_____`.
  - thread; processes 
  - process; threads (답)
  - process; other processes
  - thread; other threads
- Every thread is independent and has its own separate address space in memory.
  - FALSE (답)
  - TRUE
- Concurrency vs Parallelism
  - Concurrency : 사람2 도마1 칼1 (즉, 자원 나눠씀)
  - Parallelism: 사람2 도마2 칼2


























