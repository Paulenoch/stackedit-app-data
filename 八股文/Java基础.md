## 1. 什么是字节码，字节码的好处是什么
Java 中，JVM 可以理解的代码就叫做字节码（即扩展名为 `.class` 的文件），它不面向任何特定的处理器，只面向虚拟机。由于字节码并不针对一种特定的机器，因此，Java 程序无须重新编译便可在多种不同操作系统的计算机上运行。

## 2. Java与C++的区别
1. Java不提供指针直接访问内存，程序内存更加安全
2. Java的类是单继承的，C++是多继承，但Java可以继承多个类
3. Java有自动内存管理和垃圾回收机制GC
4. C++同时支持方法重载和操作符重载，Java只能方法重载

## 3. 位移运算符
`<<`左移n位，相当于乘以2的n次方
`>>`右移n位，相当于除以2的n次方

## 4. 基本类型和包装类的区别
1. 包装类可用于泛型，基本类型不可
2. 本数据类型的局部变量存放在 Java 虚拟机栈中的局部变量表中，基本数据类型的成员变量（未被 `static` 修饰 ）存放在 Java 虚拟机的堆中。包装类型属于对象类型，我们知道几乎所有对象实例都存在于堆中。
```java
public class Test {
    // 成员变量，存放在堆中
    int a = 10;
    // 被 static 修饰的成员变量，JDK 1.7 及之前位于方法区，1.8 后存放于元空间，均不存放于堆中。
    // 变量属于类，不属于对象。
    static int b = 20;

    public void method() {
        // 局部变量，存放在栈中
        int c = 30;
        static int d = 40; // 编译错误，不能在方法中使用 static 修饰局部变量
    }
}
```
3. 基本类型占用空间较小
4. 基本类型不赋值会有默认值，包装类不赋值为null
5. 基本类型通过==比较，包装类型通过`==`比较内存地址，通过.equals()比较值


## 5. 包装类的缓存机制
Java 基本数据类型的包装类型的大部分都用到了缓存机制来提升性能。

`Byte`,`Short`,`Integer`,`Long` 这 4 种包装类默认创建了数值 **[-128，127]** 的相应类型的缓存数据，`Character` 创建了数值在 **[0,127]** 范围的缓存数据，`Boolean` 直接返回 `TRUE` or `FALSE`。

对于 `Integer`，可以通过 JVM 参数 `-XX:AutoBoxCacheMax=<size>` 修改缓存上限，但不能修改下限 -128。实际使用时，并不建议设置过大的值，避免浪费内存，甚至是 OOM。


## 6. 什么是自动拆箱与装箱
装箱：```Integer i = Integer.valueOf(10)```
拆箱：```int n = i.intValue()```
如果频繁拆装箱的话，也会严重影响系统的性能。我们应该尽量避免不必要的拆装箱操作。

### 自动拆装箱引起NPE问题
![输入图片说明](/imgs/2025-03-26/5QxvIDmZclPMxGeE.png)
`flag ? 0 : i`这行代码中，0是基本数据类型int，返回数据时i会被强制拆箱成int类型，由于i的值是null，抛出了NPE异常

## 7. 静态变量有什么用
它可以被类的所有实例共享，无论一个类创建了多少个对象，它们都共享同一份静态变量。也就是说，静态变量只会被分配一次内存，即使创建多个对象，这样可以节省内存。
静态变量是通过类名来访问的，例如`StaticVariableExample.staticVar`（如果被 `private`关键字修饰就无法这样访问了）。

## 8. 静态方法为什么不能调用非静态成员
-   静态方法是属于类的，在类加载的时候就会分配内存，可以通过类名直接访问。而非静态成员属于实例对象，只有在对象实例化之后才存在，需要通过类的实例对象去访问。
-   在类的非静态成员不存在的时候静态方法就已经存在了，此时调用在内存中还不存在的非静态成员，属于非法操作。

## 9. 方法重写注意事项
1.  方法名、参数列表必须相同，子类方法返回值类型应比父类方法返回值类型更小或相等，抛出的异常范围小于等于父类，访问修饰符范围大于等于父类。
2.  如果父类方法访问修饰符为 `private/final/static` 则子类就不能重写该方法，但是被 `static` 修饰的方法能够被再次声明。
3.  构造方法无法被重写

## 10. 面向对象三大特征
1. 封装：类对外隐藏自己的状态信息，提供接口供外界操作
2. 继承：子类可以继承父类
3. 多态：表示一个对象具有多种的状态，具体表现为父类的引用指向子类的实例。
	-   引用类型变量发出的方法调用的到底是哪个类中的方法，必须在程序运行期间才能确定；
	-   多态不能调用“只在子类存在但在父类不存在”的方法；
	-   如果子类重写了父类的方法，真正执行的是子类重写的方法，如果子类没有重写父类的方法，执行的是父类的方法。

## 11. 浅拷贝、深拷贝、引用拷贝![输入图片说明](/imgs/2025-03-24/SClMMUPEIQC2oFRf.png)

## 12. hashcode()
`hashCode()` 的作用是获取哈希码（`int` 整数），也称为散列码。这个哈希码的作用是确定该对象在哈希表中的索引位置

`hashCode()` 和 `equals()`都是用于比较两个对象是否相等。

在一些容器（比如 `HashMap`、`HashSet`）中，有了 `hashCode()` 之后，判断元素是否在对应容器中的效率会更高（参考添加元素进`HashSet`的过程）！

如果 `HashSet` 在对比的时候，同样的 `hashCode` 有多个对象，它会继续使用 `equals()` 来判断是否真的相同。也就是说 `hashCode` 帮助我们大大缩小了查找成本。

## 13. String、StringBuffer、StringBuilder
1. `String`中的对象是可不变的，线程安全；`StringBuffer`对方法加了同步锁或者对调用的方法加了同步锁，所以是线程安全的。`StringBuilder` 并没有对方法进行加同步锁，所以是非线程安全的。
2. 每次对 `String` 类型进行改变的时候，都会生成一个新的 `String` 对象，然后将指针指向新的 `String` 对象
3. “+”和“+=”是专门为 String 类重载过的运算符，实际上是通过 `StringBuilder` 调用 `append()` 方法实现的，拼接完成之后调用 `toString()` 得到一个 `String` 对象
4. `String` 中的 `equals` 方法是被重写过的，比较的是 String 字符串的值是否相等

## 14. String s1 = new String("abc"); 创建了几个对象
-   字符串常量池中不存在 "abc"：会创建 2 个 字符串对象。一个在字符串常量池中，由 `ldc` 指令触发创建。一个在堆中，由 `new String()` 创建，并使用常量池中的 "abc" 进行初始化。
-   字符串常量池中已存在 "abc"：会创建 1 个 字符串对象。该对象在堆中，由 `new String()` 创建，并使用常量池中的 "abc" 进行初始化。

## 15. 使用try-with-resource代替try-catch-finally
在 `try-with-resources` 语句中，任何 catch 或 finally 块在声明的资源关闭后运行
```java
try (Scanner scanner = new Scanner(new File("test.txt"))) {
	while (scanner.hasNext()) { 
		System.out.println(scanner.nextLine()); 
	} 
} catch (FileNotFoundException fnfe) {
 fnfe.printStackTrace(); 
}
```
通过使用分号分隔，可以在`try-with-resources`块中声明多个资源。

不要把异常定义为静态变量，因为这样会导致异常栈信息错乱。每次手动抛出异常，我们都需要手动 new 一个异常对象抛出。

## 16. 值传递
Java 中将实参传递给方法（或函数）的方式是 **值传递**：

-   如果参数是基本类型的话，很简单，传递的就是基本类型的字面量值的拷贝，会创建副本。
-   如果参数是引用类型，传递的就是实参所引用的对象在堆中地址值的拷贝，同样也会创建副本。

## 17. 泛型擦除机制，为什么要擦除
Java在编译期间，所有的泛型信息都会被擦除，动态地将泛型T擦除为Object，或将T extend xxx擦除为其限定类型xxx

目的是为了引入泛型机制但不创建新的类型，减少虚拟机的运行开销

### 既然要擦除，为什么要用泛型不用Object
- 使用泛型可以在编译期间进行类型检测
- 使用Object需要手动添加强制类型转换
- 泛型可以使用自限定类型如T extends Comparable

## 18. 什么是通配符，有什么作用
通配符允许参数类型变化，解决泛型类型是固定的这一问题

```java
// 限制为Person的子类
<? extends Person>
// 限制为Manager的父类
<? super Manager>
```

### 和泛型T的区别
- T可以用于声明变量而?不行
- T一般用于声明泛型类或方法，?一般用于泛型方法的调用代码和形参
- T在编译期间会被擦除为限定类型或Object，通配符用于捕获具体类型

### 什么是无界通配符
无界通配符可以接受任何泛型类型数据，用于实现不依赖具体类型参数的简单方法，可以捕获参数并交由泛型方法进行处理
```java
void testMethod(Person<?> p) {
	//泛型方法自行处理
}
```

### List<?>和List有区别吗
`List<?> list`表示list是持有某种特定类型的list，但不知道具体类型，此时直接添加元素会报错
`List list`表示list持有的元素类型是object，可以添加任何类型的对象

## 19. 动态代理
**在 Java 动态代理机制中 `InvocationHandler` 接口和 `Proxy` 类是核心。**

`Proxy` 类中使用频率最高的方法是：`newProxyInstance()` ，这个方法主要用来生成一个代理对象。
```java
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        ......
    }
```
此方法共有3个参数：
1.  **loader** :类加载器，用于加载代理对象。
2.  **interfaces** : 被代理类实现的一些接口；
3.  **h** : 实现了 `InvocationHandler` 接口的对象；

要实现动态代理的话，还必须需要实现`InvocationHandler` 来自定义处理逻辑。 当我们的动态代理对象调用一个方法时，这个方法的调用就会被转发到实现`InvocationHandler` 接口类的 `invoke` 方法来调用
```java
public interface InvocationHandler {

    /**
     * 当你使用代理对象调用方法的时候实际会调用到这个方法
     */
    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}
```
1.  **proxy** :动态生成的代理类
2.  **method** : 与代理类对象调用的方法相对应
3.  **args** : 当前 method 方法的参数

### 代码示例：
1. 定义发送短信的接口
```java
public interface SmsService {
    String send(String message);
}
```
2. 实现发送短信的接口
```java
public class SmsServiceImpl implements SmsService {
    public String send(String message) {
        System.out.println("send message:" + message);
        return message;
    }
}
```
3. 定义一个jdk动态类
```java
public class DebugInvocationHandler implements InvocationHandler {
    /**
     * 代理类中的真实对象
     */
    private final Object target;

    public DebugInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws InvocationTargetException, IllegalAccessException {
        //调用方法之前，我们可以添加自己的操作
        System.out.println("before method " + method.getName());
        //`invoke()` 方法: 当我们的动态代理对象调用原生方法的时候，最终实际上调用到的是 `invoke()` 方法，然后 `invoke()` 方法代替我们去调用了被代理对象的原生方法。
        Object result = method.invoke(target, args);
        //调用方法之后，我们同样可以添加自己的操作
        System.out.println("after method " + method.getName());
        return result;
    }
}
```
4. 获取代理对象的工厂类
```java
public class JdkProxyFactory {
    public static Object getProxy(Object target) {
        return Proxy.newProxyInstance(
                target.getClass().getClassLoader(), // 目标类的类加载器
                target.getClass().getInterfaces(),  // 代理需要实现的接口，可指定多个
                new DebugInvocationHandler(target)   // 代理对象对应的自定义 InvocationHandler
        );
    }
}
```
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTY1MTgyNzcyNSwtOTM4NDg1ODk2LC05MT
k5MTUwOTUsLTM2OTcxMDk4NCwtMTY2MTg1MzMzMSwxMTc3MjI1
MDI3LDE4ODg4NDUzNzksLTM4NjE3OTE5NiwxNTg1NDI1MDY0LD
c0ODcwODQ5NSwtMTI0NzYzNzA1M119
-->