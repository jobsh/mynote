# Synchronized关键字深度解析

- - -

# 一、synchronized基础

> synchronized关键字在需要原子性、可见性和有序性这三种特性的时候都可以作为其中一种解决方案，看起来是“万能”的。的确，大部分并发控制操作都能使用synchronized来完成。在多线程并发编程中Synchronized一直是元老级角色，很多人都会称呼它为重量级锁，但是随着Java SE1.6对Synchronized进行了各种优化之后，有些情况下它并不那么重了，本文详细介绍了Java SE1.6中为了减少获得锁和释放锁带来的性能消耗而引入的偏向锁和轻量级锁，以及锁的存储结构和升级过程。

## 1.1synchronized的使用

| 修饰目标       | 锁           |                            |
| -------------- | ------------ | -------------------------- |
| 方法           | 实例方法     | 当前实例对象(即方法调用者) |
| 静态方法       | 类对象       |                            |
| 代码块         | this         | 当前实例对象(即方法调用者) |
| class对象      | 类对象       |                            |
| 任意Object对象 | 任意示例对象 |                            |

#### 1.1示例

```java
public class Synchronized {
    //synchronized关键字可放于方法返回值前任意位置，本示例应当注意到sleep()不会释放对监视器的锁定
    //实例方法
    public synchronized void instanceMethod() {
        for (int i = 0; i < 5; i++) {
            System.out.println("instanceMethod");
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    //静态方法
    public synchronized static void staticMethod() {
        for (int i = 0; i < 5; i++) {
            System.out.println("staticMethod");
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public void thisMethod() {
        //this对象
        synchronized (this) {
            for (int i = 0; i < 5; i++) {
                System.out.println("thisMethod");
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public void classMethod() {
        //class对象
        synchronized (Synchronized.class) {
            for (int i = 0; i < 5; i++) {
                System.out.println("classMethod");
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public void anyObject() {
        //任意对象
        synchronized ("anything") {
            for (int i = 0; i < 5; i++) {
                System.out.println("anyObject");
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
1234567891011121314151617181920212223242526272829303132333435363738394041424344454647484950515253545556575859606162636465666768
```

#### 1.2验证

##### 1.2.1 普通方法和代码块中使用this是同一个监视器（锁），即某个具体调用该代码的对象

```java
   public static void main(String[] args) {
        Synchronized syn = new Synchronized();
        for (int i = 0; i < 10; i++) {
            new Thread() {
                @Override
                public void run() {
                    syn.thisMethod();
                }
            }.start();
            new Thread() {
                @Override
                public void run() {
                    syn.instanceMethod();
                }
            }.start();
        }
    }
1234567891011121314151617
```

> 我们会发现输出结果总是以5个为最小单位交替出现，证明sychronized(this)和在实例方法上使用synchronized使用的是同一监视器。如果去掉任一方法上的synchronized或者全部去掉，则会出现instanceMethod和thisMethod无规律的交替输出。

##### 1.2.2 静态方法和代码块中使用该类的class对象是同一个监视器，任何该类的对象调用该段代码时都是在争夺同一个监视器的锁定

```java
    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            Synchronized syn = new Synchronized();
            new Thread() {
                @Override
                public void run() {
                    syn.staticMethod();
                }
            }.start();
            new Thread() {
                @Override
                public void run() {
                    syn.classMethod();
                }
            }.start();

        }
    }
123456789101112131415161718
```

> 输出以5个为最小单位交替出现，证明两段代码是同一把锁，如果去掉任一synchronnized则会无规律交替出现。

### 1.2、synchronized的特点

1. 可重入性
2. 当代码段执行结束或出现异常后会自动释放对监视器的锁定
3. 是公平锁，在等待获取锁的过程中不可被中断
4. synchronized的内存语义（详见[面试打怪升升级-被问烂的volatile关键字，这次我要搞懂它（深入到操作系统层面理解，超多图片示意图）](https://blog.csdn.net/weixin_42762133/article/details/105264806)）
5. 互斥性，被synchronized修饰的方法同时只能由一个线程执行

# 二、synchronized进阶

## 2.1对象头

如果对象是数组类型，则虚拟机用3个字宽（Word）存储对象头，如果对象是非数组类型，则用2字宽存储对象头。在32位虚拟机中，1字宽等于4字节，即32bit。（64位中1字宽=8字节=64bit）如表所示

| 长度     | 内容                   | 说明                             |
| -------- | ---------------------- | -------------------------------- |
| 32/64bit | Mark Word              | 存储对象的hashCode或锁信息等     |
| 32/64bit | Class Metadata Address | 存储到对象类型数据的指针         |
| 32/32bit | Array length           | 数组的长度（如果当前对象是数组） |

在`oop.hpp`中这样定义
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200406195734593.png)
HotSpot通过markOop类型实现Mark Word，具体实现位于markOop.hpp文件中。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200406200030683.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)
Java对象头的Mark Word里默认存储对象的HashCode、分代年龄和锁标志位。32位JVM的Mark Word的默认存储结构如表所示

| 锁状态   | 25bit          | 4bit         | 1bit是否偏向锁 | 2bit锁标志位 |
| -------- | -------------- | ------------ | -------------- | ------------ |
| 无锁状态 | 对象的hashCode | 对象分代年龄 | 0              | 01           |

Mark Word可能变化为存储以下4种数据，如表所示
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020040620033037.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)
age： 保存对象的分代年龄
biased_lock： 偏向锁标识位
lock： 锁状态标识位
JavaThread*： 保存持有偏向锁的线程ID
ptr: monitor的指针
epoch： 保存偏向时间戳

| 锁状态   | 25bit                        | 4bit         | 1bit         | 2bit |      |
| -------- | ---------------------------- | ------------ | ------------ | ---- | ---- |
| 23bit    | 2bit                         | 是否是偏向锁 | 锁标志位     |      |      |
| 轻量级锁 | 指向栈中所记录的指针         | 00           |              |      |      |
| 重量级锁 | 指向互斥量（重量级锁）的指针 | 10           |              |      |      |
| GC标志   | 空                           | 11           |              |      |      |
| 偏向锁   | 线程ID                       | Epoch        | 对象分代年龄 | 1    | 01   |

## 2.2synchronized实现原理

我们写个demo看下，使用javap命令，查看JVM底层是怎么实现synchronized

```java
public class TestSynMethod1 {
    synchronized void hello() {

    }

    public static void main(String[] args) {
        String anything = "anything";
        synchronized (anything) {
            System.out.println("hello word");
        }
    }
}

12345678910111213
```

同步块的jvm实现，可以看到它通过`monitorenter`和`monitorexit`实现锁的获取和释放。通过图片中的注解可以很好的解释synchronized的特性2，当代码段执行结束或出现异常后会自动释放对监视器的锁定。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200406194602860.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)
注意，如果synchronized在方法上，那就没有上面两个指令，取而代之的是有一个ACC_SYNCHRONIZED修饰，表示方法加锁了。然后可以在常量池中获取到锁对象，实际实现原理和同步块一致，后面也会验证这一点
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200406194150630.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)

## 2.3锁升级

首先讲一下==《java并发编程的艺术》==中对这一现象的描述，非常简洁生动，但是在复习的时候发现随着理解的深入多了许多疑问，最后通过阅读jvm源码和大量的资料终于搞清了我的疑问，接下来和大家分享一下。

### 2.3.1《java并发编程的艺术》的描述(引用)

------

Java SE 1.6为了减少获得锁和释放锁带来的性能消耗，引入了“偏向锁”和“轻量级锁”，在Java SE 1.6中，锁一共有4种状态，级别从低到高依次是：无锁状态、偏向锁状态、轻量级锁状态和重量级锁状态，这几个状态会随着竞争情况逐渐升级。锁可以升级但不能降级（和读写锁的升降级不是一回事），意味着偏向锁升级成轻量级锁后不能降级成偏向锁。这种锁升级却不能降级的策略，目的是为了提高获得锁和释放锁的效率，下文会详细分析。

#### 1.偏向锁

HotSpot的作者经过研究发现，大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁。当一个线程访问同步块并 获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID，以后该线程在进入和退出 同步块时不需要进行CAS操作来加锁和解锁，只需简单地测试一下对象头的Mark Word里是否 存储着指向当前线程的偏向锁。如果测试成功，表示线程已经获得了锁。如果测试失败，则需 要再测试一下Mark Word中偏向锁的标识是否设置成1（表示当前是偏向锁）：如果没有设置，则使用CAS竞争锁；如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程。

##### （1）偏向锁的撤销

偏向锁使用了一种等到竞争出现才释放锁的机制，所以当其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁。偏向锁的撤销，需要等待全局安全点（在这个时间点上没有正 在执行的字节码）。它会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着， 如果线程不处于活动状态，则将对象头设置成无锁状态；如果线程仍然活着，拥有偏向锁的栈 会被执行，遍历偏向对象的锁记录，栈中的锁记录和对象头的Mark Word要么重新偏向于其他 线程，要么恢复到无锁或者标记对象不适合作为偏向锁，最后唤醒暂停的线程。图2-1中的线 程1演示了偏向锁初始化的流程，线程2演示了偏向锁撤销的流程。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200406214055464.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)

##### （2）关闭偏向锁

偏向锁在Java 6和Java 7里是默认启用的，但是它在应用程序启动几秒钟之后才激活，如有必要可以使用JVM参数来关闭延迟：-`XX:BiasedLockingStartupDelay=0`。如果你确定应用程 序里所有的锁通常情况下处于竞争状态，可以通过JVM参数关闭偏向锁：-`XX:UseBiasedLocking=false`，那么程序默认会进入轻量级锁状态。

#### 2.轻量级锁

##### （1）轻量级锁加锁

线程在执行同步块之前，JVM会先在当前线程的栈桢中创建用于存储锁记录的空间，并将对象头中的Mark Word复制到锁记录中，官方称为Displaced Mark Word。然后线程尝试使用 CAS将对象头中的Mark Word替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失 败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁。

##### （2）轻量级锁解锁

轻量级解锁时，会使用原子的CAS操作将Displaced Mark Word替换回到对象头，如果成功，则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁。图2-2是 两个线程同时争夺锁，导致锁膨胀的流程图。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200406214315645.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)
因为自旋会消耗CPU，为了避免无用的自旋（比如获得锁的线程被阻塞住了），一旦锁升级成重量级锁，就不会再恢复到轻量级锁状态。当锁处于这个状态下，其他线程试图获取锁时， 都会被阻塞住，当持有锁的线程释放锁之后会唤醒这些线程，被唤醒的线程就会进行新一轮 的夺锁之争。

------

到此，我们可以看到一个锁升级的轮廓了，但是看完之后有一些细节却让我更加迷惑，最后经过思考后，我发现作者给出的图片和描述适用的是当两个线程拥有同样锁等级同时竞争时的状况。 下面是我关于锁升级的一些思考

### 2.3.2一些补充和验证

#### 1.小试牛刀

我们首先验证一下java6以后默认开启偏向锁，它在应用程序启动几秒钟之后才激活。
**使用JOL工具类，打印对象头**
添加maven依赖

```xml
<dependency>
  <groupId>org.openjdk.jol</groupId>
  <artifactId>jol-core</artifactId>
  <version>0.8</version>
</dependency>
12345
```

创建`O`对象

```java
public class O {
    int a = 1;
}
123
```

创建`TestInitial`测试，设置启动参数`-XX:+PrintFlagsFinal`

```java
public class TestInitial {
    public static void main(String[] args) {
        O object = new O();
        //打印对象头
        System.out.println(ClassLayout.parseInstance(object).toPrintable());
    }
}
1234567
```

结果如下，重点关注红框内的内容
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200406221708464.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200406221926998.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200406221327885.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)
64bit环境下红框内位置对应的分布如下：![在这里插入图片描述](https://img-blog.csdnimg.cn/20200406220936178.png)
我们可以看到此时对象头处于轻量级锁的无锁状态（如下图示意,重点关注后三位），但是我们的偏向锁明明是开启的，这是因为由4s中的延时开启，这一设计的目的是因为程序在启动初期需要初始化大量类，此时会发生大量锁竞争，如果开启偏向锁，在冲突时锁撤销要耗费大量时间。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020040622275269.png)
**修改`TestInitial`程序,第一行添加延时5s**

```java
public class TestInitial {
    public static void main(String[] args) throws InterruptedException {
        TimeUnit.SECONDS.sleep(5);
        O object = new O();
        System.out.println(ClassLayout.parseInstance(object).toPrintable());
    }
}
1234567
```

**测试结果如下**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200406222615254.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)
可以发现过了偏向锁延时启动时间后，我们再创建对象，对象头锁状态变成了偏向锁
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200407101157163.png)

#### 2. 锁的释放获取

解释器执行monitorenter时会进入到`InterpreterRuntime.cpp`的`InterpreterRuntime::monitorenter`函数，具体实现如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200406201702407.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)
`synchronizer.cpp`文件的`ObjectSynchronizer::fast_enter`函数：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200406204908382.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)
`BiasedLocking::revoke_and_rebias`函数过长，下面就简单分析下（着重分析一个线程先获得锁，下面会通过实验来验证结论）
**1. 当线程访问同步块时首先检查对象头中是否存储了当前线程（和java中的ThreadId不一样），如果有则直接执行同步代码块。**
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020040621043079.png)
即此时`JavaThread*`指向当前线程
**2. 如果没有，查看对象头是否是允许偏向锁且指向线程id为空，**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200406210334473.png)

**测试代码**

```java
		public class TestBiasedLock {
		    public static void main(String[] args) throws InterruptedException {
		        TimeUnit.SECONDS.sleep(5);
		        O object = new O();
		
		        synchronized (object) {
		            System.out.println("1\n" + ClassLayout.parseInstance(object).toPrintable());
		        }
		        TimeUnit.SECONDS.sleep(1);
		        System.out.println("2\n" + ClassLayout.parseInstance(object).toPrintable());
		    }
		}		
123456789101112
```

**测试结果**
![在这里插入图片描述](https://img-blog.csdnimg.cn/202004062240036.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)
结合初始化的测试，我们可以得知偏向锁的获取方式。CAS设置当前对象头指向自己，如果成功，则获得偏向锁（t1获得了偏向锁）开始执行代码。并且知道了拥有偏向锁的线程在执行完成后，偏向锁`JavaTherad*`依然指向第一次的偏向。
**3.t2尝试获取偏向锁，此时对象头指向的不是自己(指向t1，而不是t2)，开始撤销偏向锁**， **升级为轻量级锁**。偏向锁的撤销，需要等待全局安全点，然后检查持有偏向锁的线程(t1)是否活着。

> ​      (1). 如果存活：**让该线程(t1)获取轻量级锁，将对象头中的Mark Word替换为指向锁记录的指针，然后唤醒被暂停的线程。** 也就是说将当前锁升级为轻量级锁，并且让之前持有偏向锁的线程(t1)继续持有轻量级锁。
> ​     (2). 如果已经死亡：**将对象头设置成无锁状态**

**之前尝试获取偏向锁失败引发锁升级的线程(t2)尝试获取轻量级锁**，在当前线程的栈桢中然后创建用于存储锁记录的空间，并将对象头中的Mark Word复制到锁记录中，官方称为Displaced Mark Word。然后线程尝试使用 CAS将对象头中的Mark Word替换为指向锁记录的指针，如果失败，开始自旋（即重复获取一定次数），在自旋过程中过CAS设置成功，则成功获取到锁对象。**java中采用的是自适应自旋锁，即如果第一次自旋获取锁成功了，那么在下次自旋时，自旋次数会适当增加。** 采用自旋的原因是尽量减少内核用户态的切换。也就是说t2尝试获取偏向锁失败，导致偏向锁的撤销，撤销后，线程（t2）继续尝试获取轻量级锁。

```java
public class TestLightweightLock3 {
    public static void main(String[] args) throws InterruptedException {
        TimeUnit.SECONDS.sleep(5);
        O object = new O();

        Thread thread1 = new Thread() {
            @Override
            public void run() {
                synchronized (object) {
                    System.out.println("thread1 获取偏向锁成功");
                    System.out.println(ClassLayout.parseInstance(object).toPrintable());
                }
            }
        };

        Thread thread2 = new Thread() {
            @Override
            public void run() {
                synchronized (object) {
                    System.out.println("thread2 获取偏向锁失败，升级为轻量级锁，获取轻量级锁成功");
                    System.out.println(ClassLayout.parseInstance(object).toPrintable());
                }
            }
        };
        thread1.start();

        //让thread1死亡
        thread1.join();
        thread2.start();

        //thread2死亡
        thread2.join();
        System.out.println("thread2执行结束，释放轻量级锁");
        System.out.println(ClassLayout.parseInstance(object).toPrintable());
    }
}
123456789101112131415161718192021222324252627282930313233343536
```

上述测试的是，thread1获取了偏向锁，`JavaThread*`指向thread1。thread2在thread1执行完毕后尝试获取偏向锁，发现该偏向锁指向thread1，因此开始撤销偏向锁，然后尝试获取轻量级锁。
**测试结果**
t1先执行获取偏向锁成功，开始执行。
t2获取偏向锁失败，升级为轻量级锁
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200406232433268.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)
t2获取轻量级锁成功，执行同步代码块
![在这里插入图片描述](https://img-blog.csdnimg.cn/202004062324558.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)
**4. 如果t2在自旋过程中成功获取了锁，那么t2开始执行。此时对象头格式为：**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200407103509764.png)
**在t2执行结束后，释放轻量级锁，锁状态为**
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020040710364173.png)
**5. 如果t2在自旋过程中未能获得锁，那么此时膨胀为重量级锁，将当前轻量级锁标志位变为(10)重量级，创建objectMonitor对象，让t1持有重量级锁。然后当前线程开始阻塞。**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200407105710917.png)

```java
public class TestMonitor {
    public static void main(String[] args) throws InterruptedException {
        TimeUnit.SECONDS.sleep(5);
        O object = new O();
        Thread thread1 = new Thread() {
            @Override
            public void run() {
                synchronized (object) {
                    System.out.println("thread1 获得偏向锁");
                    System.out.println(ClassLayout.parseInstance(object).toPrintable());
                    try {
                        //让线程晚点儿死亡，造成锁的竞争
                        TimeUnit.SECONDS.sleep(6);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println("thread2 获取锁失败导致锁升级,此时thread1还在执行");
                    System.out.println(ClassLayout.parseInstance(object).toPrintable());
                }
            }
        };
        Thread thread2 = new Thread() {
            @Override
            public void run() {
                synchronized (object) {
                    System.out.println("thread2 获取偏向锁失败，最终升级为重量级锁，等待thread1执行完毕，获取重量锁成功");
                    System.out.println(ClassLayout.parseInstance(object).toPrintable());
                    try {
                        TimeUnit.SECONDS.sleep(3);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        };
        thread1.start();
        //对象头打印需要时间,先让thread1获取偏向锁
        TimeUnit.SECONDS.sleep(5);
        thread2.start();
    }
}

123456789101112131415161718192021222324252627282930313233343536373839404142
```

**测试结果**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200407120910867.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020040712110573.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)

> 总结：至此锁升级已经介绍完毕，接下来在介绍一下重量级锁的实现机制ObjectMonitor即可。再次梳理整个过程（主要是一个线程t1已经获得锁的情况下，另一个线程t2去尝试获取锁）：
> \1. t2尝试获取偏向锁，发现偏向锁指向t1，获取失败
> \2. 失败后开始偏向锁撤销，如果t1还存活将轻量级锁指向它，它继续运行；t2尝试获取锁，开始自旋等待t1释放轻量级锁。
> \3. 如果在自旋过程中t1释放了锁，那么t2获取轻量级锁成功。
> \4. 如果在自旋结束后，t2未能获取轻量锁，那么锁升级为重量级锁，使t1持有objectmonitor对象，将t2加入EntryList，t2开始阻塞，等待t1释放监视器

### 2.3.3jvm的monitor实现(重量级锁)

jvm中Hotspot关于synchronized锁的实现是靠ObjectMonitor（对象监视器）实现的，当多个线程同时请求一个对象监视器（请求同一个锁）时，对象监视器将设置几个状态以用于区分调用线程：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200407141206114.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)

| 属性               | 意义                                 |
| ------------------ | ------------------------------------ |
| _header            | MarkOop对象头                        |
| _waiters           | 等待线程数                           |
| _recursions        | 重入次数                             |
| _owner             | 指向获得ObjectMonitor的线程          |
| _WaitSet           | 调用了java中的wait()方法会被放入其中 |
| _cxq \| _EntryList | 多个线程尝试获取锁时                 |

#### 1.获取锁

**线程锁的获取就是改变_owner指针，让他指向自己。**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200407181659969.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)

> Contention List：首先将锁定线程的所有请求放入竞争队列
> OnDeck：任何时候只有一个线程是最具竞争力的锁，该线程称为OnDeck（由系统调度策略决定）

锁的获取在jvm中代码实现如下，`ObjectMonitor::enter`
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200407184154649.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)

1. 通过CAS尝试把monitor的_owner字段设置为当前线程；
2. 如果设置之前的_owner指向当前线程，说明当前线程再次进入monitor，即重入锁，执行_recursions ++ ，记录重入的次数；
3. 查看当前线程得得锁记录中得Displaced Mark Word，即是否是该锁的轻量级锁持有者，如果是则是第一次加重量级锁，设置_recursions为1，_owner为当前线程，该线程成功获得锁并返回；
4. 如果获取锁失败，则等待锁的释放；

而锁的并发竞争状态维护就是依靠三个队列来实现的，_WaitSet、_cxq | _EntryList|。这三个队列都是由以下的数据结构实现得，所有的线程都会被包装成下面的结构，可以看到其实就是双向链表实现。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200407183931636.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)
monitor竞争失败的线程，通过自旋执行ObjectMonitor::EnterI方法等待锁的释放，EnterI方法的部分逻辑实现如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200407185925216.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)
1、当前线程被封装成ObjectWaiter对象node，状态设置成ObjectWaiter::TS_CXQ；
2、自旋CAS将当前节点使用头插法加入cxq队列
3、node节点push到_cxq列表如果失败了，再尝试获取一次锁（因为此时同时线程加入，可以减少竞争。），如果还是没有获取到锁，则通过park将当前线程挂起，等待被唤醒，实现如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200407190627730.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)
当被系统唤醒时，继续从挂起的地方开始执行下一次循环也就是继续自旋尝试获取锁。如果经过一定时间获取失败继续挂起。

#### 2.释放锁

当某个持有锁的线程执行完同步代码块时，会进行锁的释放。在HotSpot中，通过改变ObjectMonitor的值来实现，并通知被阻塞的线程，具体实现位于ObjectMonitor::exit方法中。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200407192133590.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)
1、初始化ObjectMonitor的属性值，如果是重入锁递归次数减一，等待下次调用此方法，直到为0，该锁被释放完毕。
2、根据不同的策略（由QMode指定），从cxq或EntryList中获取头节点，通过ObjectMonitor::ExitEpilog方法唤醒该节点封装的线程，唤醒操作最终由unpark完成。

#### wait()/notify()/notifyAll()

这两个方法其实是调用内核的方法实现的，他们的逻辑是将调用wait()的线程**加入_WaitSet**中，然后**等待notify唤醒他们**，**重新加入到锁的竞争之中**，notify和notifyAll不同在于**前者只唤醒一个线程后者唤醒所有队列中的线程**。值得注意的是notify并不会立即释放锁，而是等到同步代码执行完毕。

#### 一些有意思的事情

**1. hashCode()、wait()方法会使锁直接升级为重量级锁（在看jvm源码注释时看到的），下面测试一下**
调用wait方法

```java
public class TestWait {

    public static void main(String[] args) throws InterruptedException {
        TimeUnit.SECONDS.sleep(5);
        O object = new O();

        Thread thread1 = new Thread() {
            @Override
            public void run() {
                synchronized (object) {
                    System.out.println("thread1获取锁成功，开始执行，因为thread1调用了wait()方法，直接升级为重量级锁");
                    System.out.println("2\n" + ClassLayout.parseInstance(object).toPrintable());
                    object.notify();
                }
            }
        };

        Thread thread2 = new Thread() {
            @Override
            public void run() {
                synchronized (object) {
                    System.out.println("thread2 获取偏向锁成功开始执行");
                    System.out.println("1\n" + ClassLayout.parseInstance(object).toPrintable());
                    try {
                        object.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        };
        thread2.start();

        //让thread1执行完同步代码块中方法。
        TimeUnit.SECONDS.sleep(3);
        thread1.start();
    }
}
1234567891011121314151617181920212223242526272829303132333435363738
```

**测试结果**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200407194951777.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)
调用hashCode()

```java
public class TestLightweightLock {
    public static void main(String[] args) throws InterruptedException {
        TimeUnit.SECONDS.sleep(5);
        O object = new O();
        synchronized (object) {
            System.out.println("thread1 获取偏向锁成功，开始执行代码");
            System.out.println(ClassLayout.parseInstance(object).toPrintable());
            object.hashCode();
            try {
                //等待对象头信息改变
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("hashCode() 调用后");
            System.out.println(ClassLayout.parseInstance(object).toPrintable());
        }
    }
}

1234567891011121314151617181920
```

**测试结果**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200407195514693.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)
\3. 锁也可以降级，在安全点判断是否有线程尝试获取此锁，如果没有进行锁降级（重量级锁降级为轻量级锁，和之前在书中看到的锁只能升级不同，可能理解的意思不一样）。
测试代码如下，顺便测试了一下重量级锁升级

```java
public class TestMonitor {
    public static void main(String[] args) throws InterruptedException {
        TimeUnit.SECONDS.sleep(5);
        O object = new O();
        Thread thread1 = new Thread() {
            @Override
            public void run() {
                synchronized (object) {
                    System.out.println("thread1 获得偏向锁");
                    System.out.println(ClassLayout.parseInstance(object).toPrintable());
                    try {
                        //让线程晚点儿死亡，造成锁的竞争
                        TimeUnit.SECONDS.sleep(6);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println("thread2 获取锁失败导致锁升级,此时thread1还在执行");
                    System.out.println(ClassLayout.parseInstance(object).toPrintable());
                }
            }
        };
        Thread thread2 = new Thread() {
            @Override
            public void run() {
                synchronized (object) {
                    System.out.println("thread2 获取偏向锁失败，最终升级为重量级锁，等待thread1执行完毕，获取重量锁成功");
                    System.out.println(ClassLayout.parseInstance(object).toPrintable());
                    try {
                        TimeUnit.SECONDS.sleep(3);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        };
        thread1.start();
        //对象头打印需要时间,先让thread1获取偏向锁
        TimeUnit.SECONDS.sleep(5);
        //thread2去获取锁，因为t1一直在占用，导致最终升级为重量级锁
        thread2.start();
        
        //确保t1和t2执行结束
        thread1.join();
        thread2.join();
        TimeUnit.SECONDS.sleep(1);
       

        Thread t3 = new Thread(() -> {
            synchronized (object) {
                System.out.println("再次获取");
                System.out.println(ClassLayout.parseInstance(object).toPrintable());
            }
        });
        t3.start();
    }
}

123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657
```

**测试结果**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200407195836234.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200407200237622.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjc2MjEzMw==,size_16,color_FFFFFF,t_70)
t1和t2由于争抢导致锁升级为重量级锁，等待它们执行完毕，启动t3获取同一个锁发现又降级为轻量级锁。