#### effective java之builder模式

在实际业务中，往往会遇到一个类包含很多成员变量的情况，因此在实例化该类时，要么需要包含不同参数的构造函数，要么需要在默认构造函数的基础上调用多个setter方法为该实例的各成员赋值。

针对第一种情况，当参数的个数很多，构造函数的参数相应会很多，这样将导致在客户端调用的时候 构造函数参数表意不明的情况；对于第二种情况，是在创建实例后调用setter函数对成员变量赋值，虽然在客户端上可读性变强，但在多线程下，将不能保证各个各setter操作的原子性。

这里对第一、第二种情况分别进行说明，最后介绍Builder模式，及lombok的@Builder大法。



##### 1. 使用多个构造函数

```java
@Data
public class Builder {
    private String param1;
    private String param2;
    private String param3;
    private String param4;

    public Builder() {}
    
    public Builder(String param1) {
        this.param1 = param1;
    }

    public Builder(String param1, String param2) {
        this.param1 = param1;
        this.param2 = param2;
    }

    // 三个参数的构造函数
    // 四个参数的构造函数
}

// 调用时
public static void main(String[] args) {
    Builder builder = new Builder("xixi");
    // 其他生成实例方式
    Builder builderOne = new Builder("xixi","haha","xixi", "haha");
}
```

以上面这个Builder类为例，它包含四个参数，且每个参数都是String类型，所以在这里写了五个构造函数，适应对不同成员变量赋值的情形。但是，在这种情况下存在以下明显的缺点：

- 含一个参数的构造函数只能对param1(其中一个参数赋值)，不能对任意参数赋值；
- 含两个参数的构造函数只能对其中两个成员变量赋值，不能对任意两个成员变量赋值，...；
- 在参数很多的时候，根本分不清每个参数的意义（表意不明），客户端也不方便调用，或者调用的时候极易出现赋值错误的情况。

##### 2. 使用setter函数

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Builder {
    private String param1;
    private String param2;
    private String param3;
    private String param4;
}

    // 调用时
    public static void main(String[] args) {
        Builder builder = new Builder();
        builder.setParam1("xixi");
        builder.setParam2("haha");
        builder.setParam2("xixi");
        builder.setParam2("haha");
    }
```

上述的这种方式，虽然可读性变强，但随着参数个数越多，调用的setter函数也就越多，看着不太优雅；且这样是线程不安全的，存在多个线程同时修改一个实例参数的情况。

##### 3. Builder模式的原理

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
@ToString
public class OuterClass {
    // 成员变量
    private String param1;
    private String param2;
    private String param3;
    private String param4;
    // 1.内部静态类Builder
    public static class Builder {
        // 2.Builder类有与OuterClass相同的成员变量
        private String param1;
        private String param2;
        private String param3;
        private String param4;

        // 3.Builder默认构造函数
        public Builder() {}
        // 还可以有其他构造函数
        public Builder(String param1, String param2) {
            this.param1 = param1;
            this.param2 = param2;
            // 对其他参数设默认值
            this.param3 = "xixi";
            this.param4 = "haha";
        }
        
        // 4.对不同的成员变量赋值的函数，并返回Builder对象实例
        public Builder param1(String param1) {
            this.param1 = param1;
            return this;
        }

        public Builder param2(String param2) {
            this.param2 = param2;
            return this;
        }

        public Builder param3(String param3) {
            this.param3 = param3;
            return this;
        }

        public Builder param4(String param4) {
            this.param4 = param4;
            return this;
        }

        // 5.build函数返回一个OuterClass实例
        public OuterClass build() {
            return new OuterClass(this);
        }
    }
    
    // 6.OuterClass的构造函数，以内部静态类Builder为参数
    public OuterClass(Builder builder) {
        this.param1 = builder.param1;
        this.param2 = builder.param2;
        this.param3 = builder.param3;
        this.param4 = builder.param4;
    }
}
```

从上面看出，Builder模式的几个关键点在于：

- 拥有一个静态内部类Builder，其包含于OuterClass相同的成员变量；
- 内部类包含构造函数；
- 内部类拥有可以设置其各成员变量的函数，且这些函数返回内部类实例；
- 内部类包含build函数，返回一个OuterClass实例；
- OuterClass包含一个构造函数，其参数为内部类实例。

**生成OuterClass实例**

```java
	// 调用一下OuterClass
    public static void main(String[] args) {
        OuterClass bean = new OuterClass.Builder().param1("xixi")
                .param2("haha")
                .param3("xxxxx")
                .param4("hhhhhhh")
                .build();

        // 输出：OuterClass(param1=xixi, param2=haha, param3=xxxxx, param4=hhhhhhh)
        System.out.println(bean.toString());

        OuterClass bean2 = new OuterClass.Builder("xixi", "haha")
                .param3("xxxxxxxxxxxx")
                .build();
        // 输出： OuterClass(param1=xixi, param2=haha, param3=xxxxxxxxxxxx, param4=haha)
        System.out.println(bean2.toString());
    }
```

这样一来，我们就可以通过上面的方法来生成OuterClass实例，且可以根据需要随意的设置其中的参数，解决了

`1`、`2` 节的问题。但是Builder模式同样有不足：

- 为了创建对象，必须先创建它的构建器(存在开销)；
- 代码变得更冗长了。

但是，在参数很多或者字段扩张厉害的情况下，在一开始使用构建器却是不错的选择。

##### 4. lombok @Builder注解的使用

如果要使用构建器，我们不需要自己去编写额外的代码，lombok的@Builder注解能帮我们实现这个功能。其使用如下：

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
@ToString
@Builder
public class OuterClass {
    private String param1;
    private String param2;
    private String param3;
    private String param4;

    // 调用一下OuterClass
    public static void main(String[] args) {
        OuterClass bean = OuterClass.builder()
                .param1("xixi")
                .param2("haha")
                .param3("xxxxxxxx")
                .build();

        // 输出：OuterClass(param1=xixi, param2=haha, param3=xxxxxxxx, param4=null)
        System.out.println(bean);
    }
```



##### 参考链接

lombok大法的一些使用：https://www.jianshu.com/p/2ea9ff98f7d6

github完整代码示例：https://github.com/HorizonLiu/effective.java.learning