---
title: 从Smali代码的角度看foreach和dowhile的区别
date: 2017/8/16 14:09
---

最近被问到一个问题：foreach和dowhile的区别（编译之后）。刚拿到这个问题时，一时真不知道怎么作答。不过他们的用法和共同点对于程序员来说都不陌生。那么jdk是如何编译foreach代码的呢？我们准备将Java代码编译成[smali]("https://github.com/JesusFreke/smali")代码来一探究竟。

<!-- more -->

### foreach代码解读

先写一段foreach代码

````java
public static void main(String[] args) {
  char[] chars = {'h', 'a', 'y'};
  for (char ch : chars) {
    System.out.println(ch);
  }
}
````

代码很简单，不做过多解释了。下面我们将以上代码编译成smali，看看编译后的foreach长啥样。

代码部分添加了注释

````assembly
.method public static main([Ljava/lang/String;)V
    .registers 6
    .param p0, "args"    # [Ljava/lang/String;

    .prologue
    .line 12
    const/4 v2, 0x3  #定义变量v2=3

    new-array v1, v2, [C  #构造长度为3的char数组

    fill-array-data v1, :array_16 #使用array16填充数据，array16在方法尾部。

    .line 13
    .local v1, "chars":[C
    array-length v3, v1  #求字符数组v1长度，赋值v3=3

    const/4 v2, 0x0  #循环变量v2 初始值为0

    :goto_8  #循环开始
    if-ge v2, v3, :cond_14  #如果循环变量大于等于3 跳到cond_14, 执行结束

	# 如下代码是打印char[v2]字符
    aget-char v0, v1, v2

    .line 14
    .local v0, "ch":C
    sget-object v4, Ljava/lang/System;->out:Ljava/io/PrintStream;

    invoke-virtual {v4, v0}, Ljava/io/PrintStream;->println(C)V
    # 打印字符结束
    .line 13
    add-int/lit8 v2, v2, 0x1 #循环变量v2=v2+1

    goto :goto_8 #进入下一次循环

    .line 16
    .end local v0    # "ch":C
    :cond_14
    return-void

    .line 12
    nop

    :array_16
    .array-data 2
        0x68s
        0x61s
        0x79s
    .end array-data
.end method
````

从以上smali代码能明显感觉到，这很像while语法，翻译成while语法看看：

````java
public static void main(String[] args) {
  char[] chars = {'h', 'a', 'y'};
  int v2 = 0, v3 = chars.length;
  while (v2 < v3) {
    System.out.println(chars[v2]);
    v2++;
  }
}
````

这是很常见的while写法。

### foreach与dowhile的区别

dowhile的写法不用多说了，所以这个问题就转化成while和dowhile的区别。

Oracle的j2se文档是这么总结的：

*dowhile与while的区别是，dowhile在循环的尾部计算表达式。因此do代码块总会执行至少一次。*

>The difference between `do-while` and `while` is that `do-while` evaluates its expression at the bottom of the loop instead of the top. Therefore, the statements within the `do` block are always executed at least once, as shown in the following [`DoWhileDemo`](https://docs.oracle.com/javase/tutorial/java/nutsandbolts/examples/DoWhileDemo.java) program:

````java
class DoWhileDemo {
    public static void main(String[] args){
        int count = 1;
        do {
            System.out.println("Count is: " + count);
            count++;
        } while (count < 11);
    }
}
````



#### 参考资料

1. Oracle Java Document  [The while and do-while Statements](https://docs.oracle.com/javase/tutorial/java/nutsandbolts/while.html) 
2. smali项目地址 [https://github.com/JesusFreke/smali](https://github.com/JesusFreke/smali)
3. Intellij Idea 安装使用smali [https://github.com/JesusFreke/smali/wiki/smalidea](https://github.com/JesusFreke/smali/wiki/smalidea)