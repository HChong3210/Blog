---
title: iOS开发总结系列-排序算法
date: 2018-04-09 21:41:23
tags:
    - 算法
categories:
    - 算法
---

这里主要是iOS开发中常见的排序算法. 

# 快排
原理: 从数列中挑出一个元素, 成为"基准", 重新排序数列, 所有元素比基准值小的摆放在基准前面, 所有元素比基准值大的摆在基准的后面(相同的数可以放在任一边), 在这个分区退出之后, 该基准就处于数列的中间位置, 递归的把小于基准值元素的子数列和大于基准值元素的子数列排序.

这种排序方法是基于二分的思想来实现的, 简单的说就是将一个大的问题通过不断的一分为二化简为较小的问题来解决. 通过二分化简了问题的规模来提高解决问题的效率. 因此我们将发现快速排序的平均时间复杂度是O(nlogn), 最差时间复杂度是O(n^2).

1. 定义两个哨兵i(数组的第一个位置), j(数组的最后一个位置), 一个基准数(数组的第一个元素). 
2. 从j开始, 向i比较, 直到遇到一个数小于基准数. 
3. 从i开始, 向j比较, 直到遇到一个数大于基准数.
4. 交换i, j两个位置的值.
5. 交换后, i, j继续按照之前的方式比较, 直到i与j相遇.
6. 我们交换基准数和哨兵位置上的数.
7. 接下来我们以这个哨兵位置为临界, 将原数组分为左右两部分, 再递归的调用上面的比对过程. 
8. 此时原数组的左边部分的哨兵i实际就是原i, 哨兵j实际就是i-1.
9. 此时原数组的右边部分的哨兵i实际就是原i+1, 哨兵j实际就是原j.

```
- (void)quickSortArray:(NSMutableArray *)array formLeft:(NSInteger)left right:(NSInteger)right {
    if (left >= right) {
        return;
    }
    NSInteger i = left;
    NSInteger j = right;
    //记录比较基准数
    NSInteger key = [array[i] integerValue];
    
    while (i < j) {
        //从右基准数开始向左递减, 直到一个数小于基准数(先从右边开始遍历是为了保持基准数不变)
        while (i < j && [array[j] integerValue] >= key) {
            j--;
        }
        //从左基准位置开始向右递增, 直到遇到一个数大于基准数
        while (i < j && [array[i] integerValue] <= key) {
            i++;
        }
        //交换两个数在数组中的位置
        if (i < j) {
            array[i] = @([array[i] integerValue] + [array[j] integerValue]);
            array[j] = @([array[i] integerValue] - [array[j] integerValue]);
            array[i] = @([array[i] integerValue] - [array[j] integerValue]);
        }
    }

    //此时, 两个基准数相等, 将基准数归位
    array[left] = array[i];
    array[i] = @(key);
    
    //递归处理基准数左边
    [self quickSortArray:array formLeft:left right:i - 1];
    //递归处理基准数右边
    [self quickSortArray:array formLeft:i + 1 right:right];
}
```

# 归并排序

归并排序是建立在归并操作上的一种有效的排序算法, 该算法是采用分治法(Divide and Conquer)的一个非常典型的应用. 他的时间复杂度是O(nlogn).

它的原理是假设初始序列含有n个记录, 则可以看成是n个有序的子序列, 每个子序列的长度为1, 然后两两归并, 得到n/2个长度为2或者1的有序子序列; 再两两归并, 如此反复, 直到得到一个长度为n的有序序列为止, 这种排序方法称为归并排序.
![归并排序](https://img-blog.csdn.net/20160908095240209)

```
- (void)mergeSortArray:(NSMutableArray *)array {
  //创建一个副本数组
  NSMutableArray * auxiliaryArray = [[NSMutableArray alloc]initWithCapacity:array.count];

  //对数组进行第一次二分，初始范围为0到array.count-1
  [self mergeSort:array auxiliary:auxiliaryArray low:0 high:array.count-1];
}

- (void)mergeSort:(NSMutableArray *)array auxiliary:(NSMutableArray *)auxiliaryArray low:(int)low high:(int)high {
  //递归跳出判断
  if (low>=high) {
    return;
  }
  //对分组进行二分
  int middle = (high - low)/2 + low;

  //对左侧的分组进行递归二分 low为第一个元素索引，middle为最后一个元素索引
  [self mergeSort:array auxiliary:auxiliaryArray low:low high:middle];

  //对右侧的分组进行递归二分 middle+1为第一个元素的索引，high为最后一个元素的索引
  [self mergeSort:array auxiliary:auxiliaryArray low:middle + 1 high:high];

  //对每个有序数组进行回归合并
  [self merge:array auxiliary:auxiliaryArray low:low middel:middle high:high];
}

- (void)merge:(NSMutableArray *)array auxiliary:(NSMutableArray *)auxiliaryArray low:(int)low middel:(int)middle high:(int)high {
  //将数组元素复制到副本
  for (int i=low; i<=high; i++) {
    auxiliaryArray[i] = array[i];
  }
  //左侧数组标记
  int leftIndex = low;
  //右侧数组标记
  int rightIndex = middle + 1;

  //比较完成后比较小的元素要放的位置标记
  int currentIndex = low;

  while (leftIndex <= middle && rightIndex <= high) {
    //此处是使用NSNumber进行的比较，你也可以转成NSInteger再比较
    if ([auxiliaryArray[leftIndex] compare:auxiliaryArray[rightIndex]]!=NSOrderedDescending) {
        //左侧标记的元素小于等于右侧标记的元素
        array[currentIndex] = auxiliaryArray[leftIndex];
        currentIndex++;
        leftIndex++;
    }else{
        //右侧标记的元素小于左侧标记的元素
        array[currentIndex] = auxiliaryArray[rightIndex];
        currentIndex++;
        rightIndex++;
    }
  }
  //如果完成后左侧数组有剩余
  if (leftIndex <= middle) {
     for (int i = 0; i<=middle - leftIndex; i++) {
        array[currentIndex +i] = auxiliaryArray[leftIndex +i ];
    }
  }
 }
```
----
参考资料:
1.[归并排序](https://www.jianshu.com/p/8fce5bfb0013)

