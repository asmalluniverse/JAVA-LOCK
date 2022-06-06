### 概述
synchronized，ReentrantLock都是java中提供的锁，可以保证多线程使用时的安全性，相比于synchronized，ReentrantLock需要显式的获取锁和释放锁，在jdk1.6之前ReentrantLock的效率要优与synchronized，在之后synchronized进行了优化，底层使用了cas操作，并提供了很多场景下锁的状态，包括自适应锁、自旋锁、锁消除、锁粗化、轻量级锁和偏向锁，ReentrantLock的效率和synchronized性能区别基本可以持平了。

### 区别
1. 对应使用上synchronized是java内置的锁，是一个关键字，使用的方便性要优与ReentrantLock.
2. 等待可中断，当持有锁的线程长时间不释放锁的时候，等待中的线程可以选择放弃等待，转而处理其他的任务。
3. 公平锁：synchronized和ReentrantLock默认都是非公平锁，但是ReentrantLock可以通过构造函数传参改变。只不过使用公平锁的话会导致性能下降。
4. 绑定多个条件：ReentrantLock可以同时绑定多个Condition条件对象。
