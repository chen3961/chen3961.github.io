---
layout:     post
title:      总有一些java代码超出意外
date:       2023-09-22
summary:    总结一下，不容易认识到的java使用错误
categories: java
---
__Arrays.asList()__

这个方法经常用，是将数组转成列表的常用方法。这个方法的输入参数是一个数组，可以将数组转化成java定义的
List对象。通常来说都没有问题。

可以这样用：
```java
List<String> val = Arrays.asList("a", "b", "c");
```
也可以这么用
```java
List<String> val = Arrays.asList(new String[]{"a", "b", "c"});
```
还可以这么用
```java
List<Integer> val = Arrays.asList(1，2，3);
```
但是不能这么用
```java
List<Integer> val = Arrays.asList(new int[]{1,2,3});
```
上面这个用法，生成的不是List<Integer>,而是List<int[]>. 正常来说，上面这种写法编译器是不会通过的，一看就能看出来，但是如果没有赋予变量，在流式的算法中
可能就看不出来了，会导致后续的结果不一致。
Arrays.asList 其实是一个模板方法，是带类型信息的，原生数据类型会先封箱后在运算，但是java的编译器并不会自动给int[]这样的原生数组封箱再展开就有了这样的问题。

__Double.MAX_VALUE__

有时候常常需要用浮点数的最大值作为起点和标志，如果需要检查数据是否被合法设置过的时候，需要判断当前的变量是否与最大值相等或者接近。
在这里的判断条件一定要用等号，用常用的浮点数不等号会引起不必要的麻烦。

例如：
```java
double a = Double.MAX_VALUE
dobule b = Double.MAX_VALUE
if (b < a -1) {
    
}
```
这样的判断条件完全没有无效，这样的浮点数每浮动一次，都远远大于1

__java.lang.Integer cannot be cast to java.lang.Long__
这是见到过的Java里面最傻的异常。从Long转换到Integer，报个异常还可以理解，毕竟可以值发生了变化，从Integer强转到Long居然不行就无法理解了。不理解这个为什么做不到编译器里面。

这里面虽然是因为指针的类型导致的问题，但是使用起来极其不爽。然后官方解法的建议是通过将原来的Long转成String再转成Integer，就和脱裤子放屁一样。

primitive type是没有问题的，自己写的代码应该没有什么问题，就是在序列化的时候就惨了，想要Integer的时候变成了Long，想要Long的又变成了Integer

```java
//完全没有问题
long a = 1l;
int b = (int)a;

//这里就会惨惨
ObjectMapper ob = new ObjectMapper();
Map<String, Object> val = ob.readValue("{'a':1, 'b':2}", Map.class);
int aValue = val.get("a"); 
```