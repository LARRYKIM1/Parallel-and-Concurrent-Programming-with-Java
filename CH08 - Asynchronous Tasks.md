# CH08 - Asynchronous Tasks

- Thread pool: 기존에 쓰레드를 재사용하자는 취지
- 자바의 Executor service

![image](https://user-images.githubusercontent.com/49010295/144709790-97eb353a-b3e7-4ee8-933a-ebf2a5327082.png)

- Executors 클래스에서 자주 사용되는 메소드
  - newSingleThreadExecutor() - 싱글
  - newFixedThreadExecutor(int nThreads) - 멀티

# 1 Thread Pool

![image](https://user-images.githubusercontent.com/49010295/144710891-0b773c0f-0f0d-4d9e-9716-c279e351699f.png)

샐러드를 만든다고 가정. 

다음 코드를 쓰레드풀을 만들어서 처리해보자. 

```java
class VegetableChopper extends Thread {
    public void run() {
        System.out.println(Thread.currentThread().getName() + " chopped a vegetable!");
    }
}
// 100개의 Task(스레드)가 작업 위해 대기중
public class ThreadPoolDemo {
    public static void main(String args[]) {
        for (int i=0; i<100; i++)
            new VegetableChopper().start();
    }
}
```

결과

```
Thread-75 chopped a vegetable!
Thread-71 chopped a vegetable!
Thread-74 chopped a vegetable!
Thread-73 chopped a vegetable!
Thread-76 chopped a vegetable!
Thread-28 chopped a vegetable!
Thread-14 chopped a vegetable!
Thread-77 chopped a vegetable!
//... 생략 ...
```

쓰레드풀 사용 코드 

```java
class VegetableChopper extends Thread {
    public void run() {
        System.out.println(Thread.currentThread().getName() + " chopped a vegetable!");
    }
}

public class ThreadPoolDemo {
    public static void main(String args[]) {
        int numProcs = Runtime.getRuntime().availableProcessors(); // 이용가능 프로세서수
        ExecutorService pool = Executors.newFixedThreadPool(numProcs);
        for (int i=0; i<100; i++) 
            pool.submit(new VegetableChopper()); // pool.sumbit(작업)
        pool.shutdown(); // 이거 없으면, Executor가 계속 살아있음.  
    }
}
```

결과

```
pool-1-thread-3 chopped a vegetable!
pool-1-thread-6 chopped a vegetable!
pool-1-thread-8 chopped a vegetable!
pool-1-thread-7 chopped a vegetable!
pool-1-thread-7 chopped a vegetable!
pool-1-thread-7 chopped a vegetable!
//... 생략 ...
```



# 2 Future

- Callable<V> 인터페이스
- 비동기 처리 

```java
/**
 * Check how many vegetables are in the pantry
 */

import java.util.concurrent.*;

class HowManyVegetables implements Callable {
    public Integer call() throws Exception {
        System.out.println("Olivia is counting vegetables...");
        Thread.sleep(3000);
        return 42;
    }
}

public class FutureDemo {
    public static void main(String args[]) throws ExecutionException, InterruptedException {
        System.out.println("Barron asks Olivia how many vegetables are in the pantry.");
        ExecutorService executor = Executors.newSingleThreadExecutor();
        Future result = executor.submit(new HowManyVegetables());
        System.out.println("Barron can do other things while he waits for the result...");
        System.out.println("Olivia responded with " + result.get()); // 여기서 3초 기다리게 된다. 
        executor.shutdown();
    }
}
```

# 3 Divide and Conquer

- RecursiveTask<> 클래스 
  - 
- 자바의 Fork Join 프레임워크
  - 효율적인 병렬처리를 위해 자바 7에서 소개된 새로운 기능
  - 가르고 - 합친다
  - fork 순간 다른 쓰레드에서 처리하게 한다. 

```java
/**
 * Recursively sum from 0 to 10억.
 */

import java.util.concurrent.*;

class RecursiveSum extends RecursiveTask<Long> {

    private long lo, hi;

    public RecursiveSum(long lo, long hi) {
        this.lo = lo;
        this.hi = hi;
    }

    protected Long compute() {
        if (hi-lo <= 100_000) { // base case threshold
            long total = 0;
            for (long i = lo; i <= hi; i++)
                total += i;
            return total;
        } else {
            long mid = (hi+lo)/2; // middle index for split
            RecursiveSum left = new RecursiveSum(lo, mid);
            RecursiveSum right = new RecursiveSum(mid+1, hi);
            left.fork(); // forked thread computes left half
            return right.compute() + left.join(); // current thread computes right half
        }
    }
}

public class DivideAndConquerDemo {
    public static void main(String args[]) {
        ForkJoinPool pool = ForkJoinPool.commonPool();
        // RecursiveTask 실행
        Long total = pool.invoke(new RecursiveSum(0, 1_000_000_000)); 
        pool.shutdown();
        System.out.println("Total sum is " + total);
    }
}
```



# QUIZ

- When implementing a recursive divide-and-conquer algorithm in Java, why should you use a ForkJoinPool instead of simply creating new threads to handle each subproblem?
  - The ForkJoinPool manages a thread pool to execute its ForkJoinTasks, which reduces the overhead of thread creation.
- What does a divide-and-conquer algorithm do when it reaches the base case?
  - Stop subdividing the current problem and solve it.
- What is the difference between Java's Callable and Runnable interfaces?
  - The Callable interface's call() method returns a result object but the Runnable interface's run() method does not.
- What is the purpose of a future?
  - It serves as a placeholder to access a result that may not been computed yet.
- When using a thread pool in Java, the `_____` assigns submitted tasks to specific threads within the available pool to execute.
  - thread pool executor service
- Why are thread pools useful?
  - They reuse threads to reduce the overhead that would be required to create a new, separate thread for every concurrent task.
- What does a work-to-span ratio less than one indicate?
  - The work-to-span ratio cannot be less than one.
- What is a program's "span"?
  - sum of the time for all task nodes along the critical path
  - ![image](https://user-images.githubusercontent.com/49010295/144710941-f6a4d538-076d-4533-81d4-b34b9e59e64a.png)
    ![image](https://user-images.githubusercontent.com/49010295/144710955-42f69721-8e51-454e-aca3-a02c293847a0.png)
- What is a program's "critical path"?
  - longest series of sequential operations through the program
- What is a program's "work"?
  - sum of the time for all task nodes in a computational graph
- Why are computational graphs useful?
  - They help to identify opportunities for parallel execution.









