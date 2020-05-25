---
layout:     post
title:      常见的Shell大坑
date:       2019-06-10
summary:    最容易犯的Shell编程错误
categories: 
---

开发过程常常需要编写一下shell脚本，常常被shell的语法特性气得呕血，有必要记下来讨伐一下shell。

真的很佩服那些能够写上千行shell的人。真心求教shell编程的快速入门方案，要快的那种，真心懒的了解这个怪胎。

##  赋值“=”两边不能有空格
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
    暂时还没有想到为什么赋值的“=”前后是不能有空格的。这里受到100点伤害。

##  谜一样的分割

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
    
##  写一个正确的带逻辑运算符的判断条件吧
    
    假设条件是这样的：已当前的文件名的文件存在且文件名以“aaa”开头，或者文件名以“bbb”结尾并且文件名不能是“aaabbb”。
    
    想当然的写法是这样：
    
```bash
# 这是错的
[ -f ${filname} ] && （ [ ${filename} =~ "aaa.*" ] || [ ${filename} =~ ".*bbb"] ） && [ ${filename} != "aaabbb" ]

```
   搜索半天后，不断试错，终于修复完所有语法错误后，写出来是这样的：
   
```bash
# 这是错的
[ -f ${filname} ] && [ ${filename} =~ aaa* ] || [ ${filename} =~ .*bbb ] && [ ${filename} != "aaabbb" ]

```
   充分理解语法，然后想一想后：
```bash
# 这是对的
[ -f ${filname} ] && [ ${filename} =~ aaa* || ${filename} =~ .*bbb ] && [ ${filename} != "aaabbb" ]

```
   当然千万不要告诉我这里还可以有[[]], (()), -a -o这些用法，已经迷糊了。别让我写判断条件了好吗
    
##  我想是数字

    想象一下，shell里面类似i++的功能应该怎么实现的。
    
    搜索一番后，发现不支持++运算符，但是发现有+=这个运算符，那试一下:
    
```bash
a=1
a+=2
echo ${a}

```
   得到输出居然是“12”,没错。不是“3”,也不是"1“，也不是”21“。
   
   改变一个数值，然后重新赋值需要采用下面的方案。
```bash
a=`expr $a + 1`  #注意加号周围的空格
a=$[a+1]
a=$((a+1))

# 但直接的数值是可以运算的，但是要注意乘法
a=1+1    # a=2
a=1\*3   # a=3, 乘法需要用转义符号
a=2; a=$[a*3] # a=6, 这里乘法是不需要转义符号的......   
```

   了解了规则之后，再来看一段神奇代码：
```bash
a=1
a+=2
a=$[a+1]
echo ${a}

```
   输出是"13”，没错就是13,程序不会报错，可以成功运行。
   
   别跟我说shell是一种编程语言。
    
##  弄不清楚的环境变量
    shell中的变量还是很简单的，只有两种:
    * 全局变量。默认的都是全局变量，可以在程序的所有地方访问
    * 函数本地变量，在变量前加local修饰，在函数体内定义，只能在函数体内访问
```bash
function verify_var {
local var_a="var_a"
var_b="var_b"
}
echo ${var_a}_${var_b}   # print _var_b
```
   bash中复杂的环境变量主要出现在各种bash脚本相互调用，及export，source，sudo等各种命令切换导致的环境变量一团糟。
```bash
# main.sh
foo="Global_Foo"
bash sub1.sh
echo "main.sh:${foo}"
----------
# sub1.sh
echo "sub1.sh:${foo}"
```
   上面的脚本，在main.sh里面定义了一个变量foo，在sub1.sh中调用了一次。发现在sub1.sh中该变量不存在
```bash
bash main.sh
# print
sub1.sh
mains.sh:Global_Foo
```

   原因是在shell中一个全局变量只定义在当前shell进程中，如果另外一个shell是无法访问到的。即使另外一个shell进程是当前进程的子进程，也无法访问。
   当然，如果要访问这个变量的话，也有办法，就是加上export命令，这样这个shell的子进程就可以访问了。当然是这个shell的子进程。如果不是子进程也是不能访问的。
```bash
# use export
export foo="Global_Foo"

# bash main.sh output
sub1.sh:Global_Foo
mains.sh:Global_Foo
```

   通过export的形式，虽然子进程可以访问环境变量，但是父进程和子进程使用的环境变量并不是同一个对象。子进程设置的变量并不会改变父进程的值。
```bash
# main.sh
export foo="Global_Foo"
bash sub1.sh
echo "main.sh:${foo}"
----------
# sub1.sh
export foo="local_Foo"
echo "sub1.sh:${foo}"

# bash main.sh output
sub1.sh:local_Foo
main.sh:Global_Foo
```  

   通过export定义的变量会是在当前进程。例如当前的用户会话下通过export定义变量，那么在当前用户执行多个bash脚本那么都可以访问这个变量。因为这些脚本都是当前shell下的子进程。同时如果在开一个回会话，用户重新登陆或者切换会话，则当前的变量就不生效了。为了解决这个问题，一般都在。~/.bash.rc文件中添加变量的定义，这项在用户会话初始化时自动完成加载。
   那么当然，如果在shell脚本中使用了类似sudo的命令，那么当前export的变量就又不生效了。当然可以有解决方案。就是在sudo后面加“-E”参数，这样回把当前环境变量加载到sudo。
   shell脚本的环境变量用起来非常酸爽。如果跑一个脚本发现环境变量不对，千万不要奇怪。
   
