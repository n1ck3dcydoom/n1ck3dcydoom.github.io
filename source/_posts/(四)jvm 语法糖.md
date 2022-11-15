---
title: (四)jvm 语法糖
date: 2022-11-15 17:27:55
index_img: /img/java.png
categories:
  - jvm
tags:
  - jvm
---

### 泛型

泛型的本质是参数类型化或者参数多态化; 即可以将操作数类型指定为一种特殊类型,在真正运行时能够被解析回实际的类型

### 泛型的类型擦除

在编译时 `List<String>` 和 `List<Integer>` 两个泛型会被替换为 `裸类型 Raw Type` ,因此对于 java 来说  `List<String>` 和 `List<Integer>` 在运行时其实就是同一个类型 `List` ,这就是 `类型擦除`

`类型擦除` 被当做一个 java 的语法糖,在实际运行时需要由 jvm 将其还原为支持不同类型的语法

1. 让 `ArrayList<String>` 成为 `ArrayList` 的子类
2. 通过类型擦除让 `ArrayList<String>` 还原为裸类型 `ArrayList` ,在实际操作时加入强制类型转换,将 `Object` 转换为 `String` 类型

`类型擦除` 带来的麻烦,这使得泛型不支持基本类型,例如 `ArrayList<int>` 或者 `List<long>` 都是不合法的

因为 java 不支持 `int` `long` 与 `Object` 之间的转换

为了能够让基本类型支持泛型,java 不得已又引入了新的语法糖 `自动装箱` 和 `自动拆箱`

简单来说,如果要使用 `List<long>` 就干脆使用 `List<Long>` 替换,只不过在运行自动的将包装类拆箱为基本类,将基本类装箱为包装类

这就成为了 java 泛型性能低下的根本原因,引入了大量的自动装箱和拆箱的额外操作

而且由于 `类型擦除` jvm 在运行时无法获得实际的类型,因为泛型全部变成了裸类型

### 自动拆箱,自动装箱,增强 for 循环,变长参数

* 自动拆箱: 编译后会被还原为对应的拆箱方法 `Integer.intValue()`
* 自动装箱: 编译后会被还原为对应的装箱方法 `Intger.valueOf()`
* 增强 for 循环: 即 `for(int n: nums)` 编译后会被还原为迭代器遍历,其实这就是为什么增强 for 循环要求循环的对象必须实现 `Iterable` 接口,而且对于基本类型的增强 for 循环,还会被自动装箱为包装类
* 变长参数: 即 `foo(...String)` 编译后会被还原为一个 `String[]` 的参数

自动拆箱和自动装箱引起的 bug

对于包装类 `Integer` 在做 `==` 判断的时候,不会自动拆箱,以及 `equals()` 方法也不会处理数据转换的问题

### 条件编译

jvm 会在编译 `if` 语句时时判断一些布尔表达式的结果,把那些永远不可达的语句去掉

```java
public static void main(String[] args) { 
    if (true) { 
        System.out.println("block 1"); 
    } else { 
        System.out.println("block 2"); 
    } 
}
```

在编译后就只有一条语句了

```java
public static void main(String[] args){
    System.out.println("block 1");
}
```

除了 `if` 以外其他的条件判断语句,如果出现永远不可达的语句则会直接拒绝编译

```java
public stataic void main(String[] args) {
    // Unreachable code
    while (false){
        System.out.println("block 3");
    }
}
```

此时编译会直接报错

### 其他语法糖

最常用的几个就是 `try-catch-resource` 语法糖,对于那些实现了 `CloseAble` 接口的资源,可以通过 `try-catch-resource` 语法糖来避免大量的 `catch` 关闭资源的操作

还有就是 `switch` 操作支持字符串和枚举值,这也是语法糖