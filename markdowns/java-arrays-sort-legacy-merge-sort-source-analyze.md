---
title: Arrays.sort遗留排序算法
date: 2017/8/21 14:09
---
LegacyMergeSort是JDK中一个遗留排序算法，在1.7+的JDK里，Arrays.sort(Object[])默认不使用legacyMergeSort算法，并且这个方法会在未来版本中删除。
虽然是老的排序算法，但还是有分析的价值。只有搞清楚新旧算法的差异，才能真正明白JDK为什么要废除这种算法。
<!-- more -->
LegacyMergeSort的核心方法是`java.util.Arrays#mergeSort(java.lang.Object[], java.lang.Object[], int, int, int)`。代码很简短，下面我们来一步步地分析这个方法。

### 启用LegacyMergeSort算法
如上节所述，遗留算法将在未来版本中删除，所以现在的JDK默认已经不再使用了。但是JDK给我们一个选择，看如下代码：
````java
public static void sort(Object[] a) {
    if (LegacyMergeSort.userRequested)
        legacyMergeSort(a);
    else
        ComparableTimSort.sort(a);
}
````
`LegacyMergeSort.userRequested`是一个布尔类型的常量，可以通过系统参数`-Djava.util.Arrays.useLegacyMergeSort=true`指定。

### 算法分析
mergeSort代码很少，理解起来并不难。主要使用的是二分插入排序，使用递归将长度大于等于7的数组分成前后两端，直至分成的短数组长度小于7之后，对短数组使用插入排序，最后再将排序后的前后两段合并成一个有序数组。

当数组长度小于7时，使用插入排序算法
````java
if (length < INSERTIONSORT_THRESHOLD) {
    for (int i=low; i<high; i++)
        for (int j=i; j>low &&
                 ((Comparable) dest[j-1]).compareTo(dest[j])>0; j--)
            swap(dest, j, j-1);
    return;
}
````

插入排序算法的过程总结为：遍历数组，当前下标的元素比前一个元素小，则交换这两个元素。继续往前遍历，并重复这个比较的过程，直到前一个元素比后一个元素小终止。

已数组 [10, 4, 11, 90, 3, 8] 为例，下图描述了整个插入排序的过程：

![插入排序示例](https://github.com/0x0010/notebook/raw/master/resources/Arrays-sort-legacy-merge-sort.png)

