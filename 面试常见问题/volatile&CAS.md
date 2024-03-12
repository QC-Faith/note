volatile&CAS



aqs中的状态就是使用volatile进行修饰的

Atomic中value也是，原子操作则使用cas

ThreadPoolExecutor中的runState也是



volatile保证可见性因为volatile不使用线程内存区域，而是直接从主内存进行数据的获取

因为属于乐观锁，所以读写是不锁的，且可以保证重排序





什么是重排序？

写操作最大，由于其他所有后续操作



重排序是如何保证的

使用内存屏障，写实在前后分别插入内存屏障，读是在后面插入内存屏障



什么是内存屏障？

简单理解内存屏障就是将被volatile修饰的前后添加标记位，使其他线程在进入时被阻塞







CAS

轻量级乐观锁

读取时不加锁，写回数据时先查询当前值，如果被修改则继续执行读取流程，保证了原子性



问题：cas长时间不成功导致cpu自旋，压力较大

ABA问题：添加标志位

1. 初始状态：线程 A 读取了共享变量的值为 A，并准备执行 CAS 操作。
2. 线程 A 执行 CAS 操作前，共享变量的值被其他线程 B 修改为了 B，然后又被修改回 A，此时的共享变量的值与线程 A 最初读取的值相同，但实际上已经发生了变化。
3. 线程 A 执行 CAS 操作，检查到共享变量的值仍然是 A，于是认为变量未被其他线程修改过，执行 CAS 操作成功。





*Atomic中使用了计数器来解决aba的问题

  int expect = counter.get();

​        int update = expect + 1;

​        while (!counter.compareAndSet(expect, update)) {

​            // 如果CAS操作失败，则重新获取最新的值进行尝试 

​           expect = counter.get();

​            update = expect + 1;    

​    }





*ConcurrentHashMap 在put中野使用了cas，CAS 操作尝试添加新的节点



ConcurrentHashMap 中的每个节点都包含一个版本号，该版本号就是为了解决aba问题

