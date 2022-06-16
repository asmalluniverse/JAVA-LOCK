### һ��CountDownLatch
����һ��ͬ��������������һ�������߳�һֱ�ȴ���ֱ�������߳�ִ�еĲ���ȫ����ɣ�����ִ�С�

##### 1.���췽��

���췽�����ᴫ��һ�� count ֵ�����ڼ���,�������Ϊ��Ҫ�ȴ����߳�����
```java
public CountDownLatch(int count) {
	if (count < 0) throw new IllegalArgumentException("count < 0");
	this.sync = new Sync(count); //����ͬ�����У������ó�ʼ������ֵ
}
```
##### 2.CountDownLatch�����
1. ���÷���
```java
public void await() throws InterruptedException {
	sync.acquireSharedInterruptibly(1);
}

public void countDown() {
	sync.releaseShared(1);
}
```
��һ���̵߳���await����ʱ���ͻ�������ǰ�̡߳�ÿ�����̵߳���һ�� countDown ����ʱ�������ͻ�� 1���� count ��ֵ���� 0 ��ʱ�򣬱��������̲߳Ż�������С�

2. ��������
CountDownLatch����һ���ṩ���ĸ����������������������£�
1. �����ڵ���await����ʱ���ݳ�ʱʱ��
```java
public boolean await(long timeout, TimeUnit unit)
	throws InterruptedException {
	return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
```

2. ��ȡ�ȴ��̵߳�����
```java
public long getCount() {
	return sync.getCount();
}
```


##### 3.���ʹ��
1. ���������³�����

��������һ����������˾��Ŀ�����ϳ�����һ������ bug�����ͻ�Ͷ�ߣ��쵼�����Ĺ�����������Ѹ�ٵĽ����� bug ��
��ô��һ���˽���϶��ٶ����������ǽ������������ģ�һ��ֹ���������ڣ��������������������Լ�����Ҫ��������֮���쵼�ſ��Դ𸴿ͻ����ͻ�Ҳ�������ˣ�û�취�����ͻ����ϵ����
���ǣ����ǿ������һ�� Worker ����ģ�ⵥ�����޸� bug �Ĺ��̣����߳̾����쵼��һֱ�ȴ����� Worker ����ִ�н��������̲߳ſ��Լ��������ߡ�


```java
package com.example.countdown;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.CountDownLatch;


public class CountDownTest {

    static SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(2);
        new Thread(new Worker("��ɽ",2000, countDownLatch)).start();
        new Thread(new Worker("����",3000, countDownLatch)).start();
        long startTime = System.currentTimeMillis();

        countDownLatch.await();

        System.out.println("bugȫ��������쵼���Ը��ͻ������ˣ������ܺ�ʱ��"+ (System.currentTimeMillis() - startTime));

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
            System.out.println(name + " ��ʼ�޸�Bug ����ǰʱ�䣺" + simpleDateFormat.format(new Date()));
            doWork();
            System.out.println(name + " ����޸�Bug ����ǰʱ�䣺" + simpleDateFormat.format(new Date()));
            latch.countDown();
        }

        private void doWork() {
            try {
                //ģ�⹤����ʱ
                Thread.sleep(workTime);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

```

##### 4.Դ�����
�������Ǵ���һ��CountDownLatch���󣬲�ͨ�����췽������state��ֵ�������������������Ȼ�����await������ʱ�������̶߳����װ��һ��Node�ڵ㣬�ڵ�״̬��shared״̬�� ÿ���̵߳��� countDown() ��ʱ�� ,��� state��������һ, �����ε���countDown(),ֱ����state����������0, ��ô �ͻỽ�Ѷ����У���await()����ס���߳�(Node�ڵ�)���������������ɹ��Ļ�, �ͻ���� ������һ�� ��await��������ס��SHARED�߳̽ڵ�, ��һ���̱߳����Ѻ� ��������������Ҫ�����Լ�����һ����ֱ�����б�await������await�ڵ��̶߳�������,ֱ������Ϊ�ա�  

���췽����������state��ֵ����Ͳ�������������ֱ�ӷ���await������countDown������  
��ʱ����CountDownLatch countDownLatch = new CountDownLatch(2);���������߳�A,B,���̵߳���await������

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
	//����state������ֵ����true false 
	if (tryAcquireShared(arg) < 0)
		doAcquireSharedInterruptibly(arg);
}

protected int tryAcquireShared(int acquires) {
    //getState��ȡ���õĲ����������췽�������õ�count,��ʱΪ2 ����-1��if�������㣬����doAcquireSharedInterruptibly(arg);���� 
	return (getState() == 0)   1 : -1;
}

private void doAcquireSharedInterruptibly(int arg) ��aqs�е��߼���
	throws InterruptedException {
	//�ѵ�ǰ�ڵ���뵽aqs�����У�aqs�е��߼���
	final Node node = addWaiter(Node.SHARED);
	boolean failed = true;
	try {
		for (;;) {
		    //��ȡ��ǰ�ڵ��ǰ�ýڵ�
			final Node p = node.predecessor();
			if (p == head) {
			    //����ͷ������һ���ڵ�
				int r = tryAcquireShared(arg);
				if (r >= 0) {
				    //setHeadAndPropagate �������������ȴ��� LockSupport //�������̻߳��ѡ���ͨ������parkAndCheckInterrupt()���������󣬵ȴ������̵߳Ļ���
					//��Ϊ������һ�����������������ʱr>=0 ˵��count��ֵ�Ѿ�����0
					setHeadAndPropagate(node, r);
					p.next = null; // help GC
					failed = false;
					return;
				}
			}
			//����tryAcquireShared��state����Ϊ0��r=-1�����ͻᵽ�����һ�ε���shouldParkAfterFailedAcquire������ʱ��waitStatus��0����else����
			//��ônode��waitStatus�ͻᱻ��ΪSIGNAL������true,Ȼ�����parkAndCheckInterrupt�������ͻ����LockSupport��park�����ѵ�ǰ�߳�����
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


2. countDown()��������
```java 
//�����ʱ�߳�A��ִ���ˣ�Ȼ���߳�B�ٴ�ִ��
public void countDown() {
	sync.releaseShared(1);
}

```

```java
public final boolean releaseShared(int arg) {
	// ��tryReleaseShared����true����ʾcount��������ͷź󣬵���0�ˣ�����ִ��doReleaseShared
	//�߳�A�ȵ��ø÷�������ʱstate��ֵΪ2������tryReleaseShared�����-1�������������㷵��false
	//Ȼ���߳�B�ٴ�ִ��,��ʱstate��ֵΪ1������tryReleaseShared�����-1��������һ�����0�����л����doReleaseShared������
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
	//�������������һ��ʼ��Ϊ�㣬Ҳ����falseû��Ҫִ��doReleaseShared ������ 
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
//������������0����ô�ͻ�ִ����������������������߳�,������ִ��ҵ�����
private void doReleaseShared() {
	for (;;) {
		Node h = head;
		if (h != null && h != tail) {
			int ws = h.waitStatus;
			//ǰ�����������е�ͷ����״̬��SIGNAL�����ߵ�If������
			if (ws == Node.SIGNAL) {
				if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
					continue;            // loop to recheck cases
				unparkSuccessor(h); //����ͷ������һ���ڵ㣬��ʱ�����ִ��doAcquireSharedInterruptibly����
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
		LockSupport.unpark(s.thread); //���ѽڵ� ��ʱ����ִ�� doAcquireSharedInterruptibly�б��������߳�
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
			    //��ʱcount ��Ϊ0 r����1����setHeadAndPropagate����
				int r = tryAcquireShared(arg);
				if (r >= 0) {
					//���ѵ�ǰ�ڵ�󣬼�������doReleaseShared�������������е������ڵ�
					setHeadAndPropagate(node, r);
					//��head�ڵ��nextָ����Ϊnull,����gc
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
			doReleaseShared(); //����ִ�и÷���
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
				unparkSuccessor(h); //����ͷ������һ���ڵ�
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
��˷�����ֱ�����б�await��������ס��SHARED�ڵ㣨aqs���У�����̶߳������ѣ���������,����ִ�к����ҵ����롣


### ����CyclicBarrierԴ�����
1. ����
Cyclibarrier�ڲ�����Lock��Condition���󣬶���һ����Դ��ֵ��Ȼ��������Դֵ��ͬ�������̣߳����߳������ڼ䣬����await()������ִͣ�У��������̶߳�������await()��������Դ�������ﵽ0�����һ�ε���await()���̣߳���Cyclibarrier�ڲ�����ִ��signalAll()�������еȴ��̡߳�
��������ʱ��N���߳��ǰ�AQS��ʽCondition��await()������һ�����߼���ͬʱ�����ȡlock��״̬��˭���õ���˭����ִ�С�

2. �÷�
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
                    System.out.println("�Ȳ��д�����...");
                    //����ͣ����������ڹ۲��߳�ִ�е��Ⱥ�˳��
                    Thread.sleep(2000);
                    System.out.println("���д�����->>>>>");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });


        Runner runner1 = new Runner(cyclicBarrier, "����");
        Runner runner2 = new Runner(cyclicBarrier, "����");
        Runner runner3 = new Runner(cyclicBarrier, "����");
        Runner runner4 = new Runner(cyclicBarrier, "����");

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
            //ģ��׼����ʱ
            try {
                Thread.sleep(new Random().nextInt(5000));
                System.out.println(name + " ׼��ok ");
                cyclicBarrier.await();

                long startTime = System.currentTimeMillis();

                System.out.println(name + " ���� " + simpleDateFormat.format(new Date()));
                doRunning();
                System.out.println(name + " �ɼ�Ϊ�� " + (System.currentTimeMillis() - startTime));
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


3. Դ�����

- ��Ա����
```java
// ��û���Զ���ͬ������
// ���Ƕ�����һ��Generation�࣬�������һ��broker����
private static class Generation {
    boolean broken = false;
}
 
// һ������
private final ReentrantLock lock = new ReentrantLock(); // ��������
private final Condition trip = lock.newCondition(); // Condition �����await()��singalAll()�ĵ���
private final int parties;     // ����������
private final Runnable barrierCommand; // ����ʱҪ���е�����
private Generation generation = new Generation();
 
private int count;// ��¼���ж����ڵȴ�
 
private static class Generation {
    boolean broken = false; // ֻ��һ�����ֵ
}
```

- await()����
```java

public int await() throws InterruptedException, BrokenBarrierException {
	try {
		return dowait(false, 0L);
	} catch (TimeoutException toe) {
		throw new Error(toe); // cannot happen
	}
}

//timed ����ʱʱ�� nanos=0L
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
TimeoutException {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //Generation��CyclicBarrier��һ���ڲ��࣬����broken����Ϊfalse
        final Generation g = generation;
 
        if (g.broken)
            throw new BrokenBarrierException();
 	//��������ж��쳣������жϣ�
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }
        //���߳�ִ�и÷���ʱindex��һ������count�������Ǵ���CyckicBarrier�ǵĹ��������2��
        int index = --count;
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                // ����Ƿ��ж��⶯��Ҫִ��
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run();
                ranAction = true;
                // ������һ����Ԫ�������ڲ��е��� signalAll()���������������߳�
                nextGeneration();
                return 0;
            } finally {
                if (!ranAction)
                    breakBarrier();
            }
        }
 
        // loop until tripped, broken, interrupted, or timed out   
		//���index��Ϊ�����������������ʼ����
		for (;;) {
                try {
		    //����await����������ǰ�߳�
                    if (!timed)
                        trip.await(); // һֱ����
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos); // ���г�ʱ������
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();// �ر����� 
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
// ���״̬
private void breakBarrier() {
    generation.broken = true; // ����broken��ʶ
    count = parties; // ���ü�����
    trip.signalAll(); // �������������߳�
}
```
    
```java  
//conditionԴ������е��õ���ͬһ��await����
//�����ͷŵ�ǰ�����е�����Ȼ���������״̬���ͷ�����Ŀ�����õڶ����߳̿��Խ��С�count�Ĳ�����
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

ֱ�����һ���߳�ִ��ʱ index = 0 ���� ifѭ��  ����nextGeneration();�������������߳�
```java
//�÷����ڲ��ỽ�������������̣߳��ǵ����ǿ��Լ���ִ�С����һ�������count��generation��������ʵ��Cyclic,�ȴ���һ�εĵ��á�
//�����������߳�֮�󣬵�ǰ�߳̾Ϳ���return�ˣ������߳�Ҳ�Ϳ��Լ���ִ���ˣ����ˣ����Ǽ򵥵ķ������Ĺ��ܾͽ����ˡ�
private void nextGeneration() {
// signal completion of last generation
//�������еȴ������е��߳�
trip.signalAll();
// set up next generation
//����count���´���һ��generationʵ��ѭ�����õĹ���
count = parties;
generation = new Generation();
}
```

ͨ������Ĵ�����ܽ������dowait()�������߼���

1���̵߳��ú󣬻���barrier��״̬���߳�״̬���쳣״̬���жϡ�  

2���ڳ�ʼ��CyclicBarrierʱ�����õ���Դֵcount�������--count  

3����20���߳���ǰ19���̣߳�ִ��dowait()������count!=0����˻����for(;;)�����ڲ���ִ��Condition��trip.await()����������������  

4�����������������У�a��ʱ��b�����ѣ�c�߳��жϡ�  

5������20���߳�ִ��dowait()������count==0�����ȼ�鲢ִ��command������  

6�����ִ��nextGeneration()�����ڲ�����trip.signalAll()��������trip.await()���߳�  


### �ġ�Semaphore

1. ����
Semaphore �ź�������������ͬһʱ�䣬��Դ�ɱ����ʵ��߳�������һ������������Ŀ��ơ�

2. ʹ�ó���
���磬������ 20 ����Ҫͨ������ضΣ� ��������涨ͬһʱ�䣬���ֻ��ͨ�� 5 ������
��������ֻ�ܵȴ���ֻ���õ���ɵĳ�����ͨ�����ȳ���ͨ��֮���ٹ黹��ɣ�Ȼ����������ȴ��ĳ����������ɵĳ�����ͨ�У��������ơ�

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

        //ָ�����ֻ��������߳�ͬʱִ��
        Semaphore semaphore = new Semaphore(5);

        Random random = new Random();
        for (int i = 0; i < count; i++) {
            final int no = i;
            executorService.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        //������
                        semaphore.acquire();
                        System.out.println(no +":�ų���ͨ��");
                        //ģ�⳵��ͨ�к�ʱ
                        Thread.sleep(random.nextInt(2000));
                        //�ͷ����
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

��ӡ����ҾͲ�д�ˣ���Ҫ�������й۲죬�ͻᷢ�֣���һ���������ͬʱͨ�С�Ȼ�󣬺�ߵĳ��ſ�������ͨ�У�����ͬʱͨ�еĳ������ܳ��� 5 ����  

ϸ�ĵĶ��ߣ��ͻᷢ�֣������һ���ͷ� 5 �����ǵȵ�һ�����������ͷ�֮�� �ڶ�����ʱ��Ӧ�÷���˭�أ�  

��ȷʵ��һ�����⡣���еȴ��ĳ����������õ���ɣ���ͨ�У���ô�졣�����Ҫ���õ����ˡ��������˶�ȥ����˭��������˭�������¡�  

����ȥ��һ�� Semaphore�Ĺ��캯�����ͻᷢ�֣����Դ���һ�� boolean ֵ�Ĳ��������������Ƿ��ǹ�ƽ�ġ�  

```java 
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}
public Semaphore(int permits, boolean fair) {
    sync = fair   new FairSync(permits) : new NonfairSync(permits);
}
```

Ĭ���Ƿǹ�ƽ�����Դ��� true ��ʹ�ù�ƽ����

3.Դ����� 
- ���췽��:  
```java
public Semaphore(int permits) {
//Ĭ���Ƿǹ�ƽ��
sync = new NonfairSync(permits);
}

NonfairSync(int permits) {
    //һ�������Ĺ��췽�� 
    super(permits);
}

Sync(int permits) {
//��ʵ��������state��ֵ
    setState(permits);
}

protected final void setState(int newState) {
state = newState;
}
```

- acquire()��������
```java 
public void acquire() throws InterruptedException {
sync.acquireSharedInterruptibly(1);
}
    
public final void acquireSharedInterruptibly(int arg)
    throws InterruptedException {
	if (Thread.interrupted())
		throw new InterruptedException();
		//Ĭ���Ƿǹ�ƽ����ʵ�� 
	if (tryAcquireShared(arg) < 0)
		//����ȴ����У�������ǰ�߳�
		doAcquireSharedInterruptibly(arg);
}

//�ǹ�ƽ�� 
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}

/**
* �����Էǹ�ƽ��ʽ��ȡacquires���źţ��ǹ�ƽ��ʽ�����ܵȴ������еȴ��Ѿõ������̣߳�ֱ�ӳ���ʹ��CAS��ȡ����ȡʧ�ܻᱻ����ȴ�����
* ����д��һ����ѭ���У�ֻ�л�ȡ�ɹ�����"��ǰ�ź��� < ������Ҫ��ȡ�ź���"����ѭ��
* ��ȡ�ɹ�������ֵ��ʧ�ܷ��ظ�ֵ
* ��������cas��û���õ��κ�ͬ���Ĳ�����ԭ������for(;;)��ɧ��������ȡ��ǰstateֵ��cas�����������ѭ��
* ������������߳����������߳̾�����ϵ����casʧ�ܣ����ٴ�����ִ���ڵ�ǰ��stateֵ��ȥacquire��ִ��cas
* ͨ��cas������������state��ֵ����Ϊ�ǿ���������ֱ���޸�state��ֵ��
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

//��ƽ��
//��������ֱֵ�ӷ���-1���뵽�ȵ������У������ȥ��ռ��
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

//����������ˣ��治˵�� �����Ǽ���ȴ����У�������ǰ�̣߳������ͷ�����
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

- release()���� 
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
        //�߳�1������ʱ��ȡ��currentֵ����5 
	int current = getState();
	int next = current + releases;
	if (next < current) // overflow
	    throw new Error("Maximum permit count exceeded");
	//cas��������state��ֵΪnext������trueִ�� doReleaseShared();����
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
