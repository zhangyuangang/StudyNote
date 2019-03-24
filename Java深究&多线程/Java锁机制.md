<span id = "0">

# Java锁机制

- [synchronized锁](#1)
- [Lock显式锁](#2)
- [公平锁](#3)
- [ReentrantLock锁](#4)
- [ReentrantReadWriteLock锁](#5)

<span id = "1">

<br/><br/>

# synchronized锁

- #### synchronized 简介

  > - synchronized是Java的一个**关键字**，它能够将**代码块(方法)锁起来**
  >
  >   > synchronized是一种**互斥锁**
  >   >
  >   > - **一次只能允许一个线程进入被锁住的代码块**
  >
  >   > synchronized是一种**内置锁/监视器锁**
  >   >
  >   > - Java中**每个对象**都有一个**内置锁(监视器,也可以理解成锁标记)**，而synchronized就是使用**对象的内置锁(监视器)**来将代码块(方法)锁定的！
  >
  > - synchronized保证了线程的**原子性**。
  >
  >   > 被保护的代码块是一次被执行的，没有任何线程会同时访问
  >
  > - synchronized还保证了**可见性**。
  >
  >   > 当执行完synchronized之后，修改后的变量对其他的线程是可见的

- #### synchronized底层原理

  > **同步代码块**：
  >
  > - monitorenter和monitorexit指令实现的
  >
  > **同步方法**（在这看不出来需要看JVM底层实现）
  >
  > - 方法修饰符上的ACC_SYNCHRONIZED实现。
  >
  > Java中的synchronized，通过使用内置锁，来实现对变量的同步操作，进而实现了**对变量操作的原子性和其他线程对变量的可见性**，从而确保了并发情况下的线程安全。

- #### synchronized一般我们用来修饰三种东西

  > - **修饰普通方法：用的是 该对象(内置锁)**
  >
  >   ```java
  >   // 修饰普通方法，此时用的锁是 该对象(内置锁)
  >   public synchronized void test() {
  >   }
  >   ```
  >
  > - **修饰代码块：用的是 该对象(内置锁)**
  >
  >   ```java
  >   // 修饰代码块，此时用的锁是  该对象(内置锁)--->this
  >   synchronized (this){
  >   }
  >   ```
  >
  >   > - 当然了，我们使用synchronized修饰代码块时未必使用this，还可以**使用其他的对象(随便一个对象都有一个内置锁)**
  >   > - 称之为-->**客户端锁**，这是**不建议使用的**
  >
  > - **修饰静态方法：用的是  类锁(类的字节码文件对象)：XX.class**
  >
  >   ```java
  >   // 修饰静态方法，此时用的锁是 类锁(类的字节码文件对象):	 XX.class
  >   public static synchronized void test() {
  >   }
  >   ```

  - synchronized修饰静态方法获取的是类锁(类的字节码文件对象)，synchronized修饰普通方法或代码块获取的是对象锁。
    - 它俩是不冲突的，也就是说：**获取了类锁的线程和获取了对象锁的线程是不冲突的**！

- #### 可重入

  ```JAVA
  public class Widget {
      // 锁住了
      public synchronized void doSomething() {
      }
  }
  public class LoggingWidget extends Widget {
      // 锁住了
      public synchronized void doSomething() {
          System.out.println(toString() + ": calling doSomething");
          super.doSomething();
      }
  }
  ```

  > - 当线程A进入到LoggingWidget的`doSomething()`方法时，**此时拿到了LoggingWidget实例对象的锁**。
  > - 随后在方法上又调用了父类Widget的`doSomething()`方法，它**又是被synchronized修饰**。
  > - 那现在我们LoggingWidget实例对象的锁还没有释放，进入父类Widget的`doSomething()`方法**还需要一把锁吗？**
  > - **不需要的！**
  > - 因为**锁的持有者是“线程”，而不是“调用”**。线程A已经是有了LoggingWidget实例对象的锁了，当再需要的时候可以继续**“开锁”**进去的！
  > - 这就是内置锁的**可重入性**。

- #### 怎么释放

  > 1. 当方法(代码块)执行完毕后会**自动释放锁**，不需要做任何的操作。
  > 2. **当一个线程执行的代码出现异常时，其所持有的锁会自动释放**。
  >
  > - 不会由于异常导致出现死锁现象

<span>[回到顶部](#0)</span>

<span id = "2">

<br/><br/>

# Lock显式锁

- #### Lock显示锁简介

  > - Lock显式锁是JDK1.5之后才有的，之前我们都是使用Synchronized锁来使线程安全的
  >
  > - Lock显式锁是一个接口
  >
  > - Lock方式来获取锁**支持中断、超时不获取、是非阻塞的**
  > - **提高了语义化**，哪里加锁，哪里解锁都得写出来
  > - **Lock显式锁可以给我们带来很好的灵活性，但同时我们必须手动释放锁**
  > - 支持Condition条件对象
  > - **允许多个读线程同时访问共享资源**

- #### synchronized锁和Lock锁使用哪个

  > Lock锁在刚出来的时候很多性能方面都比Synchronized锁要好，但是从JDK1.6开始Synchronized锁就做了各种的优化
  >
  > - 优化操作：适应自旋锁，锁消除，锁粗化，轻量级锁，偏向锁。

  - 所以，到现在Lock锁和Synchronized锁的性能其实**差别不是很大**！而Synchronized锁用起来又特别简单。**Lock锁还得顾忌到它的特性，要手动释放锁才行**(如果忘了释放，这就是一个隐患)
  - 所以说，我们**绝大部分时候还是会使用Synchronized锁**，用到了Lock锁提及的特性，带来的灵活性才会考虑使用Lock显式锁

<span>[回到顶部](#0)</span>

<span id = "3">

<br/><br/>

# 公平锁

> 公平锁理解起来非常简单：
>
> - 线程将按照它们**发出请求的顺序来获取锁**
>
> 非公平锁就是：
>
> - 线程发出请求的时可以**“插队”**获取锁
>
> Lock和synchronize都是**默认使用非公平锁的**。如果不是必要的情况下，不要使用公平锁
>
> - 公平锁会来带一些性能的消耗的

<span>[回到顶部](#0)</span>

<span id = "4">

<br/><br/>

# ReentrantLock锁

- **AQS是ReentrantLock的基础，AQS是构建锁、同步器的框架**

> - 比synchronized更有伸缩性(灵活)
>   - 支持公平锁(是相对公平的)，支持非公平锁
> - 使用时最标准用法是在try之前调用lock方法，在finally代码块释放锁

```JAVA
class X {
    private final ReentrantLock lock = new ReentrantLock();
    public void m() { 
        lock.lock();
        try {
            // ... method body
        } finally {
            lock.unlock()
        }
    }
}
```

<span>[回到顶部](#0)</span>

<span id = "5">

<br/><br/>

# ReentrantReadWriteLock锁

- synchronized内置锁和ReentrantLock都是**互斥锁**(一次只能有一个线程进入到临界区(被锁定的区域))

  > ReentrantReadWriteLock是一个**读写锁**：
  >
  > - 在**读**取数据的时候，可以**多个线程同时进入到到临界区**(被锁定的区域)
  > - 在**写**数据的时候，无论是读线程还是写线程都是**互斥**的
  >
  > **在读的时候可以共享，在写的时候是互斥的**
  >
  > - 读锁不支持条件对象，写锁支持条件对象
  > - 读锁不能升级为写锁，写锁可以降级为读锁
  > - 读写锁也有公平和非公平模式
  > - **读锁支持多个读线程进入临界区，写锁是互斥的**

