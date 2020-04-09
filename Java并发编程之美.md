# Java并发编程之美

## Java并发基础

### 锁

#### 公平锁与非公平锁

公平锁指线程获取锁的顺序是按照线程请求锁的时间早晚来决定的.非公平锁是运行时闯入.

ReentrantLock提供了两种锁的实现:

* 公平锁:

```java
ReentrantLock pairLock = new ReentrantLock(true);
```

* 非公平锁

```java
ReentrantLock pairLock = new ReentrantLock(false);
```

#### 独占锁和共享锁

锁只能被单个线程持有还是能被多个线程共同持有.

ReentrantLock是独占锁

ReadWriteLock是共享锁

#### 可重入锁

只要获取锁,就能可以进入被该锁锁住的代码.如果获取某个对象的锁,在调用对象其他加锁方法时可以不会被阻塞.

> Synchronized内部锁时可重入锁.可重入锁的内部原理是在锁内部维护一个线程标识,用来标记哪个线程在占用锁,然后关联一个计数器,初始值为0.当持有锁的线程再次获取锁时计数器加1,释放锁时计数器减1,当计数器为0时,锁里面的标示被重置为null,这时被阻塞的线程会被唤醒来竞争该锁

#### 自旋锁

**Java中的线程是与操作系统中的线程一一对应的**,当一个线程获取锁失败后,会被切换到内核状态而挂起.

**自旋锁则是,当前线程在获取锁时,如果发现锁已经被其他线程占有,不会马上阻塞自己,在不放弃CPU的使用权的情况下多次尝试获取(默认次数是10, -XX:PreBlockSpinsh参数设置)**

## Java并发编程高级篇

### ThreadLocalRandom类原理剖析

`ThreadLocalRandom`是Java7新增的随机数生成器,弥补Random类在多线程下的缺陷.

#### Random类的局限

```java
public int nextInt(int bound) {
    if (bound < 0) {
        ...
    }
    
    //4 根据老种子生成新的种子
    int r = next(31)
    // 5
    ...
}
```

在多线程情况下多个线程可能获取相同的种子来计算新的种子,这会导致多个线程产生的新种子是一样的,由于算法固定,所以会产生相同的随机数.若使用CAS去更新种子会造成大量线程自旋会降低并发性能.

#### ThreadLocalRandom

```java
ThreadLocalRandom random =  ThreadLocalRandom.current();
for (int i = 0; i < 10; ++i) {
    System.out.println(random.nextInt(5));
}
```

每个线程都维护自己的种子变量.

### Java并发包中的原子操作类剖析

Java并发包提供了一系列的原子性操作,都是通过非阻塞算法CAS实现的.

#### 原子变量操作类

Java中的原子类原理类似,以AtomicLong为例

```java
public final long getAndIncrement() {
    return unsafe.getAndAddLong(this. valueOffset, 1L)
}
```

Java7中的实现逻辑:

```java
public final long getAndIncrement() {
    while (true) {
        long current = get();
        long next = current + 1;
        if (compareAndSet(current, next)) {
            return current;
        }
    }
}
```

线程获取变量的当前值,然后在工作内存中对其进行增一操作,而后使用CAS修改变量的值,如果设置失败,则循环尝试,直到成功.

Java8中的实现:

```java
public final long getAndIncrement() {
    return unsafe.getAndAddLong(this. valueOffset, 1L)
}

public final long getAndAddLong(Object o, long offset, long delta) {
    long v;
    do {
        v = getLongVolatile(o, offset);
    } while (!weakCompareAndSetLong(o, offset, v, v + delta));
    return v;
}
```

在Java8中的原子操作被Unsafe 内置了.

#### compareAndSet方法

```java
public final boolean compareAndSet(long expectedValue, long newValue) {
    return U.compareAndSetLong(this, VALUE, expectedValue, newValue);
}
```

从代码可知最后调用Unsafe类的compareAndSetLong方法,如果原子变量中的value的值等于expect则会使用newValue更新值并返回true否则返回false.

#### LongAdder

在使用AtomicLong时,在高并发下大量线程会同时竞争更新同一个原子变量,但是由于同时只有一个线程的CAS操作成功,这会造成大量线程竞争失败后,会通过无限循环不断进行自旋尝试CAS操作,这会浪费CPU资源.

Java8新增了LongAdder用来解决这种缺点.

> 把一个变量分解为多个变量,让同样多的线程竞争多个资源

LongAdder在内部维护多个Cell变量,每个Cell里面有一个初始值为0的long型变量,在同等并发量的情况下,争夺单个变量的更新操作的线程量会减少,变相减少争夺共享资源的并发量.最后获取值时把所有Cell变量的值相加后再加上base值返回.



#### LongAccumulator类

LongAddder类是LongAccumulator的特例.

### Java中的并发List类

Java中的并发List只有`CopyOnWriteArrayList`是一个安全的ArrayList,底层使用写时复制策略.

在调用add方法时,线程获取独占锁,其他线程不能对List中的内部数组修改.然后复制一个新的数组大小时原来的数组大小加1,然后用新数组替换旧数组并在返回前释放锁.

写时复制会产生弱一致性问题.由于对于获取数组中的元素并未加锁,所以某个线程对数组进行操作且没完成时,另一个线程获取当前操作元素,虽然另一个线程会对当前元素修改.

### Java中的并发包中锁原理剖析

#### LockSupport工具类

LockSupport类的主要作用时挂起和唤醒线程.该类是创建锁和其他同步类的基础.

##### park方法

如果调用park方法的线程已经拿到与LockSupport关联的许可证,则调用park方法时马上返回,否则调用线程会被禁止参与调度也就是被阻塞挂起.

##### unpark方法

当一个线程调用unpark时,如果参数线程没有持有thread与LockSupport类关联的许可证,则让thread线程持有.如果thread之前因调用park()而被挂起,则调用unpark后,该线程会被唤醒.

#### 抽象同步队列AQS概述

##### AQS锁的底层支持

AQS是一个FIFO双向队列,内部通过head,tail记录首尾节点.队列的元素为Node.Node中的thread变量用来存放进入队列AQS数组.内部使用SHARED用来标记该线程是获取共享资源时被阻塞后挂起.Exclusive用来标记线程是获取独占资源时被挂起后放入AQS队列.waitStatus记录当前线程的等待状态.

AQS中维护了一个单一的状态信息state,可以通过getState,setState,compareAndSetState获取和修改其值.

* 对于ReentrantLock实现来说,state用来表示当前线程获取锁的可重入次数.
* ReentrantReadWriteLock来说,state的高16位表示读状态,低16位表示写锁的线程的可重入次数
* 对semaphore来说,state用来表示当前可用信号的次数
* 对CountDownLatch来说,state用来表示当前计数器的值

AQS有个内部类ConditionObject,用来结合锁实现线程同步

##### AQS条件变量的支持

