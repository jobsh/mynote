# final与不变性

## 1. 什么是不变性（Immutable）

如果对象在被创建后，状态就不能被修改，那么它就是不可变的（不仅是对象引用不可变，包括里面成员变量也是不可变的）

例子：person对象，age和name都不能再变

```java
/**

 * 不可变的对象，演示其他类无法修改这个对象，public也不行

 */

public class Person {

    final int age = 18;

    final String name = "Alice";

    public static void main(String[] args) {

        //new Person().age = 60;

    }

}
```

**具有不变性的对象一定是线程安全的，我们不需要对其采取任何额外的安全措施，也能保证线程安全** 

## 2 final的作用

* 早期
  * 锁定 
  * 效率：早期的Java实现版本中，会将final方法转为内嵌调用现在
  * 类防止被继承、方法防止被重写、变量防止被修改 天生是线程安全的，而不需要额外的同步开销
* 现在（由于JVM的不断优化，上面提到性能原因使用final已经显得无所差别，更多的是使用他的不可变性）
  * 类防止被继承
  * 方法防止被重写
  * 变量防止被修改 
  * 天生是线程安全的，而不需要额外的同步开销

## 3. final的三种用法：修饰变量、方法、类

### final修饰变量

> 含义：被final修饰的变量，意味着值不能被修改。如果变量是对象，那么对象的**引用不能变**，但是对象自身的内容依然可以变化final修饰：3种变量

> final instance variable（类中的final属性） final static variable（类中的static final属性） final local variable（方法中的final变量）

### final修饰变量的赋值时机

> 属性被声明为final后，该变量则只能被赋值一次。且一旦被赋值，final的变量就不能再被改变，无论如何也不会变

#### final **instance** variable（类中的final属性）—— 三种赋值时机

> 第一种是在声明变量的等号右边直接赋值 第二种就是构造函数中赋值 第三就是在类的初始代码块中赋值（不常用） 如果不采用第一种赋值方法，那么就必须在第2、3种挑一个来赋值，而不能不赋值，这是final语法所规定的

```java
/**

 * 演示final变量

 */

public class FinalVariableDemo {

    //第一种是在声明变量的等号右边直接赋值

    private final int a;//a = 1;

    //第二种就是构造函数中赋值

    //public FinalVariableDemo(int a) {

    //    this.a = a;

    //}

    //第三就是在类的初始代码块中赋值（不常用）

    //{

    //    a = 1;

    //}

}
```

#### final **static** variable（类中的static final属性）

> 两个赋值时机：除了在声明变量的等号右边直接赋值外，static final变量述可以用static初始代码块赋值，但是不能用普通的初始代码块赋值

```java
public class FinalVariableDemo {

    private static final int a;//a = 1; 

    //static {

    //    a = 1;

    //}

}
```

#### final local variable（方法中的final变量）

> 和前面两种不同，由于这里的变量是在方法里的，所以没有构造函数，也不存在初始代码块 final local variable不规定赋值时机，只要求在使用前必须赋值，这和方法中的非final变量的要求也是一样的

```java
public class FinalVariableDemo {

    void testFinal() {

       final int b;//b = 7;

       //在使用前必须赋值

        int c =b;

    }

}
```

**为什么要规定赋值时机？**

> 我们来思考一下为什么语法要这继承这样？：如果初始化不赋值，后续赋值，就是从null变成你的赋值，这就违反final不变的原则了！

```java
public class FinalVariableDemo {

    private static final Object person = null;

    //static {

    //    person = new Object();

    //}

}
```

### final修饰方法 

* 构造方法不允许final修饰 

* 不可被重写，也就是不能被@Override，即便是子类有同样名字的方法，那也不是Override，这个和static方法是一个道理 

* 引申：static方法不能被重写

  static方法可以被继承，但是不能被重写，如果父子类静态方法名相同，则会隐藏父类方法。

  静态方法是编译时绑定的，方法重写是运行时绑定的。

```java
public class FinalMethodDemo {

    int a;

    //

    public final FinalMethodDemo(int a){

        this.a = a;

    }

}

public class FinalMethodDemo {

    public void drink() {

    }

    public final void eat() {

    }

    public static void sleep() {

        System.out.println("immutable.FinalMethodDemo.sleep");

    }

}

class SubClass extends FinalMethodDemo {

    @Override

    public void drink() {

        super.drink();

        eat();

    }

    //public final void eat() {

    //}

    //public void eat() {

    //}

 

    //可以写一个同名的方法：这一点和final方法不一样，这里并不是重新，static方法在编译的时候就已经和该类绑定了，也就是父类是父类的static方法，子类是子类自己的static方发。

    public static void sleep() {

        System.out.println("immutable.SubClass.sleep");

    }

    //public static void main(String[] args) {

    //    SubClass.sleep();

    //}

}
```

### final修饰类

> 不可被继承 例如典型的String类就是final的，我们从没见过哪个类是继承String类的

## 4 final的注意点

* final修饰对象的时候，只是对象的引用不可变，而对象本身的属性是可以变化的 
* final使用原则：良好的编程习惯（明确知道某一个变量/对象生成之后不会再发生变化，就把它声明为final）

## 5 不变性和final的关系

不变性并不意味着，简单地用final修饰就是不可变

对于基本数据类型，确实被final修饰后就具有不变性 但是对于对象类型，需要该对象保证自身被创建后，状态永远不会变才可以

### **如何利用final实现对象不可变？**

把所有属性都声明为final？ //这句话是不对的： 一个类的属性如果是是对象类型的，那么这个对象的属性是可以改变的

不可变对象的正确例子总结：满足以下条件时，对象才是不可变的

* 对象创建后，其状态就不能修改 

* 所有属性都是final修饰的 
* 对象创建过程中没有发生逸出

```java
public class Person {

    final int age = 18;

    final String name = "Alice";

    final TestFinal testFinal = new TestFinal();

    public static void main(String[] args) {

        final Person person = new Person();

        person.testFinal.test = "北京";

        System.out.println(person.testFinal.test);

        person.testFinal.test = "上海";

        System.out.println(person.testFinal.test);

    }

}

class TestFinal{

    String test;

}

/**

 * 一个属性是对象，但是整体不可变，其他类无法修改set里面的数据

 */

public class ImmutableDemo {

    private final Set<String> students = new HashSet<>();

    public ImmutableDemo() {

        students.add("蔡徐坤");

        students.add("乔碧萝");

        students.add("卢本伟");

    }

    public boolean isStudent(String name) {

        return students.contains(name);

    }

}
```

### 把变量写在线程内部—栈封闭

> 在方法里新建的局部变量，实际上是存储在每个线程**私有的栈空间**，而每个栈的栈空间是不能被其他线程所访问到的，所以不会有线程安全问题。这就是著名的**“栈封闭”**技术，是“线程封闭”技术的一种情况。

```java
package immutable;

/**
 * 描述：     演示栈封闭的两种情况，基本变量和对象 先演示线程争抢带来错误结果，然后把变量放到方法内，情况就变了
 */
public class StackConfinement implements Runnable {

    int index = 0;

    public void inThread() {
        // neverGoOut栈封闭，各个线程互不影响
        int neverGoOut = 0;
        // synchronized (this) {
            for (int i = 0; i < 10000; i++) {
                neverGoOut++;
            }
        // }

        System.out.println("栈内保护的数字是线程安全的：" + neverGoOut);
    }

    @Override
    public void run() {
        for (int i = 0; i < 10000; i++) {
            index++;
        }
        inThread();
    }

    public static void main(String[] args) throws InterruptedException {
        StackConfinement r1 = new StackConfinement();
        Thread thread1 = new Thread(r1);
        Thread thread2 = new Thread(r1);
        thread1.start();
        thread2.start();
        thread1.join();
        thread2.join();
        System.out.println(r1.index);
    }
}

```

打印结果：

![image-20200805085706480](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\image-20200805085706480.png)

## 6 面试题

**题一：**

```java
public class FinalStringDemo1 {

    public static void main(String[] args) {

        String a = "wukong2";

        final String b = "wukong";

        String d = "wukong";

        String c = b + 2;// b被final修饰、编译器是知道b的值的，所以编译器就知道c的值是和a的值相同的，就直接把c的引用指向了a指向的“wokong2”的地址

        String e = d + 2;//运行时才确定的e，e是在堆上创建的

        //a c 指向常量池

        System.out.println((a == c));//true

        System.out.println((a == e));//false

    }

}
```

**题二：**

```java
public class FinalStringDemo2 {

    private static String getDashixiong() {

        return "wukong";

    }

    public static void main(String[] args) {

        String a = "wukong2";

        //编译时期无法确定b的值

        final String b = getDashixiong();

        String c = b + 2;

        System.out.println(a == c);//false

    }

}
```

