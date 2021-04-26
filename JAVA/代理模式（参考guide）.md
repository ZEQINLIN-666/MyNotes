# 静态代理+JDK/CGLIB动态代理实战

## 1 什么叫代理模式？

代理模式是一种比较好理解的设计模式。简单来说就是：

- 我们使用代理对象来代替真实对象（real object）的访问，这样就可以在不修改原目标对象的前提下，提供额外的功能操作，扩展目标对象的功能。

**代理模式的主要作用是扩展目标对象的功能，比如说在目标对象的某个方法执行前后你可以增加一些自定义的操作。**

举个例子，你要交班费，就把班费交给班长，叫他代交。班长就看做是代理我的代理对象，我就是代理对象的目标对象，代理的行为(方法)就是交班费。



代理模式有静态代理和动态代理两种实现方式。

## 2 静态代理

在实现和应用的角度：

- 静态代理中，我们对目标对象的每个方法的增强都是手动完成的，非常不灵活，比如接口一旦新增加方法，目标对象和代理对象都要进行修改，且麻烦，需要对每个目标类都单独写一个代理类。
- 实际应用场景非常非常少，日常开发几乎看不到使用静态代理的场景

在JVM层面来说：

- 静态代理的编译的时候就将接口、实现类、代理类这些都变成了一个个实际的class文件

静态代理实现步骤：
1.定义一个接口及其实现类；
2.创建一个代理类同样实现这个接口
3.将目标对象注入进代理类，然后在代理类的对应方法调用目标类中的对应方法。这样的话，我们就可以通过代理类屏蔽对目标对象的访问，并且可以在目标方法执行前后做一些自己想做的事情。



代码实现：

1.定义发送短信的接口

```java
public interface SmsService{
    String send(String message);
}
```

2.实现发送短信的接口

```java
public class SmsServiceImpl implements SmsService{
    public String send(String message){
        System.out.println("send message:"+ message);
        return message;
    }
}
```

3.创建代理类并同样实现发送短信的接口

```java
public class SmsProxy implements SmsService{
    private final SmsService smsService;
    
    public SmsProxy(SmsService smsService){
        this.smsService = smsService;
    }
    
    @Override
    public String send(String message){
        //调用方法前我们可以添加自己的操作
        System.out.println("before method send()");
        smsService.send(message);
        //调用方法之后，我们同样可以添加自己的操作
        System.out.println("after method send()");
        return null;
    }
}
```

4.实际使用

```java
public class Main{
    public static void main(String[] args){
        SmsService smsService = new SmsServiceImpl();
        SmsProxy smsProxy = new SmsProxy(smsService);
        smsProxy.send("java");
    } 
}
```



运行上述代码之后，控制台打印出：

```java
before method send（）
send message:java
after method send（）
```

 可以输出结果看出，我们已经增加了SmsServiceImp1的send（）方法。