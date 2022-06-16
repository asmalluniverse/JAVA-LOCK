### 一、CountDownLatch
它是一个同步辅助器，允许一个或多个线程一直等待，直到其他线程执行的操作全部完成，继续执行。

##### 1.构造方法

构造方法，会传入一个 count 值，用于计数,可以理解为需要等待的线程数量
```java
public CountDownLatch(int count) {
	if (count < 0) throw new IllegalArgumentException("count < 0");
	this.sync = new Sync(count); //创建同步队列，并设置初始计数器值
}
```
##### 2.CountDownLatch类解析
1. 常用方法
```java
public void await() throws InterruptedException {
	sync.acquireSharedInterruptibly(1);
}

public void countDown() {
	sync.releaseShared(1);
}
```
当一个线程调用await方法时，就会阻塞当前线程。每当有线程调用一次 countDown 方法时，计数就会减 1。当 count 的值等于 0 的时候，被阻塞的线程才会继续运行。

2. 其他方法
CountDownLatch类中一共提供了四个方法，另外两个方法如下：
1. 可以在调用await方法时传递超时时间
```java
public boolean await(long timeout, TimeUnit unit)
	throws InterruptedException {
	return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
```

2. 获取等待线程的数量
```java
public long getCount() {
	return sync.getCount();
}
```


##### 3.如何使用
1. 假设有如下场景：

现在设想一个场景，公司项目，线上出现了一个紧急 bug，被客户投诉，领导焦急的过来，想找人迅速的解决这个 bug 。
那么，一个人解决肯定速度慢啊，于是叫来张三和李四，一起分工解决。终于，当他们两个都做完了自己所需要做的任务之后，领导才可以答复客户，客户也就消气了（没办法啊，客户是上帝嘛）。
于是，我们可以设计一个 Worker 类来模拟单个人修复 bug 的过程，主线程就是领导，一直等待所有 Worker 任务执行结束，主线程才可以继续往下走。


```java
package com.example.countdown;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.CountDownLatch;


public class CountDownTest {

    static SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(2);
        new Thread(new Worker("张山",2000, countDownLatch)).start();
        new Thread(new Worker("李四",3000, countDownLatch)).start();
        long startTime = System.currentTimeMillis();

        countDownLatch.await();

        System.out.println("bug全部解决，领导可以给客户交差了，任务总耗时："+ (System.currentTimeMillis() - startTime));

    }

    static class Worker extends Thread{
        String name;
        int workTime;
        CountDownLatch latch;

        public Worker(String name, int workTime, CountDownLatch latch) {
            this.name = name;
            this.workTime = workTime;
            this.latch = latch;
        }

        @Override
        public void run() {
            System.out.println(name + " 开始修复Bug ，当前时间：" + simpleDateFormat.format(new Date()));
            doWork();
            System.out.println(name + " 完成修复Bug ，当前时间：" + simpleDateFormat.format(new Date()));
            latch.countDown();
        }

        private void doWork() {
            try {
                //模拟工作耗时
                Thread.sleep(workTime);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

```

##### 4.源码解析
首先我们创建一个CountDownLatch对象，并通过构造方法设置state的值，代表锁的重入次数。然后调用await方法的时候，所有线程都会包装成一个Node节点，节点状态是shared状态。 每个线程调用 countDown() 的时候 ,会对 state计数器减一, 如果多次调用countDown(),直到将state计数器减到0, 那么 就会唤醒队列中，被await()阻塞住的线程(Node节点)，来抢锁。抢锁成功的话, 就会接着 唤醒下一个 被await方法阻塞住的SHARED线程节点, 下一个线程被唤醒后 又能抢到锁，又要唤醒自己的下一个。直到所有被await阻塞的await节点线程都被唤醒,直到队列为空。  

构造方法就是设置state的值这里就不做分析，下面直接分析await方法和countDown方法。  
此时假设CountDownLatch countDownLatch = new CountDownLatch(2);阻塞两个线程A,B,主线程调用await方法。

1. await()

```java
countDownLatch.await();

public void await() throws InterruptedException {
	sync.acquireSharedInterruptibly(1);
}

public final void acquireSharedInterruptibly(int arg)
		throws InterruptedException {
	if (Thread.interrupted())
		throw new InterruptedException();
	//根据state变量的值返回true false 
	if (tryAcquireShared(arg) < 0)
		doAcquireSharedInterruptibly(arg);
}

protected int tryAcquireShared(int acquires) {
    //getState获取设置的参数，即构造方法中设置的count,此时为2 返回-1，if条件满足，调用doAcquireSharedInterruptibly(arg);方法 
	return (getState() == 0)   1 : -1;
}

private void doAcquireSharedInterruptibly(int arg) （aqs中的逻辑）
	throws InterruptedException {
	//把当前节点加入到aqs队列中（aqs中的逻辑）
	final Node node = addWaiter(Node.SHARED);
	boolean failed = true;
	try {
		for (;;) {
		    //获取当前节点的前置节点
			final Node p = node.predecessor();
			if (p == head) {
			    //唤醒头结点的下一个节点
				int r = tryAcquireShared(arg);
				if (r >= 0) {
				    //setHeadAndPropagate 方法负责将自旋等待或被 LockSupport //阻塞的线程唤醒。即通过下面parkAndCheckInterrupt()方法阻塞后，等待其他线程的唤醒
					//因为这里是一个自旋操作，如果此时r>=0 说明count的值已经减到0
					setHeadAndPropagate(node, r);
					p.next = null; // help GC
					failed = false;
					return;
				}
			}
			//经过tryAcquireShared后，state还不为0（r=-1），就会到这里，第一次调用shouldParkAfterFailedAcquire方法的时候，waitStatus是0，走else方法
			//那么node的waitStatus就会被置为SIGNAL，返回true,然后调用parkAndCheckInterrupt方法，就会调用LockSupport的park方法把当前线程阻塞
			if (shouldParkAfterFailedAcquire(p, node) &&
				parkAndCheckInterrupt())
				throw new InterruptedException();
		}
	} finally {
		if (failed)
			cancelAcquire(node);
	}
}

```


2. countDown()方法解析
```java 
//假设此时线程A先执行了，然后线程B再次执行
public void countDown() {
	sync.releaseShared(1);
}

```

```java
public final boolean releaseShared(int arg) {
	// 若tryReleaseShared返回true，表示count经过这次释放后，等于0了，于是执行doReleaseShared
	//线程A先调用该方法，此时state的值为2，调用tryReleaseShared会进行-1操作，不等于零返回false
	//然后线程B再次执行,此时state的值为1，调用tryReleaseShared会进行-1操作，减一后等于0，所有会调用doReleaseShared方法。
	if (tryReleaseShared(arg)) {
		doReleaseShared();
		return true;
	}
	return false;
}
```

```java 
protected boolean tryReleaseShared(int releases) {
	// Decrement count; signal when transition to zero
	//自旋操作，如果一开始就为零，也返回false没必要执行doReleaseShared 方法， 
	for (;;) {
		int c = getState();
		if (c == 0)
			return false;
		int nextc = c-1;
		if (compareAndSetState(c, nextc))
			return nextc == 0;
	}
}

```

```java 
//计数器被减到0，那么就会执行这个方法，唤醒阻塞的线程,抢锁后执行业务代码
private void doReleaseShared() {
	for (;;) {
		Node h = head;
		if (h != null && h != tail) {
			int ws = h.waitStatus;
			//前面阻塞队列中的头结点的状态是SIGNAL，会走到If条件中
			if (ws == Node.SIGNAL) {
				if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
					continue;            // loop to recheck cases
				unparkSuccessor(h); //唤醒头结点的下一个节点，此时会继续执行doAcquireSharedInterruptibly方法
			}
			else if (ws == 0 &&
					 !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
				continue;                // loop on failed CAS
		}
		if (h == head)                   // loop if head changed
			break;
	}
}


private void unparkSuccessor(Node node) {

	Node s = node.next;
	if (s == null || s.waitStatus > 0) {
		s = null;
		for (Node t = tail; t != null && t != node; t = t.prev)
			if (t.waitStatus <= 0)
				s = t;
	}
	if (s != null)
		LockSupport.unpark(s.thread); //唤醒节点 此时接着执行 doAcquireSharedInterruptibly中被阻塞的线程
}

```

```java
private void doAcquireSharedInterruptibly(int arg)
	throws InterruptedException {
	final Node node = addWaiter(Node.SHARED);
	boolean failed = true;
	try {
		for (;;) {
			final Node p = node.predecessor();
			if (p == head) {
			    //此时count 减为0 r返回1调用setHeadAndPropagate方法
				int r = tryAcquireShared(arg);
				if (r >= 0) {
					//唤醒当前节点后，继续调用doReleaseShared唤醒阻塞队列中的其他节点
					setHeadAndPropagate(node, r);
					//把head节点的next指针置为null,方便gc
					p.next = null; // help GC
					failed = false;
					return;
				}
			}
			if (shouldParkAfterFailedAcquire(p, node) &&
				parkAndCheckInterrupt())
				throw new InterruptedException();
		}
	} finally {
		if (failed)
			cancelAcquire(node);
	}
}
```

```java 
private void setHeadAndPropagate(Node node, int propagate) {
	Node h = head; // Record old head for check below
	setHead(node);

	if (propagate > 0 || h == null || h.waitStatus < 0 ||
		(h = head) == null || h.waitStatus < 0) {
		Node s = node.next;
		if (s == null || s.isShared())
			doReleaseShared(); //继续执行该方法
	}
}

```


```java 
private void doReleaseShared() {
	/*
	 * Ensure that a release propagates, even if there are other
	 * in-progress acquires/releases.  This proceeds in the usual
	 * way of trying to unparkSuccessor of head if it needs
	 * signal. But if it does not, status is set to PROPAGATE to
	 * ensure that upon release, propagation continues.
	 * Additionally, we must loop in case a new node is added
	 * while we are doing this. Also, unlike other uses of
	 * unparkSuccessor, we need to know if CAS to reset status
	 * fails, if so rechecking.
	 */
	for (;;) {
		Node h = head;
		if (h != null && h != tail) {
			int ws = h.waitStatus;
			if (ws == Node.SIGNAL) {
				if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
					continue;            // loop to recheck cases
				unparkSuccessor(h); //唤醒头结点的下一个节点
			}
			else if (ws == 0 &&
					 !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
				continue;                // loop on failed CAS
		}
		if (h == head)                   // loop if head changed
			break;
	}
}

```
如此反复，直到所有被await方法阻塞住的SHARED节点（aqs队列）里的线程都被唤醒，都抢到锁,继续执行后面的业务代码。


### 二、CyclicBarrier源码解析
1. 概述
Cyclibarrier内部持有Lock及Condition对象，定义一个资源的值，然后开启与资源值相同数量的线程，在线程运行期间，调用await()方法暂停执行，当所有线程都调用完await()方法，资源数量将达到0，最后一次调用await()的线程，在Cyclibarrier内部，会执行signalAll()唤醒所有等待线程。
当被唤醒时，N个线程是按AQS方式Condition的await()被唤醒一样的逻辑，同时进入获取lock的状态，谁先拿到锁谁继续执行。

2. 用法
```java
package com.example.countdown;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Random;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;


public class BarrierTest {

    static SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    public static void main(String[] args) {
//        CyclicBarrier cyclicBarrier = new CyclicBarrier(3);

        CyclicBarrier cyclicBarrier = new CyclicBarrier(2, new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println("等裁判吹口哨...");
                    //这里停顿两秒更便于观察线程执行的先后顺序
                    Thread.sleep(2000);
                    System.out.println("裁判吹口哨->>>>>");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });


        Runner runner1 = new Runner(cyclicBarrier, "张三");
        Runner runner2 = new Runner(cyclicBarrier, "李四");
        Runner runner3 = new Runner(cyclicBarrier, "王五");
        Runner runner4 = new Runner(cyclicBarrier, "赵六");

        ExecutorService service = Executors.newFixedThreadPool(2);
        service.execute(runner1);
        service.execute(runner2);
        service.execute(runner3);
        service.execute(runner4);

        service.shutdown();
    }

    static class Runner implements Runnable{

        private CyclicBarrier cyclicBarrier;

        private String name;

        public Runner(CyclicBarrier cyclicBarrier, String name) {
            this.cyclicBarrier = cyclicBarrier;
            this.name = name;
        }

        @Override
        public void run() {
            //模拟准备耗时
            try {
                Thread.sleep(new Random().nextInt(5000));
                System.out.println(name + " 准备ok ");
                cyclicBarrier.await();

                long startTime = System.currentTimeMillis();

                System.out.println(name + " 开跑 " + simpleDateFormat.format(new Date()));
                doRunning();
                System.out.println(name + " 成绩为： " + (System.currentTimeMillis() - startTime));
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
        }

        private static void doRunning(){
            try {
                Thread.sleep(new Random().nextInt(10000));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}


```


3. 源码分析

- 成员属性
```java
// 并没有自定义同步器，
// 而是定义了一个Generation类，里面包含一个broker属性
private static class Generation {
    boolean broken = false;
}
 
// 一堆属性
private final ReentrantLock lock = new ReentrantLock(); // 可重入锁
private final Condition trip = lock.newCondition(); // Condition 后面的await()和singalAll()的调用
private final int parties;     // 参与人数量
private final Runnable barrierCommand; // 触发时要运行的命令
private Generation generation = new Generation();
 
private int count;// 记录还有多少在等待
 
private static class Generation {
    boolean broken = false; // 只有一个标记值
}
```

- await()方法
```java

public int await() throws InterruptedException, BrokenBarrierException {
	try {
		return dowait(false, 0L);
	} catch (TimeoutException toe) {
		throw new Error(toe); // cannot happen
	}
}

//timed 代表超时时间 nanos=0L
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
TimeoutException {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //Generation是CyclicBarrier的一个内部类，其中broken属性为false
        final Generation g = generation;
 
        if (g.broken)
            throw new BrokenBarrierException();
 	//如果出现中断异常，打断中断，
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }
        //有线程执行该方法时index减一，其中count参数就是创建CyckicBarrier是的构造参数（2）
        int index = --count;
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                // 检查是否有额外动作要执行
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run();
                ranAction = true;
                // 进入下一个纪元，方法内部有调用 signalAll()唤醒所有阻塞的线程
                nextGeneration();
                return 0;
            } finally {
                if (!ranAction)
                    breakBarrier();
            }
        }
 
        // loop until tripped, broken, interrupted, or timed out   
		//如果index不为零进入自旋操作，则开始阻塞
		for (;;) {
                try {
		    //调用await方法阻塞当前线程
                    if (!timed)
                        trip.await(); // 一直阻塞
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos); // 带有超时的阻塞
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();// 关闭屏障 
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }
```
    
```java
// 打断状态
private void breakBarrier() {
    generation.broken = true; // 设置broken标识
    count = parties; // 重置计数器
    trip.signalAll(); // 唤醒所有阻塞线程
}
```
    
```java  
//condition源码解析中调用的是同一个await方法
//它会释放当前锁持有的锁，然后进入阻塞状态。释放锁的目的是让第二个线程可以进行—count的操作。
public final void await() throws InterruptedException {
    if (Thread.interrupted())
	throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
	LockSupport.park(this);
	if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
	    break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
	interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
	unlinkCancelledWaiters();
    if (interruptMode != 0)
	reportInterruptAfterWait(interruptMode);
}
```

直到最后一个线程执行时 index = 0 进入 if循环  调用nextGeneration();方法唤醒所有线程
```java
//该方法内部会唤醒所有阻塞的线程，是的它们可以继续执行。并且会重置下count和generation，真正的实现Cyclic,等待下一次的调用。
//唤醒完所有线程之后，当前线程就可以return了，所有线程也就可以继续执行了，至此，我们简单的分析它的功能就结束了。
private void nextGeneration() {
// signal completion of last generation
//唤醒所有等待队列中的线程
trip.signalAll();
// set up next generation
//重置count并新创建一个generation实现循环复用的功能
count = parties;
generation = new Generation();
}
```

通读上面的代码后，总结归纳下dowait()方法的逻辑：

1、线程调用后，会检查barrier的状态、线程状态，异常状态会中断。  

2、在初始化CyclicBarrier时，设置的资源值count，会进行--count  

3、当20个线程中前19个线程，执行dowait()后，由于count!=0，因此会进行for(;;)，在内部会执行Condition的trip.await()方法，进行阻塞。  

4、阻塞结束的条件有：a超时，b被唤醒，c线程中断。  

5、当第20个线程执行dowait()后，由于count==0，会先检查并执行command的内容  

6、最后执行nextGeneration()，在内部调用trip.signalAll()唤醒所有trip.await()的线程  


### 四、Semaphore

1. 概述
Semaphore 信号量，用来控制同一时间，资源可被访问的线程数量，一般可用于流量的控制。

2. 使用场景
比如，现在有 20 辆车要通过这个地段， 警察叔叔规定同一时间，最多只能通过 5 辆车，
其他车辆只能等待。只有拿到许可的车辆可通过，等车辆通过之后，再归还许可，然后把它发给等待的车辆，获得许可的车辆再通行，依次类推。

```java
package com.example.countdown;

import java.util.Random;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Semaphore;

public class SemaphoreTest {
    private static int count = 20;

    public static void main(String[] args) {

        ExecutorService executorService = Executors.newFixedThreadPool(count);

        //指定最多只能有五个线程同时执行
        Semaphore semaphore = new Semaphore(5);

        Random random = new Random();
        for (int i = 0; i < count; i++) {
            final int no = i;
            executorService.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        //获得许可
                        semaphore.acquire();
                        System.out.println(no +":号车可通行");
                        //模拟车辆通行耗时
                        Thread.sleep(random.nextInt(2000));
                        //释放许可
                        semaphore.release();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
        }

        executorService.shutdown();

    }
}

```

打印结果我就不写了，需要读者自行观察，就会发现，第一批是五个车同时通行。然后，后边的车才可以依次通行，但是同时通行的车辆不能超过 5 辆。  

细心的读者，就会发现，这许可一共就发 5 个，那等第一批车辆用完释放之后， 第二批的时候应该发给谁呢？  

这确实是一个问题。所有等待的车辆都想先拿到许可，先通行，怎么办。这就需要，用到锁了。就所有人都去抢，谁先抢到，谁就先走呗。  

我们去看一下 Semaphore的构造函数，就会发现，可以传入一个 boolean 值的参数，控制抢锁是否是公平的。  

```java 
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}
public Semaphore(int permits, boolean fair) {
    sync = fair   new FairSync(permits) : new NonfairSync(permits);
}
```

默认是非公平，可以传入 true 来使用公平锁。

3.源码分析 
- 构造方法:  
```java
public Semaphore(int permits) {
//默认是非公平锁
sync = new NonfairSync(permits);
}

NonfairSync(int permits) {
    //一个参数的构造方法 
    super(permits);
}

Sync(int permits) {
//其实就是设置state的值
    setState(permits);
}

protected final void setState(int newState) {
state = newState;
}
```

- acquire()方法分析
```java 
public void acquire() throws InterruptedException {
sync.acquireSharedInterruptibly(1);
}
    
public final void acquireSharedInterruptibly(int arg)
    throws InterruptedException {
	if (Thread.interrupted())
		throw new InterruptedException();
		//默认是非公平锁的实现 
	if (tryAcquireShared(arg) < 0)
		//加入等待队列，阻塞当前线程
		doAcquireSharedInterruptibly(arg);
}

//非公平锁 
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}

/**
* 尝试以非公平方式获取acquires个信号，非公平方式即不管等待队列中等待已久的其他线程，直接尝试使用CAS获取，获取失败会被加入等待队列
* 方法写到一个死循环中，只有获取成功或者"当前信号数 < 本次需要获取信号数"跳出循环
* 获取成功返回正值，失败返回负值
* 方法除了cas外没有用到任何同步的操作，原因在与for(;;)的骚操作，获取当前state值与cas都包在这个死循环
* 代码块里，如果本线程由于其他线程竞争关系导致cas失败，会再次重新执行在当前的state值减去acquire后执行cas
* 通过cas操作，来设置state的值（因为是可重入锁，直接修改state的值）
*/
final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
	int available = getState();
	int remaining = available - acquires;
	if (remaining < 0 ||
	    compareAndSetState(available, remaining))
	    return remaining;
    }
}

//公平锁
//队列中有值直接返回-1加入到等到队列中，否则才去抢占锁
protected int tryAcquireShared(int acquires) {
    for (;;) {
	if (hasQueuedPredecessors())
	    return -1;
	int available = getState();
	int remaining = available - acquires;
	if (remaining < 0 ||
	    compareAndSetState(available, remaining))
	    return remaining;
    }
}

//上面分析过了，真不说了 ，就是加入等待队列，阻塞当前线程（不会释放锁）
private void doAcquireSharedInterruptibly(int arg)
throws InterruptedException {
final Node node = addWaiter(Node.SHARED);
boolean failed = true;
try {
    for (;;) {
	final Node p = node.predecessor();
	if (p == head) {
	    int r = tryAcquireShared(arg);
	    if (r >= 0) {
		setHeadAndPropagate(node, r);
		p.next = null; // help GC
		failed = false;
		return;
	    }
	}
	if (shouldParkAfterFailedAcquire(p, node) &&
	    parkAndCheckInterrupt())
	    throw new InterruptedException();
    }
} finally {
    if (failed)
	cancelAcquire(node);
}
}
```

- release()方法 
```java 
public void release() {
sync.releaseShared(1);
}

public final boolean releaseShared(int arg) {
if (tryReleaseShared(arg)) {
    doReleaseShared();
    return true;
}
return false;
}

protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        //线程1进来此时获取的current值等于5 
	int current = getState();
	int next = current + releases;
	if (next < current) // overflow
	    throw new Error("Maximum permit count exceeded");
	//cas操作设置state的值为next，返回true执行 doReleaseShared();方法
	if (compareAndSetState(current, next))
	    return true;
    }
}
```

```java 
private void doReleaseShared() {
/*
 * Ensure that a release propagates, even if there are other
 * in-progress acquires/releases.  This proceeds in the usual
 * way of trying to unparkSuccessor of head if it needs
 * signal. But if it does not, status is set to PROPAGATE to
 * ensure that upon release, propagation continues.
 * Additionally, we must loop in case a new node is added
 * while we are doing this. Also, unlike other uses of
 * unparkSuccessor, we need to know if CAS to reset status
 * fails, if so rechecking.
 */
for (;;) {
    Node h = head;
    if (h != null && h != tail) {
	int ws = h.waitStatus;
	if (ws == Node.SIGNAL) {
	    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
		continue;            // loop to recheck cases
	    unparkSuccessor(h);
	}
	else if (ws == 0 &&
		 !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
	    continue;                // loop on failed CAS
    }
    if (h == head)                   // loop if head changed
	break;
}
}
```
