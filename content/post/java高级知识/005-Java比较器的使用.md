---
title: "Java比较器的使用"
date: 2020-03-02T00:29:44+08:00
draft: false
Categories: ["java"]
Tags: ["java", "比较器"]
---

在java中实现对象排序的方式有两种：

- 自然排序：java.lang.Comparable
- 定制排序：java.util.Comparator

## 1. 自然排序：java.lang.Comparable

Comparable是直接加在类上面的，需要实现Comparable接口，并且重写compareTo(Object obj)方法，规则是：

- 如果this>obj,则返回正整数；
- 如果this<obj,则返回负整数；
- 如果this=obj，则返回0.

实现Comparable接口的对象列表（和数组）可以通过 Collections.sort 或 Arrays.sort进行自动排序。实现此接口的对象可以用作有序映射中的键或有 序集合中的元素，无需指定比较器。

```java
import java.util.Arrays;

class Goods implements Comparable<Goods> {
    private String name;
    private double price;

    @Override
    public int compareTo(Goods o) {
        if (this.price > o.price) {
            return 1;
        } else if (this.price < o.price) {
            return -1;
        }
        return 0;
    }
   //构造器、getter、setter、toString()方法略
}


public class ComparableTest {
    public static void main(String[] args) {
        Goods[] goods = new Goods[2];
        goods[0] = new Goods("钢笔", 13.5);
        goods[1] = new Goods("铅笔", 2.5);
        Arrays.sort(goods);
        // [Goods{name='铅笔', price=2.5}, Goods{name='钢笔', price=13.5}]
        System.out.println(Arrays.toString(goods));
    }
}
```



## 2. 定制排序：java.util.Comparator

当元素的类型没有实现java.lang.Comparable接口而又不方便修改代码， 或者实现了java.lang.Comparable接口的排序规则不适合当前的操作，那么可以考虑使用 Comparator 的对象来排序，强行对多个对象进行整体排 序的比较。

重写compare(Object o1, Object o2)方法，比较o1和o2的大小：

- 如果o1>o2,则返回正整数；
- 如果o1<o2，则返回负整数；
- 如果o1=o2，则返回0。

可以将 Comparator 传递给 sort 方法（如 Collections.sort 或 Arrays.sort）， 从而允许在排序顺序上实现精确控制。

```java
Goods[] goods = new Goods[4];
goods[0] = new Goods("钢笔", 13.5);
goods[1] = new Goods("铅笔", 2.5);
goods[2] = new Goods("铅笔B", 1.5);
goods[3] = new Goods("毛笔", 8);
//创建Comparator的匿名实例
Arrays.sort(goods, new Comparator<Goods>() {
    @Override
    public int compare(Goods o1, Goods o2) {
        return -Double.compare(o1.getPrice(), o2.getPrice());
    }
});
//[Goods{name='钢笔', price=13.5}, Goods{name='毛笔', price=8.0}, Goods{name='铅笔', price=2.5}, Goods{name='铅笔B', price=1.5}]
System.out.println(Arrays.toString(goods));
```

