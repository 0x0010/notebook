### 什么是LegacyMergeSort
LegacyMergeSort是JDK中一个遗留排序算法，在1.7+的JDK里，Arrays.sort(Object[])默认不使用legacyMergeSort算法，并且这个方法会在未来版本中删除。
虽然是老的排序算法，但还是有分析的价值。只有搞清楚新旧算法的差异，才能真正明白JDK为什么要废除这种算法。

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
`LegacyMergeSort.userRequested`是一个
