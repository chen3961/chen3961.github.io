---
layout:     post
title:      lombok中@Data注解的一个坑
date:       2023-06-26
summary:    使用@Data时重写了对象的equals方法，导致逻辑混乱
categories: java
---
在一次代码编写中，遇到一个大坑。java里面的HashMap对象用自定义的数据结构作为key，由于时对象类型，默认key生效就是对象的内存地址，同时这个逻辑也是符合上下文逻辑的。整个代码的设计也是按照这个逻辑进行的。

但是在实际的运行过程中，同一个对象的引用在HashMap两次containsKey方法校验时均判断为false，即使再第一次false的时候已经把这个对象作为key put到HashMap中去了，行为及其奇怪。更奇怪的是，同一个HashMap用同一个对象做的key的两次put居然都成功了，在HashMap产生了两条数据，可是在用相同对象的引用当key去get数据的时候居然get不到。这个行为困扰了很久，因为整个逻辑就是按照相同对象的内存地址来设计的，预期和实际就是同一个内存对象。debug的时候发现key的内存地址也是一致的，所以完全不理解代码的执行顺序。

经过长时间的对比和实验，才发现造成这个原因是@Data注解引起的，它重写了自定义对象的equals方法导致按照内存地址区分的逻辑消失了。但是@Data生成的equals方式的逻辑缺无从得知。

可以看下下面的代码
这是一个用@Data定义的数据结构
```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class TestData {
    private String key;
    private List<TestData> value;
}
```
这是一个没有重写equals的数据结构
```java
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class BeanData {
    private String key;
    private List<BeanData> value;
}
```
如果用下面的代码来检测具体的执行行为，就会发现奇怪的逻辑。两次相同逻辑的HashMap的key值检查结果居然完全不同
```java
@Test
    public void checkLomokBehavior() {
        TestData dataA = new TestData();
        dataA.setKey("A");
        TestData dataA1 = new TestData();
        dataA1.setKey("A1");

        TestData dataARef = dataA;
        Map<TestData, String> dataMap = new HashMap<>();
        assertFalse(dataMap.containsKey(dataA));
        dataA.setValue(Arrays.asList(dataA1));
        dataMap.put(dataA, "1");
        assertTrue(dataMap.containsKey(dataARef));
        dataA1.setKey("A1-");
        assertFalse(dataMap.containsKey(dataARef)); \\这里会发现data的引用已经无法匹配到原先放进hashmap的对象了

        BeanData dataB = new BeanData();
        dataB.setKey("B");
        BeanData dataB1 = new BeanData();
        dataB1.setKey("B1");

        BeanData dataBRef = dataB;
        dataB.setValue(Arrays.asList(dataB1));
        Map<BeanData, String> dataBMap = new HashMap<>();
        dataBMap.put(dataB, "1");
        assertTrue(dataBMap.containsKey(dataBRef));
        dataA1.setKey("B1-");
        assertTrue(dataBMap.containsKey(dataBRef));//没有用@Data定义的数据没有这个问题，还是通过地址比对，数据值改变并不影响检查
    }
```
从上面看到，内部数据的值的改变会导致匹配存在差异其实这里还是有很多疑问的：
1. lombok的@Data生成的equals方式到底是什么逻辑？
2. 如果在自定义的数据结构中，使用了@Data注解，那么想切换到内存地址匹配的equals方式应如何设置呢？现在好像没有找到合适方法。
3. 这段逻辑中非常奇怪，即使@Data生成的equals方法使用的值匹配。但是在java中对象都是引用，那么在第一次放入hashmap中的key的对象的值是什么，难道是当时的对象值的一个快照？然后对象中的值改变了，理论上所有从引用获得值应该都发生了改变，那么在hashmap的key值比对过程中，哪个值是对象的某个快照而产生了不一致，是hashmap中存储值还是传入的对比值？
