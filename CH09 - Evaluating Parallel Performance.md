# CH09 - Evaluating Parallel Performance

- Weak scaling과 Strong scaling
  - weak: 더 많은 프로세스들을 추가하고, 프로세스 당 업무량을 유지하여, 더 큰 문제를 해결하도록 하는 것.
    - 예, 10개 작업을 2명이서 각각 10개씩 했을때 걸리는 시간이 어떻게 달라지는지
  - strong:
    - 예, 10개 작업을 2명이서 같이 작업 했을때 걸리는 시간이 어떻게 달라지는지

# 1 Amdahl's law

- 병렬 처리했을때 속도 증가율 
- 회사의 자원이 병렬처리를 통해 이득을 보고 있는지 확인 가능해보인다.  
- 오직 병렬 프로세싱을 엄청 많이 하는 시스템들만이 프로세스의 수가 많을때 큰 이득을 본다. (병렬작업 없는데 프로세스 수 늘리면 자원낭비)
- 공식

![image](https://user-images.githubusercontent.com/49010295/144711544-bb1903b2-6e28-4178-bfe0-cb0bb782f90d.png)

병렬 처리 portion에 따른 프로세서가 많아질 수록 속도 증가율 (병렬 부분이 줄어들 수록 속도는 급감)

![image](https://user-images.githubusercontent.com/49010295/144711666-4e8a4a08-597a-40f9-9abd-0e73438d1df0.png)

# 2 속도 측정 

- speedup = 순차처리로 걸린시간 / 병렬처리로 걸린시간
- 속도만으로 부족해서 efficiency도 측정한다. 
- efficiency = speedup / 프로세스개수 

```JAVA
/**
 * Measure the speedup of a parallel algorithm = 순차와 병렬 시간 비교 
 */

import java.util.concurrent.*;

// parallel implementation
class RecursiveSum extends RecursiveTask<Long> {

    private long lo, hi;

    // constructor for recursive instantiations
    public RecursiveSum(long lo, long hi) {
        this.lo = lo;
        this.hi = hi;
    }

    // returns sum of numbers between lo and hi
    protected Long compute() {
        if (hi-lo < 100_000) { // base case threshold
            long total = 0;
            for (long i=lo; i<=hi; i++)
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

public class MeasureSpeedupDemo {

    // sequential implementation
    private static long sequentialSum(long lo, long hi) {
        long total = 0;
        for (long i=lo; i<=hi; i++)
            total += i;
        return total;
    }

    public static void main(String args[]) {
        final int NUM_EVAL_RUNS = 10;
        final long SUM_VALUE = 1_000_000_000L;

        System.out.println("Evaluating Sequential Implementation...");
        long sequentialResult = sequentialSum(0, SUM_VALUE); // "warm up"
        double sequentialTime = 0;
        for(int i=0; i<NUM_EVAL_RUNS; i++) {
            long start = System.currentTimeMillis();
            sequentialSum(0, SUM_VALUE);
            sequentialTime += System.currentTimeMillis() - start;
        }
        sequentialTime /= NUM_EVAL_RUNS;

        System.out.println("Evaluating Parallel Implementation...");
        ForkJoinPool pool = ForkJoinPool.commonPool();
        long parallelResult = pool.invoke(new RecursiveSum(0, SUM_VALUE)); // "warm up"
        pool.shutdown();
        double parallelTime = 0;
        for(int i=0; i<NUM_EVAL_RUNS; i++) {
            long start = System.currentTimeMillis();
            pool = ForkJoinPool.commonPool();
            pool.invoke(new RecursiveSum(0, SUM_VALUE));
            pool.shutdown();
            parallelTime += System.currentTimeMillis() - start;
        }
        parallelTime /= NUM_EVAL_RUNS;

        // display sequential and parallel results for comparison
        if (sequentialResult != parallelResult)
            throw new Error("ERROR: sequentialResult and parallelResult do not match!");
        System.out.format("Average Sequential Time: %.1f ms\n", sequentialTime);
        System.out.format("Average Parallel Time: %.1f ms\n", parallelTime);
        System.out.format("Speedup: %.2f \n", sequentialTime/parallelTime);
        System.out.format("Efficiency: %.2f%%\n", 100*(sequentialTime/parallelTime)/Runtime.getRuntime().availableProcessors());
    }
}
```

결과

```
Evaluating Sequential Implementation...
Evaluating Parallel Implementation...
Average Sequential Time: 823.4 ms
Average Parallel Time: 245.2 ms
Speedup: 3.36 
Efficiency: 41.98%
```

`if (hi-lo < 100_000)`부분을 `if (hi-lo < 100)`로 바꿀경우 결과 

```
Evaluating Sequential Implementation...
Evaluating Parallel Implementation...
Average Sequential Time: 782.7 ms
Average Parallel Time: 462.4 ms
Speedup: 1.69 
Efficiency: 21.16%
```



# 추가

Latency와 Throughput 계산 방법 

![image](https://user-images.githubusercontent.com/49010295/144711397-af260145-fb59-486c-b3d4-6a110c9b4df0.png)

![image](https://user-images.githubusercontent.com/49010295/144711404-c8ddc2c5-5d1e-4631-86c1-8048dd1adf00.png)

![image](https://user-images.githubusercontent.com/49010295/144711413-fc10c9b4-a849-428d-a95e-6d6d80bef9fe.png)

속도 계산 방법 

![image](https://user-images.githubusercontent.com/49010295/144711443-20d301f3-42fc-4e37-8c17-5da178c0730a.png)

![image](https://user-images.githubusercontent.com/49010295/144711471-d36018ef-46ab-4aa3-8eb3-c1b313d185f3.png)

![image](https://user-images.githubusercontent.com/49010295/144711473-c002820f-529a-47cc-a39a-15af1f094a51.png)



# QUIZ

- Why should you average the execution time across multiple runs when measuring a program's performance?
  - The execution time will vary from run-to-run depending on how the operating system chooses to schedule your program.
- What does calculating a program's efficiency (speedup divided by number of parallel processors) provide an indicator of?
  - how well the parallel processing resources are being utilized
- Amdahl's Law calculates a(n) `_____` for the overall speedup that parallelizing a program will achieve.
  - upper limit
- If 70% of a program is parallelizable so that using a four-core processor will produce a 4x speedup for that portion of the code, what is the maximum overall speedup the program can achieve?
  - 2.1 
    - 1/(1-0.7+(0.7/4))
- Increasing the number processors with a fixed total problem size leverages strong scaling to accomplish `_____` in `_____`.
  - same work; less time
- Increasing the number processors with a fixed problem size per processor leverages weak scaling to accomplish `_____` in `_____`.
  - more work; same time
- What is a program's "speedup"?
  - ratio of sequential execution time to the parallel execution time with some number of processors
- What is a program's "throughput"?
  - number of tasks that can be executed in a certain amount of time
- Which of these describes a program's "latency"?
  - amount of time a task takes to execute





