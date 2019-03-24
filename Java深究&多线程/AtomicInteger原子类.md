## AtomicInteger 原子类的作用



- #### 多线程操作，Synchronized 性能开销太大`count++`并**不是原子**操作。因为`count++`需要经过`读取-修改-写入`三个步骤。

  - `count++`并**不是原子**操作。因为`count++`需要经过`读取-修改-写入`三个步骤。
  - 可以这样做：

    ``` java
    public synchronized void increase() {
        count++;
    }
    ```

  - Synchronized锁是独占的，意味着如果有别的线程在执行，当前线程只能是等待！



- ####  用CAS操作

  - CAS有3个操作数：

    > - 内存值V
    > - 旧的预期值A
    > - 要修改的新值B

  - 当多个线程尝试使用CAS同时更新同一个变量时，**只有其中一个线程能更新变量的值**(A和内存值V相同时，将内存值V修改为B)，而其它线程都失败，失败的线程并不会被挂起，而是被告知这次竞争中失败，**并可以再次尝试(或者什么都不做)**。

  - 我们可以发现CAS有两种情况：

    > - 如果内存值V和我们的预期值A**相等**，则将内存值修改为B，操作成功！
    > - 如果内存值V和我们的预期值A**不相等**，一般也有两种情况：
    >   - 重试(自旋)
    >   - 什么都不做

  - 理解CAS的核心就是：

    > **CAS是原子性的**，虽然你可能看到比较后再修改(compare and swap)觉得会有两个操作，但终究是原子性的！



- #### 原子变量类在`java.util.concurrent.atomic`包下，总体来看有这么多个

  > - 基本类型：
  >   - AtomicBoolean：布尔型
  >   - AtomicInteger：整型
  >   - AtomicLong：长整型
  > - 数组：
  >   - AtomicIntegerArray：数组里的整型
  >   - AtomicLongArray：数组里的长整型
  >   - AtomicReferenceArray：数组里的引用类型
  > - 引用类型：
  >   - AtomicReference：引用类型
  >   - AtomicStampedReference：带有版本号的引用类型
  >   - AtomicMarkableReference：带有标记位的引用类型
  > - 对象的属性
  >   - AtomicIntegerFieldUpdater：对象的属性是整型
  >   - AtomicLongFieldUpdater：对象的属性是长整型
  >   - AtomicReferenceFieldUpdater：对象的属性是引用类型
  > - JDK8新增DoubleAccumulator、LongAccumulator、DoubleAdder、LongAdder
  >   - 是对AtomicLong等类的改进。比如LongAccumulator与LongAdder在高并发环境下比AtomicLong更高效。

  - Atomic包里的类基本都是使用**Unsafe**实现的包装类
  - Unsafe里边有几个我们喜欢的方法(CAS)：

    ```java
    // 第一和第二个参数代表对象的实例以及地址，第三个参数代表期望值，第四个参数代表更新值
    public final native boolean compareAndSwapObject(Object var1, long var2, Object var4, Object var5);
    
    public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
    
    public final native boolean compareAndSwapLong(Object var1, long var2, long var4, long var6);  
    ```



- #### 原子变量类使用

  ```java
  class Count{
      // 共享变量(使用AtomicInteger来替代Synchronized锁)
      private AtomicInteger count = new AtomicInteger(0);
      public Integer getCount() {
          return count.get();
      }
      public void increase() {
          count.incrementAndGet();
      }
  }
  ```



- #### 原子类的ABA问题

  - 下面的操作都可以正常执行完的，这样会发生什么问题呢？？线程C无法得知线程A和线程B修改过的count值，这样是有**风险**的。

    > 1. 现在我有一个变量`count=10`，现在有三个线程，分别为A、B、C
    > 2. 线程A和线程C同时读到count变量，所以线程A和线程C的内存值和预期值都为10
    > 3. 此时线程A使用CAS将count值修改成100
    > 4. 修改完后，就在这时，线程B进来了，读取得到count的值为100(内存值和预期值都是100)，将count值修改成10
    > 5. 线程C拿到执行权，发现内存值是10，预期值也是10，将count值修改成11

    

- #### 解决ABA问题

  > 要解决ABA的问题，我们可以使用JDK给我们提供的AtomicStampedReference和AtomicMarkableReference类。
  >
  > 简单来说就是在给为这个对象提供了一个**版本**，并且这个版本如果被修改了，是自动更新的。
  >
  > 原理大概就是：维护了一个Pair对象，Pair对象存储我们的对象引用和一个stamp值。每次CAS比较的是两个Pair对象

  

- #### LongAdder 性能比 AtomicLong 要好

  - 使用AtomicLong时，在高并发下大量线程会同时去竞争更新**同一个原子变量**，但是由于同时只有一个线程的CAS会成功，所以其他线程会不断尝试自旋尝试CAS操作，这会浪费不少的CPU资源。

  - 而LongAdder可以概括成这样：内部核心数据value**分离**成一个数组(Cell)，每个线程访问时,通过哈希等算法映射到其中一个数字进行计数，而最终的计数结果，则为这个数组的**求和累加**。
    - 简单来说就是将一个值分散成多个值，在并发的时候就可以**分散压力**，性能有所提高。




