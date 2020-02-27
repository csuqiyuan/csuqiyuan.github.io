---
title: Java 8 新特性——Optional  使用详解
author:
  name: qiyuan
  avatar: https://res.cloudinary.com/dkzvjuptx/image/upload/v1578820041/info/favicon_s4pmzz.jpg
top: false
date: 2020-02-25 21:55:13
description:  使用 Optional，别用 null
thumbnail: https://res.cloudinary.com/dkzvjuptx/image/upload/v1582735810/Java/Java%208%20%E6%96%B0%E7%89%B9%E6%80%A7%E2%80%94%E2%80%94%E7%94%B1%20Optional%20%E5%BC%95%E5%8F%91%E7%9A%84%E6%80%9D%E8%80%83/java_8_mygmrc.png
tags:
- Java
- Java 8 新特性
categories: Java
keywords: Java, Java8新特性, Optional, Lambda, 函数式接口, 方法引用
---

最近在被 code review 的时候，组里大佬对我代码中几处使用 if 判断对象是否为空的地方提了一句话：

“使用 Optional ，能不用 null 就不用 null”

Java 8 的发布已经有很长时间了，其新特性几乎是面试必问，老生常谈的问题，但我在实际开发中却很少用到，也很少见人用到，甚至在某程序员社区看到有人发帖称“老板禁止我用 Stream ，说怕别人不会用...”。

之前在朋友圈夸过，我们组的代码是我见过最简洁规范的 Java 代码，能用一行代码解决的事情绝不用两行代码。虽然我能随口说出Java 8 的新特性，但使用经验实在太少，用句学生时代的老话说：“没有把书本上的知识变成自己的知识”。于是我在网上查阅了关于 Optional 使用的资料。

## Optional 的作用

Optional 类是一个可以包含可选值的包装类，也可以理解为一个包含可选对象的容器。它所包含的对象可以为空，是实现 Java 函数式编程的强劲一步。

但 Optional 类主要解决的问题是每个程序员都非常熟悉的，臭名昭著的空指针异常（NullPointerException）——这是网上的说法，就我的感觉来讲，Optional 并没有解决空指针异常，而是简化里对于空指针异常的处理过程。

在没有 Optional 时为了解决空指针异常，常常要在访问每一个对象前使用逻辑判断语句对值做检查，下面是个例子：

假设有下面三个类，现在拥有一个 User 对象，想获取 isoCode 值：

```java
public class User {
    private Address address;

    public Address getAddress() {
        return address;
    }
    // ...
}
public class Address {
    private Country country;

    public Country getCountry() {
        return country;
    }
    // ...
}
public class Country {
    private String isoCode;
    
    public String getIsoCode() {
        return isoCode;
    }
}
```

下面是两种方案，方案一是不做任何处理，极有可能报 NullPointerException，方案二是使用逻辑判断语句做检查：

```java
// 方案一，容易出现 NullPointerException
String isocode = user.getAddress().getCountry().getIsocode().toUpperCase();

// 方案二
if (user != null) {
    Address address = user.getAddress();
    if (address != null) {
        Country country = address.getCountry();
        if (country != null) {
            String isocode = country.getIsocode();
            if (isocode != null) {
                isocode = isocode.toUpperCase();
            }
        }
    }
}
```

可以看到方案二为排除 NullPointerException 而写的代码很低效，如果需要对 对象 为空的情况做处理，那么代码将更加冗长，难以维护。讲到这里想到了一个经典笑话——特工说“我拿到了敌军的系统代码，但只有最后几千行”，指挥官大喜：“快让我看看”，结果只见：“...... }}}}}}}}}}}}}}}}}}}}}}}}} ......”。

我们先来看看 Optional 的常用方法。

## Optional 常用方法详解

|      | **方法及描述**                                               |
| ---- | ------------------------------------------------------------ |
| 1    | **Optional()**<br>私有构造方法，被 empty() 方法用于创建空 Optional 对象 |
| 2    | **Optional(T value)**<br>私有构造方法。被 of() 方法用于创建带值的Optional 对象 |
| 3    | **static\<T> Optional<T> empty()**<br/>返回空的 Optional 实例 |
| 4    | **static \<T> Optional<T> of(T value)**<br/>返回一个指定非null值的Optional |
| 5    | **static \<T> Optional<T> ofNullable(T value)**<br/>如果为非空，返回 Optional 描述的指定值，否则返回空的 Optional |
| 6    | **T get()**<br/>如果在这个Optional中包含这个值，返回值，否则抛出异常：NoSuchElementException |
| 7    | **boolean isPresent()**<br/>如果值存在则方法会返回true，否则返回 false |
| 8    | **void ifPresent(Consumer<? super T> consumer)**<br/>如果值存在则使用该值调用 consumer , 否则不做任何事情 |
| 9    | **T orElse(T other)**<br/>如果存在该值，返回值， 否则返回 other |
| 10   | **T orElseGet(Supplier<? extends T> other)**<br/>如果存在该值，返回值， 否则触发 other，并返回 other 调用的结果 |
| 11   | **\<X extends Throwable> T orElseThrow(Supplier<? extends X> exceptionSupplier)**<br/>如果存在该值，返回包含的值，否则抛出由 Supplier 继承的异常 |
| 12   | **Optional\<T> filter(Predicate<? super T> predicate)**<br>如果值存在，并且这个值匹配给定的 predicate，返回一个Optional用以描述这个值，否则返回一个空的Optional |
| 13   | **\<U> Optional\<U> map(Function<? super T, ? extends U>**<br>如果有值，则对其执行调用映射函数得到返回值。如果返回值不为 null，则创建包含映射返回值的Optional作为map方法返回值，否则返回空Optional |
| 14   | **\<U> Optional\<U> flatMap(Function<? super T, Optional\<U>> mapper)**<br>如果值存在，返回基于Optional包含的映射方法的值，否则返回一个空的Optional |

除去以上方法，Optional 类还包含 equals、hashCode、toString 等重写 Object 类的方法。

## 常用方法使用

**empty() 和 get()**

empty() 用于创建一个空的 Optional 容器，看看源码：

```java
public final class Optional<T> {
    private static final Optional<?> EMPTY = new Optional<>();
    
    private Optional() {
        this.value = null;
    }
    
    public static<T> Optional<T> empty() {
        @SuppressWarnings("unchecked")
        Optional<T> t = (Optional<T>) EMPTY;
        return t;
    }
}
```

私有构造方法、私有静态对象、公有静态方法，这里使用了饿汉式单例模式。

get() 方法用于取出 Optional 容器中的对象，但如果对象为空，将报 NoSuchElementException 异常，原因可见源码：

```java
public T get() {
    if (value == null) {
        throw new NoSuchElementException("No value present");
    }
    return value;
}
```

**of() 和 ofNullable()**

静态方法，都是用于创建包含值的 Optional 容器，区别就是，of() 值不能为 null ，否则报 NullPointerException ，ofNullable() 可以传 null 。因此使用 of() 时应明确对象不能为 null ，如果对象可能为 null，应使用 ofNullable() 。另外，这里已经不是使用单例模式创建的对象了，见源码：

```java
private Optional(T value) {
    this.value = Objects.requireNonNull(value);
}

public static <T> Optional<T> of(T value) {
    return new Optional<>(value);
}

public static <T> Optional<T> ofNullable(T value) {
    return value == null ? empty() : of(value);
}
```

可以看到，每次调用这两者，都会新创建一个对象。ofNullable() 进行了一次判断，跟据传入值是否为空，决定使用 empty() 或 of() 方法。

**isPresent() 和 ifPresent(Consumer<? super T> consumer)**

从这里开始，便是 Optional 使用的关键了。

isPresent() 用于判断 Optional 内是否为空，如果不为空返回 true，为空返回 false，和 List 中的 isEmpty() 恰好相反。

ifPresent(Consumer<? super T> consumer) 除了判断是否为空外，还传入了一个 Consumer (消费者)函数式接口，如果不为空，会执行 Lambda 表达式。Consumer 是 Java 定义好的函数式接口，与 Lambda 表达式配合，可以实现函数式编程。Consumer 消费者接口接收一个参数，返回值为 void 。

直接贴使用代码，感兴趣的可以自己查阅源码：

```java
// 示例一
// getString() 是返回一个 String 值的假想方法，这个值可为 null
Optional<String> opt = Optional.ofNullable(getString());
if(opt.isPresent()){
    System.out.println(opt.get());
}

// 示例二
opt.ifPresent(s -> System.out.println(s));
// 亦可写作 opt.ifPresent(System.out::println);

// 传统写法
String str = getString();
if(str != null){
    System.out.println(str);
}
```

当然，输出 String 并不会报 NullPointerException，这只是个简单的例子展示，传入 Optional 的类型可以变为任何一个对象，对其做的操作也不仅局限于输出。可以看到 示例二 明显更简洁和美观。

**orElse(T other)、orElseGet(Supplier<? extends T> other) 和 orElseThrow(Supplier<? extends X> exceptionSupplier)**

orElse 方法的用法是，当容器内不为空时，返回容器内的对象，否则返回参数传入的对象；而 orElseGet 的用法是：当容器不为空时，返回容器内的对象，否则执行传入的 Supplier (供应者)函数式接口。Supplier 供应者接口不接收参数，返回一个 T 泛型。

```java
/**
 * 假如有如下场景。
 * 通过 getUser() 获取一个 User 对象，可能为空。如果为空时，需要返回一个默认对象。
**/

// 传统解决方案
User user = getUser();
User result ;
if(user == null){
    result = new User();
}
else{
    result = user;
}
return result;

// orElse 示例
Optional<User> opt = Optional.ofNullable(getUser());
User result = opt.orElse(new User("..."));
return result

// orElseGet 示例
Optional<User> opt = Optional.ofNullable(getUser());
User result = opt.orElseGet(() -> new User("..."));
return result;
```

可以看到，使用 Optional 的方案代码简洁许多。orElse 和 orElseGet 的区别似乎仅仅是一个传入对象，一个传入函数式接口（Lambda 表达式）。除此之外，它们的区别还在于：orElse 不管存不存在值，后面的创建新对象的过程是必然发生的；而 orElseGet 就仅仅在值不存在时触发创建新对象的代码。后者相对来说更节省性能。

orElseThrow 则会在值为空时抛出异常，可以指定抛出的异常。

```java
// 传统解决方案
User user = getUser();
if(user == null){
    throw new IllegalArgumentException();
}
// orElseThrow 方案
Optional<User> opt = Optional.ofNullable(getUser());
opt.orElseThrow(() -> new IllegalArgumentException());
// 进一步简化
opt.orElseThrow(IllegalArgumentException::new);
```

**map(Function<? super T, ? extends U> 和 flatMap(Function<? super T, Optional\<U>> mapper)**

map 和 flatMap 都传入了 Function 函数式接口，即接收一个参数，有一个返回值。map 和 flatMap 都可以把当前 Optional 对象转化为其他类型的 Optional 对象，但实现方式略有不同。

map 是将一个对象包装成 Optional ，而 flatMap 直接返回了包含对象的 Optional 。这点从接口函数的参数定义中可以看出：? extends U 和 Optional\<U>> mapper; 即 map 函数式接口返回的是一个普通对象，flatMap 函数式接口返回的是一个 Optional 对象。

这两个方法可以对返回值进行链式调用。

下面将我们最初例子使用 Optional 重写一遍最初获取 isoCode 的例子：

```java
class User {
    private Address address;

    public User(){
        address = new Address();
    }

    public Address getAddress() {
        return address;
    }
}
class Address {
    private Country country;

    public Address(){
        country = new Country("test");
    }
    
    public Country getCountry() {
        return country;
    }
}
class Country {
    private String isoCode;

    public Country(String isoCode){
        this.isoCode = isoCode;
    }

    public String getIsoCode() {
        return isoCode;
    }
}
// 经典解决方案
if (user != null) {
    Address address = user.getAddress();
    if (address != null) {
        Country country = address.getCountry();
        if (country != null) {
            String isocode = country.getIsocode();
            if (isoCode != null) {
                isoCode = isocode.toUpperCase();
            }
        }
    }
}
System.out.println(isoCode)

// Optional map 方案：
Optional<User> opt = Optional.ofNullable(user);
String isoCode = opt.map(u -> u.getAddress())
    .map(a -> a.getCountry()) // 可使用方法引用简化 map(User::getContry)
    .map(c -> c.getIsoCode())
    .orElse("default")
    .toLowerCase();
System.out.println(isoCode);

// Optional flatMap 方案：需要对对象做一些修改
// getAddress() 和 getCountry() 方法修改如下
public Optional<Address> getAddress() {
    return Optional.ofNullable(address);
}

public Optional<Country> getCountry() {
    return Optional.ofNullable(country);
}

// flatMap 解决方案如下
Optional<User> opt = Optional.ofNullable(user);
String isoCode = opt.flatMap(u -> u.getAddress())
    .flatMap(a -> a.getCountry())
    .map(c -> c.getIsoCode())
    .orElse("default")
    .toLowerCase();
System.out.println(isoCode);
```

如果调用链过程中有任何一个对象为空，将返回默认值 “default”，过程简洁明了。也可以使用方法引用进一步简化这个过程。

**filter(Predicate<? super T> predicate)**

filter() 接受一个 Predicate 函数式接口参数，这个接口接收一个参数，返回 boolean 值。如果这个参数返回的结果为 true，filter 函数返回这个 Optional 对象本身，否则返回一个空的 Optional 对象。

```java
User user = new User("anna@gmail.com", "1234");
Optional<User> result = Optional.ofNullable(user)
    .filter(u -> u.getEmail() != null && u.getEmail().contains("@"));
System.out.println(result.get().getEmail());
```

filter 函数可以用于过滤某些不符合要求的参数。

## Java 9 的 Optional 新特性

新增了三个方法：or()、ifPresentOrElse()、stream()。

**后面我会持续更新这三个方法的使用，以及 Lambda 表达式、函数式接口、方法引用等特性的用法**

