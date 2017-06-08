---
title: 特定场景下String#equalsIgnoreCase的替代方案
date: 2017/6/8 11:28
---

先说说我遇到的场景：由于Oracle的JDBC驱动无法通过数据源参数来开关StatementCache功能，所以在设计连接池时需自己实现这个开关功能。我的做法是开放可配置的系统参数，用户根据需要自行设置。

举个栗子：

````properties
# 开启Oracle连接的语句缓存
datasource.enableStatementCache=true
# 设置Oracle连接缓存语句的个数
datasource.statementCacheSize=500
````
<!-- more -->
以上就是场景中描述的开关配置。

在我定义*enableStatementCache*参数值时遇到了问题，纠结于以下两种选择：

- **true/false** 

  最先想到的就是布尔值，这似乎也更符合人性化设计。但是Properties文件是以字符串的形式加载到内存里的，在使用时需要进行字符串比较。后续会说到字符串比较的细节。

- **1/0**

  使用1和0作为开关值并不少见，并且转换成数字在内存中比较的效率也不低。

最终选择第一种true/false的方案，选择依据其实很简单，觉得1/0与*enableStatementCache*参数名有点违和，true/false显得人性化，说明我们的产品是有生命，有温度的。

接下来，我们需要做的事情是判断*enableStatementCache*的值是true还是false。实现这个判断的做法很多，市面上两种常用的做法是字符串比较和转换成布尔值再比较。

### 字符串比较

常用的比较字符串的API有两个，它们的差异在于是否对大小写敏感。

````java
// 大小写敏感
java.lang.String#equals
// 大小写不敏感
java.lang.String#equalsIgnoreCase
````

实际使用时用户不太可能严格遵守大小写的约定，所以使用对大小写不敏感的API。代码片段如下：

````java
private static void jdkEqualsIgnoreCase() {
  // true --> expected value
  // True --> user input
  if ("true".equalsIgnoreCase("True")) {
  }
}
````

执行1亿次字符串直接比较的耗时平均在2000毫秒左右：

````
Jdk.equalsIgnoreCase Costs:2644
Jdk.equalsIgnoreCase Costs:2013
Jdk.equalsIgnoreCase Costs:1991
Jdk.equalsIgnoreCase Costs:1993
Jdk.equalsIgnoreCase Costs:2005
````

### 转换成布尔值再比较

转换成布尔值再进行比较比较简单。

````java
private static void boolEquals() {
  // True --> user input
  if (Boolean.valueOf("True") == Boolean.TRUE) {
  }
}
````

其实*java.lang.Boolean#valueOf(java.lang.String)*也是通过*java.lang.String#equalsIgnoreCase*实现的。

执行1亿次转换成布尔值再比较平均耗时在2000毫秒左右。

````
Boolean.equals Costs:2865
Boolean.equals Costs:1978
Boolean.equals Costs:1998
Boolean.equals Costs:2004
Boolean.equals Costs:2023
````

### 优化比较算法一

分析一下这个比较的场景，如果用户想开启cache功能，那么需要我们考虑的入参就是{[T,t], [R,r], [U,u], [E,e]}四组字符的组合，一共16种可能性。只要入参是这16种组合里的任意一个，就认为入参是true。

那么这16种组合是什么呢？我们使用代码来快速穷举。

````java
private static Map<String, Integer> boolBag;
static {
  boolBag = new HashMap<>();
  char[] 
      t = {'t', 'T'}, 
      r = {'r', 'R'}, 
      u = {'u', 'U'}, 
      e = {'e', 'E'};
  
  for (int i = 0; i < 16; i++) {
    boolBag.put("" 
        + t[(i & 0b1000) >> 3] 
        + r[(i & 0b0100) >> 2] 
        + u[(i & 0b0010) >> 1] 
        + e[i & 0b0001], 1);
  }
}
````

最终*boolBag*的key就是穷举出来的16种组合。

````
{True=1, TruE=1, TrUe=1, TrUE=1, TRue=1, TRuE=1, TRUe=1, TRUE=1, true=1, truE=1, trUe=1, trUE=1, tRue=1, tRuE=1, tRUe=1, tRUE=1}
````

只要输入在这16种组合中就认为是真值。

使用Map.containsKey(Object)判断真值的执行效率如何呢？看下边的测试结果：

````
Hash key Costs:680
Hash key Costs:681
Hash key Costs:682
Hash key Costs:734
Hash key Costs:688
````

从测试结果来看，效率提升了3-4倍，提升效果还是比较可观的。如果按照现有思路，效率方面还有没有继续提升的空间呢？

今天先写到这里，后续应该还有两种算法来提升这种特定场景的比较效率问题。







