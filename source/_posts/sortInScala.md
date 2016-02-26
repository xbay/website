date: 2016-02-26 13:26:00
author: [hiddenlotus]
title: 基本排序算法的Scala实现
tags: [Scala, algorithm]
----
排序算法是算法的基础部分。  
本文讲用函数式的方法，用Scala实现4种常用的排序算法。  
由于是用函数式的方式，所以使用List代替Array。下文中统一由“列表”代替。

选择排序
-------
从一个列表中选出最小或最大的元素，插入到列表头或列表尾。  
循环一次，对列表中的每个元素都做同样的处理。  
平均时间复杂度为N的平方，N为列表的长度。
<img src="/images/selection.png" style="width: 250px" />
```scala
def selectionSort[T <% Ordered[T]](list: List[T]): List[T] = {
  def remove(e: T, list: List[T]): List[T] =
    list match {
      case Nil => Nil
      case x :: xs => if (x == e) xs else x :: remove(e, xs)
    }

  list match {
    case Nil => Nil
    case _ =>
      val min = list.min
      min :: selectionSort(remove(min, list))
  }
}
```

插入排序
-------
像平时我们打扑克时抓牌，并依次按照顺序插入到列表中相应的位置。  
平均时间复杂度为N的平方。
<img src="/images/insertion.png" style="width: 250px" />
```scala
def insertionSort[T <% Ordered[T]](list: List[T]): List[T] = {
  def insert(e: T, list: List[T]): List[T] =
    list match {
      case Nil => List(e)
      case x :: xs => if (x < e) x :: insert(e, xs) else e :: list
    }
  list.foldLeft(List.empty[T])((res, init) => insert(init, res))
}
```

归并排序
-------
递归的将列表分成长度更小的列表。当每个子列表的长度小于等于1时，将列表们归并在一起。  
平均时间复杂度为NlogN。
<img src="/images/merge.png" style="width: 250px" />
```scala
def mergeSort[T <% Ordered[T]](list: List[T]): List[T] = {
  def merge(l1: List[T], l2: List[T]): List[T] =
    (l1, l2) match {
      case (Nil, l) => l
      case (l, Nil) => l
      case (x :: xs, y :: ys) =>
        if (x < y) x :: merge(xs, l2)
        else y :: merge(l1, ys)
    }
  val n = list.length / 2
  if (n == 0) list
  else {
    val (ys, zs) = list.splitAt(n)
    merge(mergeSort(ys), mergeSort(zs))
  }
}
```

快速排序
-------
从一个列表中随机选出一个元素作为基准，分别在过滤出小于它的元素，大于等于它的元素，  
递归的对这些列表调用该算法，最后将列表合并。  
这里并没有随机从列表中取元素，只取第一个元素作为基准元素。  
平均时间复杂度为NlogN。
<img src="/images/quick.png" style="width: 250px" />
```scala
def quickSort[T <% Ordered[T]](list: List[T]): List[T] =
  list match {
    case Nil => Nil
    case x :: _ =>
      val (smaller, eqs, bigger) = list.foldLeft((List.empty[T], List.empty[T], List.empty[T])) {
        (res, init) =>
        if (init < x) (init :: res._1, res._2, res._3)
        else if (init == x) (res._1, init :: res._2, res._3)
        else (res._1, res._2, init :: res._3)
      }
      quickSort(smaller) ::: eqs ::: quickSort(bigger)
  }
```

注1：图片来源于网络
注2：```T <% Ordered[T]```为View Bound，表示“I can use any T, so long as T can be treated as an Ordered[T].”
