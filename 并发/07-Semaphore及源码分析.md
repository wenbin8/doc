# Semaphore及源码分析

## 使用案例

semaphore也就是我们常说的信号灯，semaphore可以控制同时访问的线程个数，通过acquire获取一个许可，如果没有就等待，通过release释放一个许可。有点类似限流的作用。叫信号灯的原因也和他的用处有关，比如某商场就5个停车位，每个停车位只能停一辆车，如果这个时候来了10辆车，必须要等前面有空的车位才能进入。

## 使用场景

Semaphore比较常见的就是用来做限流操作了

## 源码分析

从Semaphore的功能来看，我们基本能猜测到它底层实现一定是基于AQS共享锁，因为需要实现多个线程共享一个令牌池。

创建Semaphore实例的时候，需要一个参数permits，这个基本上可以确定是设置给AQS的state的，然后每个线程调用acquire的时候，执行state=teate-1，release的时候执行state=stete+1，当然，acquire的时候，如果state=0，说明没有了资源了，需要等待其他线程release。

```java
package juc;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class SemaphoreTest {
    public static void main(String[] args) {
        Semaphore semaphore = new Semaphore(5);

        for (int i = 0; i < 10; i++) {
            new Car(i,semaphore).start();
        }
    }

    static class Car extends Thread {
        private int num;
        private Semaphore semaphore;

        public Car(int num, Semaphore semaphore) {
            this.num = num;
            this.semaphore = semaphore;
        }

        @Override
        public void run() {
            try {
                semaphore.acquire(); // 获取一个许可
                System.out.println("第" + num +"占用一个停车位。");
                TimeUnit.SECONDS.sleep(2);
                System.out.println("第" + num + "俩车走喽");
                semaphore.release();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

Semaphore分为公平策略和非公平策略

### FairSync

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = 2014338818796000944L;

    FairSync(int permits) {
        super(permits);
    }

    protected int tryAcquireShared(int acquires) {
        for (;;) {
            // 与公平锁的区别就在于是不是会先判断是否有现成在排队，然后才进行CAS减操作
            if (hasQueuedPredecessors())
                return -1;
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }
}
```

### NofairSync 

通过对比发现公平和非公平的区别就在于是否多了一个hasQueuedPreadecessors判断

```java
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = -2694183684443567898L;

    NonfairSync(int permits) {
        super(permits);
    }

    protected int tryAcquireShared(int acquires) {
        return nonfairTryAcquireShared(acquires);
    }
}
```

由于后面的代码和 CountDownLatch 的是完全一样，都是 基于共享锁的实现，所以也就没必要再花时间来分析了。 