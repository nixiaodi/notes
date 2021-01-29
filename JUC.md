# JUC

## 1 SaleTicket

核心：在高内聚低耦合的前提下，`线程`  `操作`  `资源类`

> 对于资源类的操作应该保存在资源内部
>
> 线程只是对资源已经封装好的操作进行调用

### 1.1 资源类

```java
@Slf4j
public class Ticket {
    private int num = 1000;

    Lock lock = new ReentrantLock();

    public void sell() {
        lock.lock();
        try {
            if (num > 0) {
                log.info(Thread.currentThread().getName() + "卖出第" + num-- + "张票，还剩下" + num + "张票");
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

> 资源类中包含共享变量以及对资源操作的封装

### 1.2 线程

```java
public class SaleSystem {
    public static void main(String[] args) {
        Ticket ticket = new Ticket();

        new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                ticket.sell();
            }
        },"A").start();

        new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                ticket.sell();
            }
        },"B").start();

        new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                ticket.sell();
            }
        },"C").start();
    }
}
```

> 创建资源类，创建线程用于操作资源类

## 2 生产者消费者模型

核心：多线程之间通信，主要步骤：

- 判断条件是否成立，成立就阻塞
- 不成立则执行操作
- 执行完操作唤醒其他线程

### 2.1 一般生产消费模型

#### 2.1.1 资源类

```java
@Slf4j
public class Resource {
    private int num = 0;

    public synchronized void increment() throws InterruptedException {
        if (num != 0) {
            this.wait();
        }
        num++;
        log.info(Thread.currentThread().getName() + num);
        this.notifyAll();
    }

    public synchronized void decrement() throws InterruptedException {
        if (num == 0) {
            this.wait();
        }
        num--;
        log.info(Thread.currentThread().getName() + num);
        this.notifyAll();
    }
}
```

> 通过synchronized关键字对整个方法加锁，通过if判断是否需要阻塞该线程，如果线程没有阻塞，在执行完操作之后唤醒其他线程

#### 2.1.2 线程

```java
public class Processor {
    public static void main(String[] args) {
        Resource resource = new Resource();

        new Thread(() -> {
            try {
                for (int i = 0; i < 100; i++) {
                    resource.increment();
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        },"生产者").start();

        new Thread(() -> {
            try {
                for (int i = 0; i < 100; i++) {
                    resource.decrement();
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        },"消费者").start();
    }
}
```

通过结果可以看出传统方式可以实现生产消费模型

`但是思考一个问题，如果现在存在多个生产者和消费者会出现什么情况呢？`

> 多线程通信过程中如果在对条件进行判断时使用的是if条件，就可能出现虚假唤醒
>
> 考虑一种情况，最初资源数量为0，那么两个消费者线程都会处于阻塞状态，假设生产者A先抢到CPU，则判断通过之后操作资源数量为1，这时消费者A和B实际上都是从阻塞中被唤醒，再假设消费者A先抢到CPU，则执行操作另资源数量变为0，之后消费者B抢到CPU，由于这里使用的是if判断，故对于消费者B已经经过了判断，直接执行操作，令资源数量变为-1

故在多线程通信中，对临界条件进行判断时一定不能使用if，需要使用while

#### 2.1.3 优化资源类

```java
@Slf4j
public class Resource {
    private int num = 0;

    public synchronized void increment() throws InterruptedException {
        while (num != 0) {
            this.wait();
        }
        num++;
        log.info(Thread.currentThread().getName() + "\t" + num);
        this.notifyAll();
    }

    public synchronized void decrement() throws InterruptedException {
        while (num == 0) {
            this.wait();
        }
        num--;
        log.info(Thread.currentThread().getName() + "\t" + num);
        this.notifyAll();
    }
}
```

### 2.2 新版生产消费模型

#### 2.1.1 资源类

```java
@Slf4j
public class Resource {

    private int num = 0;
    Lock lock = new ReentrantLock();
    Condition condition = lock.newCondition();

    public void increment() throws InterruptedException {
        lock.lock();
        try {
            while (num != 0) {
                condition.await();
            }
            num++;
            log.info(Thread.currentThread().getName() + "\t" + num);
            condition.signalAll();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void decrement() throws InterruptedException {
        lock.lock();
        try {
            while (num == 0) {
                condition.await();
            }
            num--;
            log.info(Thread.currentThread().getName() + "\t" + num);
            condition.signalAll();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

> Condition接口实例和Lock实例进行绑定，因此可以通过Condition实例对持有锁和处于阻塞状态的代码块进行操作
>
> Condition接口实例等同于替换了对象监视器(Object monitor)方法(wait,notify,notifyAll)，替换为(await,sign,signAll)
>
> 实际上当while判断条件成立时，await方法也会类似于wait方法`释放锁`(注意：`sleep方法虽然阻塞但是不会释放锁`)
>
> 可以看出外部的lock锁是为了实现互斥，而内部的condition是为了实现同步

### 2.3 Lock锁实现精确通知顺序

#### 2.3.1 资源类

```java
@Slf4j
public class Resource2 {
    private int number = 1;

    private Lock lock = new ReentrantLock();
    private Condition condition1 = lock.newCondition();
    private Condition condition2 = lock.newCondition();
    private Condition condition3 = lock.newCondition();

    public void print5() {
        lock.lock();
        try {
            while (number != 1) {
                condition1.await();
            }
            for (int i = 0; i < 5; i++) {
                System.out.println(Thread.currentThread().getName());
            }
            number = 2;
            condition2.signal();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void print10() {
        lock.lock();
        try {
            while (number != 2) {
                condition2.await();
            }
            for (int i = 0; i < 10; i++) {
                System.out.println(Thread.currentThread().getName());
            }
            number = 3;
            condition3.signal();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void print15() {
        lock.lock();
        try {
            while (number != 3) {
                condition3.await();
            }
            for (int i = 0; i < 15; i++) {
                System.out.println(Thread.currentThread().getName());
            }
            number = 1;
            condition1.signal();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

> 同一个lock对象，但是从lock对象中映射出三个condition对象(也就是一把锁多个钥匙)
>
> 在进行判断时如果满足条件就用自己的钥匙上锁，当不满足时就执行操作，操作之后调用指定钥匙去开锁(也就是唤醒指定的操作)

#### 2.3.2 线程

```java
public class Processor {
    public static void main(String[] args) {
        Resource2 resource2 = new Resource2();

        new Thread(resource2::print5,"A").start();

        new Thread(resource2::print10,"B").start();

        new Thread(resource2::print15,"C").start();
    }
}
```

## 3 八锁现象

假设现在有一个Phone资源类，提供了两种操作方式

```java
class Phone {
    public synchronized void sendEmail() {
        System.out.println("sendEmail...");
    }

    public synchronized void sendSms() {
        System.out.println("sendSms...");
    }
}
```

### 3.1 标准访问

```java
public class Lock8 {
    public static void main(String[] args) throws InterruptedException {
        Phone phone = new Phone();

        new Thread(phone::sendEmail,"A").start();

        TimeUnit.SECONDS.sleep(2);

        new Thread(phone::sendSms,"B").start();
    }
}
```

> 当线程A就绪之后，main线程休眠2秒钟之后，才令B线程就绪，所以一定会先打印邮件再打印短信

### 3.2 邮件方法休眠4秒钟

修改资源类

```java
class Phone {
    public synchronized void sendEmail() {
        try {
            TimeUnit.SECONDS.sleep(4);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("sendEmail...");
    }

    public synchronized void sendSms() {
        System.out.println("sendSms...");
    }
}
```

执行线程

```java
public class Lock8 {
    public static void main(String[] args) throws InterruptedException {
        Phone phone = new Phone();

        new Thread(phone::sendEmail,"A").start();

        TimeUnit.SECONDS.sleep(2);

        new Thread(phone::sendSms,"B").start();
    }
}
```

> 实际上还是先打印邮件，虽然邮件睡眠4秒钟
>
> 然而由于main线程休眠2秒钟，这是邮件线程已经抢夺到执行权，并且加锁；而slepp方法不会释放锁

### 3.3 新增普通方法hello()，先打印邮件还是hello

修改资源类

```java
class Phone {
    public synchronized void sendEmail() {
        try {
            TimeUnit.SECONDS.sleep(4);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("sendEmail...");
    }

    public synchronized void sendSms() {
        System.out.println("sendSms...");
    }
    
    public void hello() {
        System.out.println("hello...");
    }
}
```

> 这里的hello方法并没有加锁

线程操作

```java
public class Lock8 {
    public static void main(String[] args) throws InterruptedException {
        Phone phone = new Phone();

        new Thread(phone::sendEmail,"A").start();

        TimeUnit.SECONDS.sleep(2);

        new Thread(phone::hello,"B").start();
    }
}
```

> 结果是先打印hello，再打印邮件
>
> 因为邮件线程虽然执行时先抢到锁，然而hello线程并没有上锁，只用等待main线程休眠2秒钟之后，就可以就绪--运行

### 3.4 两部手机，先打印邮件还是短信

资源类不变，线程操作

```java
public class Lock8 {
    public static void main(String[] args) throws InterruptedException {
        Phone phone = new Phone();
        Phone phone2 = new Phone();

        new Thread(phone::sendEmail,"A").start();

        TimeUnit.SECONDS.sleep(2);

        new Thread(phone2::sendSms,"B").start();
    }
}
```

> 结果是先打印短信，再打印邮件
>
> 因为两部手机存在两把锁，实际上两个线程持有的锁不是同一个，也就是说邮件线程及时休眠不释放锁，也不妨碍短信线程
>
> 实际上这里的锁是`对象锁`，两个不同的Phone对象持有的对象锁也不同

### 3.5 两个静态同步方法，同一部手机

修改资源类

```java
class Phone {
    public static synchronized void sendEmail() {
        try {
            TimeUnit.SECONDS.sleep(4);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("sendEmail...");
    }

    public static synchronized void sendSms() {
        System.out.println("sendSms...");
    }

    public void hello() {
        System.out.println("hello...");
    }
}
```

修改线程操作

```java
public class Lock8 {
    public static void main(String[] args) throws InterruptedException {
        Phone phone = new Phone();

        new Thread(() -> {
            phone.sendEmail();
        },"A").start();

        TimeUnit.SECONDS.sleep(2);

        new Thread(() -> {
            phone.sendSms();
        },"A").start();
    }
}
```

> 结果是先打印邮件，再打印短信
>
> 因为这里拿到的同步锁实际上Phone类的class对象，不管怎么样都只有唯一一个

### 3.6 两个静态同步方法，两部手机

修改线程操作

```java
public class Lock8 {
    public static void main(String[] args) throws InterruptedException {
        Phone phone = new Phone();
        Phone phone2 = new Phone();

        new Thread(() -> {
            phone.sendEmail();
        },"A").start();

        TimeUnit.SECONDS.sleep(2);

        new Thread(() -> {
            phone2.sendSms();
        },"A").start();
    }
}
```

> 结果依然是先打印邮件，再打印短信
>
> 因为同步锁是Phone类的class对象

### 3.7 一个普通、一个静态同步方法，两部手机

修改资源类

```java
class Phone {
    public static synchronized void sendEmail() {
        try {
            TimeUnit.SECONDS.sleep(4);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("sendEmail...");
    }

    public synchronized void sendSms() {
        System.out.println("sendSms...");
    }

    public void hello() {
        System.out.println("hello...");
    }
}
```

修改线程操作

```java
public class Lock8 {
    public static void main(String[] args) throws InterruptedException {
        Phone phone = new Phone();

        new Thread(() -> {
            phone.sendEmail();
        },"A").start();

        TimeUnit.SECONDS.sleep(2);

        new Thread(() -> {
            phone.sendSms();
        },"A").start();
    }
}
```

> 结果是先打印短信，再打印邮件
>
> 因为短信线程拿到的是phone的对象锁，而邮件线程拿到的是Phone类的class对象锁，线程之间互不干扰

### 3.8 一个静态、一个普通同步方法，两部手机

修改线程操作

```java
public class Lock8 {
    public static void main(String[] args) throws InterruptedException {
        Phone phone = new Phone();
        Phone phone2 = new Phone();

        new Thread(() -> {
            phone.sendEmail();
        },"A").start();

        TimeUnit.SECONDS.sleep(2);

        new Thread(() -> {
            phone2.sendSms();
        },"A").start();
    }
}
```

> 结果依然是先打印短信，再打印邮件
>
> 因为本质上邮件线程持有的是class对象锁，而短信线程持有的是phone2对象锁

## 4 集合分析

### 4.1 ArrayList分析

假设一种场景，创建一个ArrayList，通过不同的线程进行读写操作

```java
public class NotSafeDemo {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();

        for (int i = 0; i < 30; i++) {
            new Thread(() -> {
                list.add(UUID.randomUUID().toString().substring(0,8));
                System.out.println(list);
            },String.valueOf(i)).start();
        }
    }
}
```

> 通过for循环建立了30个线程对list进行读写操作，而ArrayList线程不安全，不同的线程执行存在随机性，可能某一个线程在进行写操作时，另外一个线程也在进行写操作

所以执行时会报出`ConcurrentModificationException`(并发修改异常)

```bash
Exception in thread "8" Exception in thread "20" Exception in thread "15" Exception in thread "21" Exception in thread "23" Exception in thread "28" java.util.ConcurrentModificationException
```

为什么会出现这种情况呢？(或者说为什么AarrayList线程不安全呢)

分析下ArrayList的源码

```java
public void add(int index, E element) {
    rangeCheckForAdd(index);

    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,size - index);
    elementData[index] = element;
    size++;
}
```

> add方法本身没有加锁，在高并发的情况下任何一个线程都可以直接对其进行读写

Vector线程安全，为什么呢？

```java
public synchronized boolean add(E e) {
    modCount++;
    ensureCapacityHelper(elementCount + 1);
    elementData[elementCount++] = e;
    return true;
}
```

> add方法通过synchronized关键字加锁，保证了一个线程在进行操作时，其他的线程只能阻塞

然而虽然保证了线程的安全，却极大的降低了效率(这也是为什么vector不常用的原因)

#### 4.1.1 通过Collections.synchronizedList优化

现在已经知道ArrayList线程不安全，如何解决该问题呢？

可以通过`Collections.synchronizedList(new ArrayList<>())`操作原有的ArrayList使得线程安全

`synchronizedList`方法是如何执行的呢？

```java
public static <T> List<T> synchronizedList(List<T> list) {
    return (list instanceof RandomAccess ?
            new SynchronizedRandomAccessList<>(list) :
            new SynchronizedList<>(list));
}
```

> 首先判断参数中的list是否属于RandomAccess接口的实现类对象，不管是否返回的实际上都是加锁的对象

#### 4.1.2 创建CopyOnWriteArrayList对象

单线程时可以使用ArrayList，高并发则使用CopyOnWriteArrayList

那么为什么CopyOnWriteArrayList在高并发情况下使用呢？

> 因为CopyOnWriteArrayList不仅能在保证线程安全的前提下，达到数据的最终一致性，那么是如何实现的呢？

1、首先CopyOnWriteArrayList中维护了一个数组和一个lock锁

![image-20210124203201667](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210124203201667.png)

2、当进行写操作时

```java
/**
 * Appends the specified element to the end of this list.
 *
 * @param e element to be appended to this list
 * @return {@code true} (as specified by {@link Collection#add})
 */
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

> 首选拿出类中维护的lock锁并上锁，之后获取到当前的数组对象，然后通过`Arrays.copyOf`方法复制了当前的数组并使长度加1，然后将添加的元素放入数组中，最后通过set方法将新的数组对象返回；然后再解锁
>
> 核心点：
>
> 1、复制原有的数组，不影响读操作
>
> 2、进行写操作时加锁

3、当进行读操作时

```java
/**
 * {@inheritDoc}
 *
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public E get(int index) {
    return get(getArray(), index);
}
```

> 读数据时并没有加锁，虽然不是即时获取到最新的数据，然而也保证了数据的最终一致性

### 4.2 HashSet分析

类似于Arraylist，HashSet同样线程不安全

同样可以通过`Collections.synchronizedSet()`令原有的HashSet保证线程安全，然而这样会极大降低读写效率

同样的，也可以使用`new CopyOnWriteArraySet<>()`在保证线程安全的前提下，提高读写效率

#### 4.2.1 扩展

通过源码看出，HashSet底层维护了一个HashMap和一个Object对象

![image-20210124205019463](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210124205019463.png)

那么这两个属性的作用是什么呢？

```java
public HashSet() {
    map = new HashMap<>();
}
```

> 从HashSet的构造器中看出，map就是一个HashMap对象

当调用add方法时

```java
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

本质上就是向map中放数据，key就是add方法的参数，而value实际上就是一个固定值，也就是一开始维护的Object对象

当调用remove方法时

```java
public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}
```

本质上就是从map中移除一个数据，我们知道map的remove方法返回的是对应key的value，而value就是一个固定值PRESENT

> 总结如下：
>
> 1、HashSet底层就是HashMap，只不过只使用了HashMap的key(因为key唯一)，而value就是一个固定值Object对象

### 4.3 HashMap分析

类似于Arraylist，HashMap同样线程不安全

同样可以通过`Collections.synchronizedMap()`令原有的HashMap保证线程安全，然而这样会极大降低读写效率

然而并不存在`CopyOnWriteArrayMap<>()`，而是`ConcurrentHashMap<>()`

#### 4.3.1 ConcurrentHashMap分析

//TODO

## 5 Callable接口

我们知道如果需要新建一个线程，必须通过`new Thread()`这种方式实现

但是参考下Thread的构造器

![image-20210124215034560](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210124215034560.png)

还可以在构造器中传入Runable接口或者ThreadGroup线程组

那么如何使用Callable接口呢？

我们知道构造器中没有Callable接口作为参数，所以就必须想办法将Callable接口和Thread构造器联系起来

那么可以将Runable接口作为桥梁将两边联系起来

看下Runbale接口在API中的定义：

![image-20210124215340440](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210124215340440.png)

已知实现类中有一个FutureTask类，而FutureTask类又是什么样的呢？

![image-20210124215436046](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210124215436046.png)

FutureTask类的构造器中可以传入一个Callable接口对象，那么就将Callable接口和Thread联系到一起

### 5.1 Callable接口测试

```java
public class CallableDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        FutureTask futureTask = new FutureTask(new MyThread());

        new Thread(futureTask,"A").start();

        System.out.println(futureTask.get());
    }
}


class MyThread implements Callable<Integer> {

    @Override
    public Integer call() throws Exception {
        System.out.println("线程被调用");
        return 1024;
    }
}
```

> 首先MyThread继承了Callable接口(并指定泛型，泛型就是call方法的返回值类型)
>
> 然后在主线程中定义了FutureTask对象传入MyThread对象为参数，进而创建线程
>
> 最后调用了futureTask的get方法获取call方法中的返回值
>
> `个人理解：FutureTask类(从名字上是未来任务)，异步执行，而get方法就是获取异步执行之后的结果`

### 5.2 Callable接口和Runable接口的区别

| Callable                                               | Runable           |
| ------------------------------------------------------ | ----------------- |
| 方法存在返回值(返回值就是实现Callable接口时定义的泛型) | 没有返回值        |
| 可以抛异常                                             | 不能抛异常        |
| 落地方法为call方法                                     | 落地方法为run方法 |

### 5.3 Callable接口细节

#### 5.3.1 Callable接口中的思想

我们知道FutureTask对象可以通过get方法获取线程执行之后的结果，其实这本质就是异步的思想

![image-20210124224631207](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210124224631207.png)

假设每个步骤执行过程中需要5秒钟，而步骤3执行需要20秒

正常情况下多个步骤如果是同步执行的话，就是串行化执行，那么执行时间就是35秒

当进行异步操作时，对于步骤3另起一个线程，那么步骤1,2,4执行完成需要15秒，而在步骤3执行时定义为FutrueTask对象，从而主线程执行完成之后只需要等待主线程5秒钟，并接收到步骤3的结果；从而执行时间为20秒

`也就是说对于异步操作，我们只需要给异步操作一个新的线程，并对该线程进行监听，在此期间可以继续执行其他操作，当监听到异步操作的线程结束，就接收对应的返回值，从而提高效率`

#### 5.3.2 FutureTask.get()方法一般放在结尾

我们可以尝试下不将FutureTask.get()方法放在结尾会出现什么情况呢？

```java
public class CallableDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        FutureTask futureTask = new FutureTask(new MyThread());

        new Thread(futureTask,"A").start();

        System.out.println(futureTask.get());
        System.out.println("其他步骤已完成");
    }
}


class MyThread implements Callable<Integer> {

    @Override
    public Integer call() throws Exception {
        System.out.println("线程被调用");
        TimeUnit.SECONDS.sleep(3);
        return 1024;
    }
}
```

结果：

![image-20210124230121565](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210124230121565.png)

> 实际上结果中先阻塞着等待线程结果计算之后再打印"其他步骤已完成"

`本质上get()方法阻塞，如果将get()放在前面就会影响main线程的执行，故FutureTask对象结果的计算需要放在最后避免阻塞其他线程`

#### 5.3.3 FutureTask任务多线程并发访问时只会执行一次

测试：

```java
public class CallableDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        FutureTask futureTask = new FutureTask(new MyThread());

        new Thread(futureTask,"A").start();
        new Thread(futureTask,"B").start();

        System.out.println(futureTask.get());
        System.out.println("其他步骤已完成");
    }
}


class MyThread implements Callable<Integer> {

    @Override
    public Integer call() throws Exception {
        System.out.println("线程被调用");
        TimeUnit.SECONDS.sleep(3);
        return 1024;
    }
}
```

结果：

![image-20210124230121565](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210124230121565.png)

原因：

[FutureTask异步执行结果](https://blog.csdn.net/u010013573/article/details/87435850?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.control)

[FutureTask只会执行一次](https://blog.csdn.net/chenliguan/article/details/54345993)

[FutureTask源码](https://www.cnblogs.com/yaowen/p/11388001.html)

## 6 JUC辅助类

### 6.1 CountDownLatch

假设一个场景，主线程下面有6个子线程，主线程结束必须要在子线程都结束之后才能结束

模拟场景：

```java
public class CountDownLatchDemo {
    public static void main(String[] args) {
        for (int i = 0; i < 6; i++) {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + "已结束");
            },String.valueOf(i)).start();
        }

        System.out.println(Thread.currentThread().getName() + "已结束");
    }
}
```

结果：

```bash
0已结束
4已结束
2已结束
main已结束
3已结束
1已结束
5已结束
```

可以看出main线程可以会在其他线程没有结束之前结束，那么如何控制该顺序呢？

```java
public class CountDownLatchDemo {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(6);

        for (int i = 0; i < 6; i++) {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + "已结束");
                latch.countDown();
            },String.valueOf(i)).start();
        }
        latch.await();
        System.out.println(Thread.currentThread().getName() + "已结束");
    }
}
```

> CountDownLatch就是用于控制某个线程在其他线程结束之后才能执行

#### 6.1.1 构造器

```java
/**
 * Constructs a {@code CountDownLatch} initialized with the given count.
 *
 * @param count the number of times {@link #countDown} must be invoked
 *        before threads can pass through {@link #await}
 * @throws IllegalArgumentException if {@code count} is negative
 */
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```

传入的参数count从注释中可以看出就是`其他线程被调用的次数`

#### 6.1.2 countDown方法

```java
/**
 * Decrements the count of the latch, releasing all waiting threads if
 * the count reaches zero.
 *
 * <p>If the current count is greater than zero then it is decremented.
 * If the new count is zero then all waiting threads are re-enabled for
 * thread scheduling purposes.
 *
 * <p>If the current count equals zero then nothing happens.
 */
public void countDown() {
    sync.releaseShared(1);
}
```

从注释中可以看出就是每次调用时就会使计数减少1，当计数值为0时就会唤醒被阻塞的线程

#### 6.1.3 理解

其实就是调用`latch.await()`方法时在计数为0之前main线程会一直阻塞

而其他线程执行之后调用`latch.countDown()`方法时就会将计数减少1

### 6.2 CyclicBarric

我们知道CountDownLatch是倒着开始计数，而CyclicBarric就是正着开始计数，当数量到达指定的值之后，该进程就不会再阻塞

#### 6.2.1 构造器

```java
/**
 * Creates a new {@code CyclicBarrier} that will trip when the
 * given number of parties (threads) are waiting upon it, and which
 * will execute the given barrier action when the barrier is tripped,
 * performed by the last thread entering the barrier.
 *
 * @param parties the number of threads that must invoke {@link #await}
 *        before the barrier is tripped
 * @param barrierAction the command to execute when the barrier is
 *        tripped, or {@code null} if there is no action
 * @throws IllegalArgumentException if {@code parties} is less than 1
 */
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}
```

从注释中看出第一个参数`parties`就是指定需要阻塞等待多少个线程执行结束，而`barrierAction`参数就是不再阻塞时执行的线程

#### 6.2.2 await方法

```java
/**
 * Waits until all {@linkplain #getParties parties} have invoked
 * {@code await} on this barrier.
 */
    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }
```

await方法就是在其他线程执行操作结束之后进行调用，每调用一次CyclicBarrier的属性`parties`就会减少1，当数值为0时就会调用CyclicBarrier中的线程

#### 6.2.3 测试

```java
public class CyclicBarrierDemo {
    public static void main(String[] args) {
        CyclicBarrier barrier = new CyclicBarrier(7,()->{
            System.out.println("召唤神龙");
        });

        for (int i = 0; i < 7; i++) {
            final int temp = i;
            new Thread(() -> {
               System.out.println("第" + temp + "颗龙珠");
               try {
                   barrier.await();
               } catch (InterruptedException | BrokenBarrierException e) {
                   e.printStackTrace();
               }
           },String.valueOf(i)).start();
        }
    }
}
```

> 首先定义了一个CyclicBarrier指定了需要等待多少个线程先执行，定义了自己的线程执行操作
>
> 在其他线程进行操作结束之后通过调用CyclicBarrier对象的await方法减少`parties`属性的数量

### 6.3 Semaphore

信号量，主要用于多线程的并发控制和资源的互斥，其实也就是`操作系统中的互斥信号量和同步信号量`

假设一种情况，资源和线程都有多个，但是线程的数量大于资源的数量

```java
public class SemaphoreDemo {
    public static void main(String[] args) {
        Semaphore semaphore = new Semaphore(3);

        for (int i = 0; i < 6; i++) {
            new Thread(() -> {
                try {
                    semaphore.acquire();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + "获取了资源");
                try {
                    TimeUnit.SECONDS.sleep(3);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    semaphore.release();
                    System.out.println(Thread.currentThread().getName() + "释放了资源");
                }
            },String.valueOf(i)).start();
        }
    }
}
```

> 通过Semaphore构造器创建了一个资源数量为3的信号量
>
> 紧接着创建了6个线程，不同线程之间通过`semaphore.acquire()`获取资源，明显当资源数量为0时，其他线程就会阻塞
>
> 假设线程休眠3秒就是进行了操作，最后通过`semaphore.release()`释放了资源，释放之后就可以通知`其他阻塞的线程`获取该资源

#### 6.3.1 构造器

```java
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}
/**
 * Creates a {@code Semaphore} with the given number of
 * permits and the given fairness setting.
 * @param fair {@code true} if this semaphore will guarantee
 *        first-in first-out granting of permits under contention,
 *        else {@code false}
*/
 public Semaphore(int permits, boolean fair) {
 	sync = fair ? new FairSync(permits) : new NonfairSync(permits);
 }
```

构造器中如果传入两个参数，第一个参数代表资源数量，第二参数代表判断是否公平

#### 6.3.2 acquire方法

```java
/**
 * Acquires a permit from this semaphore, blocking until one is
 * available, or the thread is {@linkplain Thread#interrupt interrupted}.
 *
 * <p>Acquires a permit, if one is available and returns immediately,
 * reducing the number of available permits by one.
*/
public void acquire() throws InterruptedException {
	sync.acquireSharedInterruptibly(1);
}
```

通过调用已经创建的Semaphore对象的acquire方法获取一个资源

#### 6.3.3 release方法

```java
/**
 * Releases a permit, returning it to the semaphore.
 *
 * <p>Releases a permit, increasing the number of available permits by
 * one.  If any threads are trying to acquire a permit, then one is
 * selected and given the permit that was just released.  That thread
 * is (re)enabled for thread scheduling purposes.
 *
 * <p>There is no requirement that a thread that releases a permit must
 * have acquired that permit by calling {@link #acquire}.
 * Correct usage of a semaphore is established by programming convention
 * in the application.
 */
public void release() {
    sync.releaseShared(1);
}
```

通过调用已经创建的Semaphore对象的release方法释放一个资源

### 6.4 ReadWriteLock

多个线程同时读同一个资源类没有任何问题，为了满足并发量，读取共享资源可以同时进行

但是如果有一个线程想要写共享资源，就不应该再有其他线程可以对该资源进行读或写

> 总结下来就是
>
> 1、读-读共享
>
> 2、写-读独占
>
> 3、写-写独占

如果假设读写都不加锁的条件下，会出现什么现象呢？

```java
public class ReadWriteLockDemo {
    public static void main(String[] args) {
        MyCache cache = new MyCache();

        for (int i = 0; i < 5; i++) {
            final String temp = String.valueOf(i);
            new Thread(() -> {
                cache.put(temp,temp);
            },temp).start();
        }

        for (int i = 0; i < 5; i++) {
            final String temp = String.valueOf(i);
            new Thread(() -> {
                cache.get(temp);
            },temp).start();
        }
    }
}

class MyCache {
    private volatile HashMap<String, Object> map = new HashMap<>();

    public void put(String key,Object value) {
        System.out.println(Thread.currentThread().getName() + "准备写入数据" + key);
        map.put(key,value);
        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + "写入数据完成");
    }

    public Object get(String key) {
        System.out.println(Thread.currentThread().getName() + "准备读取数据");
        Object res = map.get(key);
        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + "读取数据完成" + res);
        return res;
    }
}
```

这里读和写都没有加锁，同时分别开了5个线程用于读和写

结果：

![image-20210125114904914](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210125114904914.png)

> 当线程0准备写入数据时，其他线程直接抢占了CPU执行权，很明显这和事务的ACID违背(原子性)

#### 6.4.1 使用ReadWriteLock控制读写

```java
public class ReadWriteLockDemo {
    public static void main(String[] args) {
        MyCache cache = new MyCache();

        for (int i = 0; i < 5; i++) {
            final String temp = String.valueOf(i);
            new Thread(() -> {
                cache.put(temp,temp);
            },temp).start();
        }

        for (int i = 0; i < 5; i++) {
            final String temp = String.valueOf(i);
            new Thread(() -> {
                cache.get(temp);
            },temp).start();
        }
    }
}

class MyCache {
    private volatile HashMap<String, Object> map = new HashMap<>();
    private ReadWriteLock lock = new ReentrantReadWriteLock();

    public void put(String key,Object value) {
        lock.writeLock().lock();
        try {
            System.out.println(Thread.currentThread().getName() + "准备写入数据" + key);
            map.put(key,value);
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() + "写入数据完成");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.writeLock().unlock();
        }
    }

    public Object get(String key) {
        lock.readLock().lock();
        try {
            System.out.println(Thread.currentThread().getName() + "准备读取数据");
            Object res = map.get(key);
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() + "读取数据完成" + res);
            return res;
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.readLock().unlock();
        }
        return null;
    }
}
```

> 首先ReadWriteLock只是一个接口，具体操作时需要使用实现类ReentrantReadWriteLock

我们只是要对写操作进行控制，那么为什么读操作还要加锁呢？

> 为了防止在进行读操作时阻止写操作，但是如果多个读操作可以同步进行
>
> `读锁是共享锁，多个读操作共享；写锁是排他锁，其他任何操作不能共享`

#### 6.4.2 writeLock和readLock

```java
public class ReentrantReadWriteLock {
	/** Inner class providing readlock */
    private final ReentrantReadWriteLock.ReadLock readerLock;
    /** Inner class providing writelock */
    private final ReentrantReadWriteLock.WriteLock writerLock;
    public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
    public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }
```

> 类中维护了两个属性分别为ReadLock对象和WriteLock对象，分别对读写进行控制
>
> 故当资源类涉及到读写操作时，读操作可以使用读锁，写操作可以使用写锁
>
> 而ReadLock和WriteLock都实现了Lock接口，因此可以使用lock方法和unlock方法进行控制

![image-20210125120926736](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210125120926736.png)

结果：

![](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210125121010446.png)

> 每次进行写操作时都不会有其他的线程占用资源(资源独占)
>
> 当进行完写操作，再进行读操作时，多个读操作可以共享资源

## 7 BlockingQueue阻塞队列

一般情况下线程不阻塞肯定比较好，但是某种情况下不得不阻塞，必须要阻塞

例如当火锅店人特别多的时候，后面的人就必须要排队等待叫号，那么队伍就可以看做阻塞队列

![5bd6cc3d0001a58310000407](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/5bd6cc3d0001a58310000407.jpg)

> 当队列为空时，从队列中获取元素的操作将会被阻塞
>
> 当队列为满时，从队列中添加元素的操作将会被阻塞

`类似于生产者消费者模型`

用处：

> 在多线程领域：
>
> 		所谓阻塞，在某些情况下会`挂起`线程；一旦条件满足，被阻塞的线程又被重新唤醒(比如wait和notify方法)
>
> 为什么需要BlockingQueue：
>
> ​			不需要关心什么时候需要阻塞线程，什么时候唤醒线程，BlockingQueue会进行操作

### 7.1 继承树

![image-20210125203021009](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210125203021009.png)

> 在Collection接口下Queue接口和List、Set接口平级，而BlockingQueue接口继承于Queue接口
>
> 核心实现类：
>
> ​	ArrayBlockingQueue：由数组构成的`有界`阻塞队列
>
> ​	LinkedBlockingQueue：由链表组成的`有界`(但大小默认值为integer.MAX_VALUE)阻塞队列
>
> ​	SynchronousQueue：不存储元素的阻塞队列，也即单个元素的队列

### 7.2 API

#### 7.2.1 构造器

```java
/**
 * Creates an {@code ArrayBlockingQueue} with the given (fixed)
 * capacity and default access policy.
 *
 * @param capacity the capacity of this queue
 * @throws IllegalArgumentException if {@code capacity < 1}
 */
public ArrayBlockingQueue(int capacity) {
    this(capacity, false);
}
```

通过传入一个int参数capacity来指定阻塞队列的大小，从而说明了ArrayBlockingQueue是一个**有界的阻塞队列**

#### 7.2.2 add()

向阻塞队列中添加元素，当阻塞队列满时会抛出IllegalStateException，提示`Queue full`

演示：

```java
public class BolcokingQueueDemo {
    public static void main(String[] args) {
        BlockingQueue<String> queue = new ArrayBlockingQueue<>(3);

        System.out.println(queue.add("a"));
        System.out.println(queue.add("b"));
        System.out.println(queue.add("c"));

        System.out.println(queue.add("x"));
    }
}
```

结果：

```
true
true
true
Exception in thread "main" java.lang.IllegalStateException: Queue full
```

> 当没有超出队列长度时，则返回布尔值，表示插入是否成功

#### 7.2.3 remove()

从阻塞队列中移除元素，当阻塞队列为空时，再次移除会抛出NoSuchElementException

演示：

```java
public class BolcokingQueueDemo {
    public static void main(String[] args) {
        BlockingQueue<String> queue = new ArrayBlockingQueue<>(3);

        System.out.println(queue.add("a"));
        System.out.println(queue.add("b"));
        System.out.println(queue.add("c"));

        System.out.println(queue.remove());
        System.out.println(queue.remove());
        System.out.println(queue.remove());
        System.out.println(queue.remove());
    }
}
```

结果：

```
a
b
c
Exception in thread "main" java.util.NoSuchElementException
```

> 从出队列的顺序也可以看出队列的先进先出

#### 7.2.4 element()

不会从队列中取出元素，而只是检查队列的头元素，当队列为空时，会抛出NoSuchElementException

演示：

```java
public class BolcokingQueueDemo {
    public static void main(String[] args) {
        BlockingQueue<String> queue = new ArrayBlockingQueue<>(3);

        System.out.println(queue.add("a"));
        System.out.println(queue.add("b"));
        System.out.println(queue.add("c"));

        System.out.println(queue.element());
        System.out.println(queue.remove());
        System.out.println(queue.remove());
        System.out.println(queue.remove());
        System.out.println(queue.element());
    }
}
```

结果：

```
a
a
b
c
Exception in thread "main" java.util.NoSuchElementException
```

#### 7.2.5 offer()

向队列中插入元素，当队列为满时并不会抛出异常，而只是返回false

#### 7.2.6 poll()

从队列中取出元素，当队列为空时也不会抛出异常，而只是返回null

#### 7.2.7 peek()

不会从队列中取出元素，而只是检查队列的头元素；当队列为空时不会抛出异常，而只是返回null

#### 7.2.8 put()

向队列中插入元素，当队列为满时不会抛出异常，也不会返回null；而是处于阻塞状态

```java
public class BolcokingQueueDemo {
    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<String> queue = new ArrayBlockingQueue<>(3);

        queue.put("a");
        queue.put("a");
        queue.put("a");
        queue.put("a");
    }
}
```

> 程序一直在执行，说明处于阻塞状态

#### 7.2.9 take()

从队列中取出元素，当队列为空时不会抛出异常，也不会返回null；而是处于阻塞状态

#### 7.2.10 offer(e,time,unit)

向队列中插入元素，当队列为满时不会抛出异常，也不会返回null，而是处于阻塞状态；然而这种阻塞状态具有时间限制，当超时时程序就会结束，也不会抛出异常，没有任何返回

```java
public class BolcokingQueueDemo {
    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<String> queue = new ArrayBlockingQueue<>(3);

        queue.offer("a");
        queue.offer("a");
        queue.offer("a");
        queue.offer("a",3,TimeUnit.SECONDS);
    }
}
```

结果：

![image-20210125212852806](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210125212852806.png)

#### 7.2.11 poll(e,time,unit)

从队列中取出元素，当队列为空时不会抛出异常，也不会返回null，而是处于阻塞状态；然而这种阻塞状态具有时间限制，当超时时程序就会结束，也不会抛出异常，没有任何返回



## 8 volatile

volatile是Java虚拟机提供的轻量级同步机制，特点：

- 保证可见性
- `不保证原子性`
- 禁止指令重排

### 8.1 JMM(JAVA内存模型)

#### 8.1.1 可见性

JMM本身是一种抽象的概念并不真实存在，描述的是一组规则或规范，通过这组规范定义了程序中各个变量(包括实例字段，静态字段和构成数组对象的元素)的访问方式

JMM关于同步的规定：

- 线程解锁前，必须把共享变量的值刷新回主内存
- 线程加锁前，必须读取主内存最新值到自己的工作内存
- 加锁解锁是同一把锁

由于JVM运行程序的实体是线程，而每个线程创建时JVM都会为器创建一个工作内存(栈空间)，工作内存是每个线程的私有数据区域，而Java内存模型规定所有变量都存储在`主内存`，主内存是共享内存区域，所有的线程都可以访问，`但线程对变量的操作(读写)必须在工作内存中进行，首先要将变量从主内存拷贝到自己的工作内存空间，然后对变量进行操作后再将变量写回主内存`，不能直接操作主内存中的变量，各个线程的工作内存中主要存储着主内存的`变量拷贝副本`，因此不同的线程之间无法访问对方的工作内存，线程间的通信必须通过主内存来完成

![1673435-20190822095001999-1852846254](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/1673435-20190822095001999-1852846254.png)

举个例子，主内存中有一个student对象，目前有三个线程操作该对象：

![image-20210126143828104](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210126143828104.png)

> 主内存中存放的就是创建的student对象，而三个线程各自的工作内存中存放的是student对象的副本，当t1对student对象进行操作时，`实际上修改的值t2和t3并不知道`，只有当t1将新的student对象写回到主内存时，主内存就会通知其他线程变更为最新的student对象
>
> 主内存更新之后其他线程可以第一时间获取到最新的共享变量，这就是可见性，对于线程可见

##### 8.1.1.1 可见性测试

加入共享变量中没有添加volatile关键字：

```java
public class VolatileDemo {
    public static void main(String[] args) {
        MyData myData = new MyData();

        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "\t come in");
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            myData.add60();
            System.out.println(Thread.currentThread().getName() + "\t change value:" + myData.num);
        },"A").start();

        while (myData.num == 0) {

        }

        System.out.println(Thread.currentThread().getName() + "mission over");
    }
}

class MyData {
    int num = 0;

    public void add60() {
        this.num = 60;
    }
}
```

> 首先MyData定义了共享变量和共享变量的修改操作
>
> main方法中新起一个线程对共享变量进行修改，main线程监控共享变量的变化，如果没有变化就一直在死循环中

结果：

![image-20210126150245866](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210126150245866.png)

> 从结果中可以看出main线程一直在循环过程中，共享变量的修改并没有对main线程进行通知

共享变量添加volatile关键字：

```java
public class VolatileDemo {
    public static void main(String[] args) {
        MyData myData = new MyData();

        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "\t come in");
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            myData.add60();
            System.out.println(Thread.currentThread().getName() + "\t change value:" + myData.num);
        },"A").start();

        while (myData.num == 0) {

        }

        System.out.println(Thread.currentThread().getName() + "mission over");
    }
}

class MyData {
    volatile int num = 0;

    public void add60() {
        this.num = 60;
    }
}
```

结果：

![image-20210126150620834](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210126150620834.png)

#### 8.1.2 volatile不保证原子性

##### 8.1.2.1 测试volatile不保证原子性

```java
public class VolatileDemo {
    public static void main(String[] args) {
        MyData myData = new MyData();

        for (int i = 0; i < 20; i++) {
            new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    myData.addPlus();
                }
            },String.valueOf(i)).start();
        }

        while (Thread.activeCount() > 2) {
            Thread.yield();
        }

        System.out.println(Thread.currentThread().getName() + "\t finally value : " + myData.num);
    }
}

class MyData {
    volatile int num = 0;

    public void add60() {
        this.num = 60;
    }

    public void addPlus() {
        this.num++;
    }
}
```

> Thread.activeCount()静态方法计算当前的活跃线程，我们已知现在有main线程和GC线程
>
> Thread.yield()静态方法使当前线程礼让其他线程，让其他线程先执行

结果：

![image-20210126152725585](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210126152725585.png)

> 如果保证原子性，那么每次操作时其他线程都不能干涉，最终结果一定是20000，可是结果始终都不是20000表示不保证原子性

#### 8.1.2.2 不保证原子性原理

为什么每次运行的结果都不是20000呢？

![image-20210126153504113](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210126153504113.png)

> 假设共享变量初始为0，3个线程分别拿到了变量副本，假设3个线程这时同时调用方法将自己的变量副本变为1；此时三个线程互相之间不知道，当t1线程准备将自己的副本写回主内存时突然被挂起了，这时t2线程将自己的变量副本写回到主内存，主内存通知其他线程修改自己的变量副本，而这时t1线程的变量副本已经变为1了
>
> 从而造成计算数据的丢失

## 9 ThreadPool线程池

线程池的工作就是控制运行的线程数量，**处理过程中将任务放入队列**，然后在线程创建后启动这些任务，`如果线程数量超过了最大数量，超出数量的线程排队等候`，等其他线程执行完毕后，再从队列中取出任务来执行

特点：

- 线程复用；重复利用已创建的线程降低线程创建和销毁造成的消耗
- 控制最大并发数量
- 提高响应速度；当任务达到时，不需要等待创建线程就可以立即执行
- 提高线程可管理性；线程是稀缺资源，如果无限制的创建，不仅会消耗资源，还会降低系统的稳定性；线程池可以对线程进行统一的分配、调优和监控

### 9.1 继承树

![image-20210125220909109](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210125220909109.png)

#### 9.1.1 Executor

```
An object that executes submitted Runnable tasks. This interface provides a way of decoupling task submission from the mechanics of how each task will be run, including details of thread use, scheduling, etc. An Executor is normally used instead of explicitly creating threads. For example, rather than invoking new Thread(new(RunnableTask())).start() for each of a set of tasks, you might use:
 Executor executor = anExecutor;
 executor.execute(new RunnableTask1());
 executor.execute(new RunnableTask2());
 ...
```

从官网的解释可以看出，Executor是用来执行提交的Runnable任务，也就是线程池的顶级接口

#### 9.1.2 ExecutorService

![image-20210125221441366](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210125221441366.png)

实际使用的线程池接口(相对于Executor接口提供了更多的方法)

#### 9.1.3 ThreadPoolExecutor

![image-20210125221705249](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210125221705249.png)

创建线程池对象使用的实现类

#### 9.1.4 Executors

![image-20210125221808298](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210125221808298.png)

Executor的工具类(工厂类)，用来生成ThreadPoolExecutor对象

### 9.2 创建线程池

#### 9.2.1 Executors.newFixedThreadPool

创建指定线程数量的线程池，池中的线程数量不会发生改变；`适用于执行长期任务`

测试：

```java
public class ExectorDemo {
    public static void main(String[] args) {
        ExecutorService pool = Executors.newFixedThreadPool(5);

        try {
            for (int i = 0; i < 10; i++) {
                pool.execute(() -> {
                    System.out.println(Thread.currentThread().getName() + "\t 办理业务");
                });
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            pool.shutdown();
        }
    }
}
```

> 线程池调用execute方法，里面传入Runnable接口实例作为执行任务，线程池取出线程来执行对应的任务
>
> 线程池适用完毕需要调用shutdown方法关闭线程池资源
>
> Executors.newFixedThreadPool传入int参数来指定线程池中线程数量

#### 9.2.2 Executors.newSingleThreadExecutor()

创建单例线程池，也就是线程池中只有一个线程

```java
public class ExectorDemo {
    public static void main(String[] args) {
        ExecutorService pool = Executors.newSingleThreadExecutor();

        try {
            for (int i = 0; i < 10; i++) {
                pool.execute(() -> {
                    System.out.println(Thread.currentThread().getName() + "\t 办理业务");
                });
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            pool.shutdown();
        }
    }
}
```

#### 9.2.3 Executors.newSingleThreadExecutor()

创建可扩容的线程池，当有多个任务提交任务进入队列时，就会创建多个线程；当任务数量较少时，当前线程数量完全可以满足，就不进行扩容

```java
public class ExectorDemo {
    public static void main(String[] args) {
        ExecutorService pool = Executors.newCachedThreadPool();

        try {
            for (int i = 0; i < 10; i++) {
                pool.execute(() -> {
                    System.out.println(Thread.currentThread().getName() + "\t 办理业务");
                });
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            pool.shutdown();
        }
    }
}
```

### 9.3 ThreadPoolExecutor底层原理

首先我们先看下通过工具类创建线程池的三种方式是如何进行创建的？

![image-20210125223446688](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210125223446688.png)

> 实际上三种方式都是通过创建ThreadPoolExecutor对象且已经指定了一些参数来创建线程池

#### 9.3.1 7大参数

![image-20210125224008896](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210125224008896.png)

- corePoolSize：线程池中常驻线程核心数
- maximumPoolSize：线程池中能够容纳同时执行的最大线程数，必须大于1；`当阻塞队列满的时候，就会创建非核心线程`
- keepAliveTime：多余的空闲线程的存活时间，当前线程池中线程数量超过核心线程数时(而非核心线程又没有使用)，当空闲时间达到keepAlive时，多余线程就会被销毁直到只剩下核心线程数为止
- unit：keepAliveTime的单位
- workQueue：任务队列，被提交但是未执行的任务阻塞队列
- threadFactory：生成线程池中工作线程的线程工厂，用于创建线程；`一般使用默认即可`
- handler：拒绝策略，当阻塞队列满了，且工作线程已经大于等于线程池的最大线程数时如何拒绝请求执行的runnable的策略

#### 9.3.2 线程池处理流程

![cec330143ed7455eb64a6ed1c40ab075](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/cec330143ed7455eb64a6ed1c40ab075.jpg)

![u=380500544,3378234251&fm=26&gp=0](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/u=380500544,3378234251&fm=26&gp=0.png)

以银行作为例子进行解析：

> 1、银行当天的当值窗口也就是核心线程数，当有人办理业务(也就是提交任务)时就进入当值窗口
>
> 2、当当值窗口满了的时候，就开始排队等候(也就是阻塞队列)
>
> 3、当队列也满了的时候，没办法就只能把没有当值的窗口(最大线程数)也开了，再把队列中的人在非核心窗口办理，而之后到来的人进入队列
>
> 4、当队列再次满了的时候，就开始执行拒绝策略
>
> 5、办理的人越来越少，当非核心线程处于空闲状态，且已经超过了最大存活时间，就开始慢慢的关闭非核心线程

#### 9.3.3 不能通过Executors创建线程池

阿里巴巴Java开发手册：

```
3. 【强制】线程资源必须通过线程池提供，不允许在应用中自行显式创建线程
    说明：使用线程池的好处是减少在创建和销毁线程上所花的时间以及系统资源的开销，解决资
    源不足的问题。如果不使用线程池，有可能造成系统创建大量同类线程而导致消耗完内存或者
    “过度切换”的问题
4. 【强制】线程池不允许使用Executors去创建，而是通过ThreadPoolExecutor的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险
	说明： Executors 返回的线程池对象的弊端如下：
	1） FixedThreadPool 和 SingleThreadPool :
	允许的请求队列长度为 Integer.MAX_VALUE，可能会堆积大量的请求，从而导致OOM 
	2） CachedThreadPool和ScheduledThreadPool :
	允许的创建线程数量为 Integer.MAX_VALUE，可能会创建大量的线程，从而导致OOM 
```

