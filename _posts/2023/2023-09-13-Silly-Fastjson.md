---
layout:     post
title:      愚蠢到不能再愚蠢的Fastjson
date:       2023-09-13
summary:    选择用Fastjson做序列化真实愚蠢到家了
categories: java
---
最近接手了一个项目，项目默认使用的fastjson作为为json的序列化工具。

之前了解Fastjson用的不算多，唯一的认识就是自认为比Gson序列化快的一个工具，其宣传词就是Gson就是龟son，嘲弄其速度慢。
本想这个项目也没有特别的需求，json序列化的性能也不是主要的瓶颈，也就没太在意。

结果在项目真实使用的时候，真是遇到大坑了。由于需要跨不同子系统调用，发现json字符串的格式总是不对，还经常出现奇怪的字符串。
上下游总是对不起来。搞的很苦恼，结果最后定位发现原来是Fastjson的问题，它居然在JSON序列化的时候自作多情的加
个自己的逻辑。这个神操作也是醉了，话说你加就加吧，居然把json的数据格式给弄坏了。

后来上网查相关文档，加上测试了一下行为，发现Fastjson简直就是作死，要用它来做序列化工具简直就是脑残行为呀。

不说了，先看例子：
```java
@Data
public class OuterModel implements Serializable {
    @ApiModelProperty(value = "元数据对象")
    @JsonProperty(value = "InnerModelList")
    @JSONField(name = "InnerModelList")
    private List<InnerModel> InnerModelList;
}

@Data
public class InnerModel {
    @JSONField(name = "Values")
    private List<String> values;
}
```
这是两个数据格式定义
```java
@Test
public void testJson(){
        OuterModel model=new OuterModel();
        model.setInnerModelList(new ArrayList<>());
        System.out.println(JSON.toJSONString(model));
}
```
尝试输出一下，你猜FastJson 序列化输出一下，看看输出的是什么：
```text
{"InnerModelList":[]{"$ref":"$.InnerModelList"}}
```
惊呆了，是了解Fastjson有一套自己的什么引用的系统，但是这个破坏json结构也是太暴力了。话说出现什么内部引用，循环引用报错了就好了，来这么一出，序列化的结果
要么不可用还不知道错在哪。

当然最后发现可能是OuterModel里面的属性名字第一个字母大写了，但这并不是错，而且用jackson也是没有问题的，有问题的就是fastjson太自以为是了。