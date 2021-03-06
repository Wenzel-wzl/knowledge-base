# 常见排序算法

排序分内排序和外排序。

外排序指在排序期间全部对象个数太多,不能同时存放在内存,必须根据排序过程的要求,不断在内、外存之间移动的排序。

内排序指在排序期间数据对象全部存放在内存的排序。

内排序的方法有许多种,按所用策略不同,可归纳为六类:插入排序、选择排序、交换排序、归并排序、分配排序和计数排序。

+ ### 插入排序
  
  - #### 折半插入排序
  
  - #### 希尔排序
  
+ ### 选择排序
  
  - #### 直接选择排序
  
  - #### 堆排序
  
+ ### 交换排序

  - #### 冒泡排序
  
  - #### 快速排序

+ ### 归并排序

  - #### 二路归并(常用的归并排序)
  
  - #### 自然归并
  
+ ### 分配排序
  
  - #### 箱排序
  
  - #### 基数排序
  
- ### 计数排序


## 排序稳定性

假设在待排序的文件中,存在两个或两个以上的记录具有相同的关键字,在用某种排序法排序后,若这些相同关键字的元素的相对次序仍然不变,则这种排序方法是稳定的。

稳定排序有[冒泡排序](#冒泡排序)，[插入排序](#插入排序)，[基数排序](#基数排序)，[归并排序](#归并排序)；

不稳定排序有[选择排序](#选择排序)，[快速排序](#快速排序)，[希尔排序](#希尔排序)，[堆排序](#堆排序)。 

# 参考
  * [常见排序算法](http://www.cnblogs.com/xkfz007/archive/2012/07/01/2572017.html)
