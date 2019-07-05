---
layout:     post
title:      Shell十大坑
date:       2019-06-10
summary:    最容易犯的Shell编程错误
categories: 
---

开发过程常常需要编写一下shell脚本，常常被shell的语法特性气得呕血，有必要记下来讨伐一下shell。

真的很佩服那些能够写上千行shell的人。真心求教shell编程的快速入门方案，要快的那种，真心懒的了解这个怪胎。

1.  赋值“=”两边不能有空格
    你知道吗？下面的语句是非法的： 
```bash
HELLO = "world"
echo $HELLO
```
    执行这个脚本会得到这个错误：
```bash
line 1: HELLO: command not found 
```
    正确写法是：
```bash
HELL0="world"
echo $HELLO
```
    暂时还没有想到还有那些场景下，赋值的“=”前后是不能有空格的。这里受到100点伤害。

2.  谜一样的分割

    体验一下这段shell，语法都是搜索shell的数组定义找出来的。
```bash
a=( "Over Great Wall" "Hello world" )
for i in ${a[@]}
do
	echo $i
done
echo "-------------------"
for ((i=0;i<${#a[@]};i++))
do
	echo ${a[$i]}
done
```
    输出结果是这样的：
```bash
Over
Great
Wall
Hello
world
-------------------
Over Great Wall
Hello world
```
    这里就不说，想遍历数组中的元素用的是：**${a[@]}**这么个奇怪的符号。找了很多文档才了解**[@]**是游标的意思。

    然后也不说取数组的长度，居然用**${#a[@]}**这么个符号。只能说这个符号真的很长。

    可是为什么两种遍历结果的输出为什么不一样。字符串的按空格分割的逻辑到底发生在那里？？？？
    
    这样的逻辑在写程序时不出问题就见鬼了。



