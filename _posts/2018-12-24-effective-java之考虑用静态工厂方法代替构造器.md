#### 前言

接触java半年以来，发现对java的认识少之又少，很多时候又是在写业务，所以给我的感觉是，基本功能会用，但很多有意思的功能仍然没有用到。且每次都是用什么查什么，很有可能根据自己的常识误以为java的某个特性应该这样用，但其实不然。所以从今天起，以《effective java》这本书为切入点，看看别人都是怎样用的，前辈们有什么样的建议，并在这里稍作记录。

#### 考虑用静态工厂方法代替构造器

##### 创建对象的方法

首先，我们是怎样来创建对象的呢？我想大家脑子中一定会出现类似下面的代码：

```java
Person person = new Person();
```

对，new一个。new对象是调用类的构造方法，我们还可以通过类的静态方法返回一个实例。

`创建对象的方法：`

`1. 使用类的共有构造器`

`2. 使用类的静态方法返回一个实例`

##### 使用静态方法创建对象的优点

1. 静态方法的名字由我们自己命名，而构造方法必须与类同名；

```java
// 定义两个类
class PersonA {
    public PersonA() {}
    
    // 调用该函数时必须先实例化PersonA对象
    public void test() {
        System.out.println("just test in PersonA \n")
    }
}

class PersonB {
    public PersonB() {}
    // 静态工厂生产方法
    public static PersonB newInstance() {
        return new PersonB();
    }
    // 调用该函数不必先实例化PersonB对象，可以通过PersonB.test()直接调用
    public static void test() {
        System.out.println("just for test in PersonB \n");
    }
}

// 实例化两个对象
PersonA pA = new PersonA();
PersonB pB = PersonB.newInstance();
```

2. 构造方法每次调用都会创建一个对象，而静态工厂方法每次调用的时候就不会创建一个新的对象，这对于一个频繁创建对象的程序来说，可以极大的提高性能，单例模式就是这样实现的。

```java
// 单例模式
class PersonC {
    // 1.声明一个静态成员pC
    private static PersonC pC;
    // 2.私有化构造函数
    private PersonC() {}
    // 3.通过静态函数获取一个该类的实例
    public static PersonC newInstance() {
        if (pC == null) {
            return new PersonC();
        }
    }
}
```

单例模式包含以上三个主要关键点，若考虑到多线程安全性，在`newInstance`函数生成 `PersonC` 对象实例时，还需使用`synchronized`关键字。

3. 静态工厂方法可以返回任何类型的对象；
4. 静态工厂方法在创建参数化类型实例时，可以使代码变得更简洁。

```java
// 普通的实例化
private Map<String, List<String>> map = new HashMap<String, List<String>>();

// 使用静态方法实例化
class MyHashMap {
    public static <K,V> HashMap<K,V> newInstance() {
        return new HashMap<K,V>();
    }
}

// 接下来就可以使用静态方法来实例化一个HashMap，简洁了些
Map<String, List<String>> map = MyHashMap.newInstance();
```

##### 静态工厂方法创建对象的缺点

1. 若一个类只能通过静态工厂方法获取实例，那么该类的构造函数就不能是共有的或受保护的，那么该类就不会有子类，即该类不能被继承。单例模式就必须要私有化构造函数；
2. 静态工厂方法和其他静态方法一样，都是函数，所以在区分时最好使用一些惯用名称，如valueOf、getInstance等等。



##### 总结

这一节看似总结了静态工厂方法的各种优点和缺点，但总的来说，无非就是`静态函数和普通成员函数、静态函数和构造函数` 的区别：

- 静态函数可以直接通过类名调用，不需要实例化对象；
- 静态函数的名字和普通函数一样，可以自定义

