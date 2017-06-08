---
title: 简单选择排序算法
date: 2016/6/18 14:52:31
---

简单选择排序算法（Simple Selection Sort)的排序思想是，在要排序的一组数中，选出最小（或者最大）的一个数与第1个位置的数交换；然后在剩下的数当中再找最小（或者最大）的与第2个位置的数交换，依次类推，直到第n-1个元素（倒数第二个数）和第n个元素（最后一个数）比较为止。

<!-- more -->

简单选择排序示例：

![Simple Selection Srot Sample Image](http://my.csdn.net/uploads/201207/18/1342586432_7130.jpg)

### 代码实现

```java
    int[] unsorted = {10, 8, 12, 4, 3, 0, 11, 8, 7, 40, 43};

    /**
     * 简单选择排序
     *
     * @param unsorted 无序表
     * @param sortType 排序类型 1-asc 2-desc
     * @return 有序表
     */
    int[] sort(final int[] unsorted, int sortType) {
        // 拷贝一份无序表，不对原数据做操作
        int[] o = Arrays.copyOf(unsorted, unsorted.length);
        for (int i = 0; i < o.length - 1; i++) {
            // 使用临时变量exchangeValue保存“极端元素”，极端元素表示目前为止最大或者最小。
            // 使用exchangeIndex存储待交换元素的下标。
            int a = o[i], exchangeValue = a, exchangeIndex = -1;
            for (int j = i + 1; j < o.length; j++) {

                if (sortType == 1) { // 升序 ASC，寻找尾表中最小元素
                    if (o[j] < exchangeValue) {
                        exchangeIndex = j;
                        exchangeValue = o[j];
                    }
                } else if (sortType == 2) { // 降序，寻找尾表中最大元素
                    if (o[j] > exchangeValue) {
                        exchangeIndex = j;
                        exchangeValue = o[j];
                    }
                }
            }
            // 交换元素
            if (exchangeIndex > -1) {
                o[i] = o[exchangeIndex];
                o[exchangeIndex] = a;
            }
        }
        return o;
    }
```

测试程序代码
```
    public static void main(String[] args) {
        SimpleSelectionSort simpleSelectionSort = new SimpleSelectionSort();
        System.out.println("Original array:" + Arrays.toString(simpleSelectionSort.unsorted));
        System.out.println("Asc-sorted array:" + Arrays.toString(simpleSelectionSort.sort(simpleSelectionSort.unsorted, 1)));
        System.out.println("Desc-sorted array:" + Arrays.toString(simpleSelectionSort.sort(simpleSelectionSort.unsorted, 2)));
    }
}
```
测试结果
```
Original array:[10, 8, 12, 4, 3, 0, 11, 8, 7, 40, 43]
Asc-sorted array:[0, 3, 4, 7, 8, 8, 10, 11, 12, 40, 43]
Desc-sorted array:[43, 40, 12, 11, 10, 8, 8, 7, 4, 3, 0]
```
