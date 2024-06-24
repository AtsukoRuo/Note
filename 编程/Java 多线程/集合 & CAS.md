# 集合 & CAS

## CAS

 CAS 实现原子型背后的一个假设是共享变量的当前值与当前线程所提供的旧值相同，我们就认为这个变量没有被其他线程修改过（乐观锁的思想）。在执行 CAS 指令时，CPU会履行 Coherence 协以确保 CAS 操作的可见性。

CAS 虽然很高效，但是它也存在三大问题：

- **ABA问题**：对于共享变量V，当前线程看到它的值为 A 那一刻，其他线程已经将其值更新为B，接着在当前线程执行 CAS 的时候，该变量的值又被其他线程更新为 A。这就是ABA问题，该问题是否可以接受与算法有关。想要规避 ABA 问题也不难，为共享变量的更新引入一个修订号（也称时间戳）即可。AtomicStampedReference 类就是基于这种思想的。
- **特定的性能问题**，空旋消耗 CPU 资源，此外在SMP架构上，还可能导致总线风暴
- **只能保证一个共享变量的原子操作**。对一个共享变量执行操作时，CAS能够保证原子操作，但是对多个共享变量操作时，CAS是无法保证操作的原子性的。

使用 CAS 进行无锁编程的基本模型：

~~~java
do {
	获得字段的期望值（oldValue）;
	计算出需要替换的新值（newValue）;
} while (!CAS(内存地址，oldValue，newValue))
~~~

### Unsafe

我们可以使用 sun.misc.Unsafe 下的 CAS 操作。由于 Unsafe 类不可以实例化，我们只能通过反射技术来间接实例化：

~~~java
public static Unsafe getUnsafe(){
    try {
        Field theUnsafe =
            Unsafe.class.getDeclaredField("theUnsafe");
        theUnsafe.setAccessible(true);
        return (Unsafe) theUnsafe.get(null);
    } catch (Exception e) {
        throw new AssertionError(e);
    }
}
~~~

Unsafe提供的 CAS 方法主要如下

- `public final native boolean compareAndSwapObject(Object o, long offset, Object  expected, Object update);`

  - 字段所在的对象
  - 字段在对象中的偏移量
  - 期望值
  - 更新值

  如果字段值和期望值相同，那么就将值设置为更新值。

- `public final native boolean compareAndSwapInt(Object o, long offset, int expected,int update);`

- `public final native boolean compareAndSwapLong(Object o, long offset, long expected, long update);`



Unsafe 提供的获取字段（属性）偏移量的方法：

- `public native long staticFieldOffset(Field field);`
- `public native long objectFieldOffset(Field field);`

下面给出一个使用示例：

~~~java
while (!unSafeCompareAndSet(oldValue, oldValue + 1))
    ;
~~~

~~~java
public final boolean unSafeCompareAndSet(
    int oldValue, 
    int newValue) {
    long valueOffset = unsafe.objectFieldOffset(OptimisticLockingPlus.class.getDeclaredField("value"));
    return unsafe.compareAndSwapInt(this, valueOffset, oldValue, newValue);
}
~~~



###  原子变量类

 原子变量类（Atomics）是基于 CAS 实现的工具类。能够保证 read-modify-write 操作的原子性和可见性。在执行 CAS 指令时，CPU会履行 Coherence 协以确保 CAS 操作的可见性

Java为我们提供的CAS类：

- 基础数据型：AtomicInteger、AtomicLong、AtomicBoolean
- 数组型：AtomicIntegerArray、AtomicLongArray、AtomicReferenceArray
- 字段更新器：AtomiclntegerFieldUpdater、AtomicLongFieldUpdater、AtomicReferenceFieldUpdater
- 引用型：AtomicReference、AtomicStampedReference、AtomicMarkableReference

`AtomicLong` 类的常用方法

- `long get()`
- `long getAndIncrement()`
- `long getAndDecrement()`
- `long incrementAndGet()`
- `void set(long newValue)`
- `boolean compareAndSet(long expect, long update)`

`AtomicBoolean`类主要提供了以下方法：

- `get()`：获取当前值。
- `set(boolean newValue)`：设置新的值。
- `getAndSet(boolean newValue)`：获取当前值，并设置新的值。
- `compareAndSet(boolean expect, boolean update)`：如果当前值`==`预期值，则以原子方式将该值设置为输入值（update）。

AtomicIntegerArray 类提供了以下方法：

- `int get(int i)` 
- `int getAndSet(int i, int newValue)`
- `int getAndIncrement(int i)`
- `int getAndDecrement(int i)` 
- `int getAndAdd(int i, int delta)` 
- `boolean compareAndSet(int i, int expect, int update)` 
- `void lazySet(int i, int newValue)`:在 set 之后的一小段时间内，仍有可能读到旧值

### ABA

JDK 提供了一个 AtomicStampedReference 类来解决 ABA 问题。AtomicStampedReference 除了要比较当前值和预期值外，还需要比较 Stamp。它的构造函数需要指定一个初始 Stamp

~~~java
AtomicStampedReference(V initialRef, int initialStamp)
~~~

AtomicStampReference 常用的几个方法如下：

- `V getRerference()`
- `int getStamp()`
- `boolean compareAndSet(V expectedReference, V newReference, int expectedStamp,int newStamp)`

### 空旋

可以以空间换时间来解决 CAS 恶性空旋，较为常见的方案为：

- 分散操作热点，使用 LongAdder 替代基础原子类 AtomicLong
- 使用队列削峰，将发生 CAS 争用的线程加入一个队列中排队，降低 CAS 争用的激烈程度。JUC中非常重要的基础类 AQS（抽象队列同步器）就是这么做的。

## 集合

## AQS

