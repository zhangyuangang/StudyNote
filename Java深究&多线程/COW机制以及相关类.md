# COW机制以及相关类



- #### Vector和SynchronizedList

  - 我们知道ArrayList是用于替代Vector的，Vector是线程安全的容器。因为它几乎在每个方法声明处都加了**synchronized关键字**来使容器安全。
  - 如果使用`Collections.synchronizedList(new ArrayList())`来使ArrayList变成是线程安全的话，也是几乎都是每个方法都加上synchronized关键字的，只不过**它不是加在方法的声明处，而是方法的内部**。

  - 多线程下for循环迭代Vector或者SynchronizedList，进行delete和get操作会发生数组下标错误的异常。
    - 在JDK5以后，Java推荐使用`for-each`(迭代器)来遍历我们的集合，好处就是**简洁、数组索引的边界值只计算一次**。
  - 如果使用`for-each`(迭代器)来做上面的操作，会抛出ConcurrentModificationException异常。
  - 如果想要完美解决上面所讲的问题，我们可以在**遍历前加锁**：
    - **遍历一下容器都要我加上锁，这这这不是要慢死了吗**。的确是挺慢的。因为加锁粒度太大。

  

- #### CopyOnWriteArrayList是同步List的替代品，CopyOnWriteArraySet是同步Set的替代品。

  > - Hashtable、Vector加锁的粒度大(直接在方法声明处使用synchronized)
  > - ConcurrentHashMap、CopyOnWriteArrayList加锁粒度小(用各种的方式来实现线程安全，比如我们知道的ConcurrentHashMap用了cas锁、volatile等方式来实现线程安全..)
  > - JUC下的线程安全容器在遍历的时候**不会**抛出ConcurrentModificationException异常

  - 所以一般来说，我们都会**使用JUC包下给我们提供的线程安全容器**，而不是使用老一代的线程安全容器。

  

- #### CopyOnWriteArrayList实现原理

  > - CopyOnWriteArrayList是线程安全容器(相对于ArrayList)，底层通过**复制数组**的方式来实现。
  > - CopyOnWriteArrayList在遍历的使用不会抛出ConcurrentModificationException异常，并且遍历的时候就不用额外加锁
  > - 元素可以为null

  ```java 
  /** 可重入锁对象 */
      final transient ReentrantLock lock = new ReentrantLock();
      /** CopyOnWriteArrayList底层由数组实现，volatile修饰 */
      private transient volatile Object[] array;
  
      final Object[] getArray() {
          return array;
      }
      final void setArray(Object[] a) {
          array = a;
      }
      // 初始化CopyOnWriteArrayList相当于初始化数组
      public CopyOnWriteArrayList() {
          setArray(new Object[0]);
      }
  ```

  - CopyOnWriteArrayList底层就是数组，加锁就交由ReentrantLock来完成。

    - 通过代码我们可以知道：在add()，set()，remove() 的时候就上锁，并**复制一个新数组，增加操作在新数组上完成，将array指向到新数组中**，最后解锁。
    - **在修改时，复制出一个新数组，修改的操作在新数组中完成，最后将新数组交由array变量指向**。
    - **写加锁，读不加锁**

    

- #### CopyOnWriteArrayList缺点

  - **内存占用**：如果CopyOnWriteArrayList经常要增删改里面的数据，经常要执行`add()、set()、remove()`的话，那是比较耗费内存的。
    - 因为我们知道每次`add()、set()、remove()`这些增删改操作都要**复制一个数组**出来。
  - **数据一致性**：CopyOnWrite容器**只能保证数据的最终一致性，不能保证数据的实时一致性**。
    - 从上面的例子也可以看出来，比如线程A在迭代CopyOnWriteArrayList容器的数据。线程B在线程A迭代的间隙中将CopyOnWriteArrayList部分的数据修改了(已经调用`setArray()`了)。但是线程A迭代出来的是原有的数据。
