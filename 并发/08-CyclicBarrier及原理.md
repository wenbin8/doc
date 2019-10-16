# CyclicBarrier及原理

CyclicBarrier的字面意思是可循环使用（Cyclic）的屏障（Barrier）。它要做的事情是，让 一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续工作。

CyclicBarrier默认的构造方法是CyclicBarrier默认的构造方法是CyclicBarrier(int parties)，其参数表示屏障来接的线程数量，每个线程调用await方法告诉CyclicBarrier当前线程已经到达了屏障，然后当前线程被阻塞。

## 使用场景

当存在需要所有的子任务都完成时，才执行主任务，这个时候就可以选择使用CyclicBarrier。

## 使用案例

```java
public class DataImportThread extends Thread {

    private CyclicBarrier cyclicBarrier;

    private String path;

    public DataImportThread(CyclicBarrier cyclicBarrier, String path) {
        this.cyclicBarrier = cyclicBarrier;
        this.path = path;
    }

    @Override
    public void run() {
        System.out.println("开始导入：" + path + "位置的数据");
        try {
            cyclicBarrier.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }
    }
}
```

```java
public class CycliBarrierDemo extends Thread {
    @Override
    public void run() {
        System.out.println("开始进行数据分析");
    }

    public static void main(String[] args) {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(3, new CycliBarrierDemo());

        new Thread(new DataImportThread(cyclicBarrier, "file 1")).start();
        new Thread(new DataImportThread(cyclicBarrier, "file 2")).start();
        new Thread(new DataImportThread(cyclicBarrier, "file 3")).start();
    }
}
```

执行结果：

```verilog
开始导入：file 1位置的数据
开始导入：file 3位置的数据
开始导入：file 2位置的数据
开始进行数据分析
```

## 注意点

1. 对于指定计数值parties，若由于某种原因，没有足够的线程调用CyclicBarrier的await，则所有调用await的线程都会被阻塞。
2. 同样的CyclicBarrier也可以调用await(timeout,unit)，设置超时时间，在设定时间内，如果没有足够线程到达，则接触阻塞状态，继续工作。
3. 通过reset重置计数，会使得进入await的线程出现BrokenBarrierException；
4. 如果采用是CyclicBarrier(int parties,Runable barrierAction)构造方法，执行barrierAction操作的是最后一个到达的线程

## 实现原理

CyclicBarrier相比CountDownLatch来说，要简单很多，源码实现是基于ReentrantLock和Condition的组合使用。CyclicBarrier和CountDownLatch很像，只是CyclicBarrier可以有布置一个栅栏，因为它的栅栏（Barrier）可以重复使用（Cyclic）

CyclicBarrier还提供了一些其他有用的方法，比如getNumberWaiting()方法可以获得CyclicBarrier阻塞的线程数量，isBroken()方法用来了解阻塞的线程是否被中断；

CountDownLatch允许一个或多个线程等待一组事件的产生，而CyclicBarrier用于等待其他线程运行到栅栏位置。










