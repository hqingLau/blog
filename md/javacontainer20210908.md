<style type="text/css"> img{display:block;margin:0 auto;}</style>

Java 集合， 也叫作容器，主要是由两大接口派生而来：一个是 `Collecton`接口，主要用于存放单一元素；另一个是 `Map` 接口，主要用于存放键值对。对于`Collection` 接口，下面又有三个主要的子接口：`List`、`Set` 和 `Queue`。

![image-20210908160734030](https://gitee.com/hqinglau/img/raw/master/img/20210908160734.png)

![image-20210908160806337](https://gitee.com/hqinglau/img/raw/master/img/20210908160806.png)

`Deque` 还提供有 `push()` 和 `pop()` 等其他方法，可用于模拟栈。

**List**：

`Arraylist`： `Object[]` 数组，`List`的主要实现类，适用于频繁查找，线程不安全

`Vector`：`Object[]` 数组，线程安全

`LinkedList`： 双向链表(JDK1.6 之前为循环链表，JDK1.7 取消了循环)，不保证线程安全（指定位置`i`删除的话，复杂度也为O(n)，因为要先移动过去）

**Set**：

不可重复性：添加的元素按照 `equals()`判断时 ，返回 false，需要同时重写 `equals()`方法和 `HashCode()`方法。

`equals()` 方法被覆盖过，则 `hashCode()` 方法也必须被覆盖

`HashSet`(无序，唯一): 基于 `HashMap` 实现的，底层采用 `HashMap` 来保存元素，线程不安全

```java
HashSet<Integer> set = new HashSet<>();
set.add(1);
// 源码
private static final Object PRESENT = new Object();
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

`LinkedHashSet`: `LinkedHashSet` 是 `HashSet` 的子类，并且其内部是通过 `LinkedHashMap` 来实现的。有点类似于我们之前说的 `LinkedHashMap` 其内部是基于 `HashMap` 实现一样，不过还是有一点点区别的

`TreeSet`(有序，唯一): 红黑树(自平衡的排序二叉树)

**Queue**：

`PriorityQueue`: `Object[]` 数组来实现二叉堆，通过堆元素的上浮和下沉，实现了在 O(logn) 的时间复杂度内插入元素和删除堆顶元素，非线程安全。**典型例题包括堆排序、求第K大的数、带权图的遍历等**

`ArrayDeque`：`object[]` 数组 + 双指针，不支持`NULL`数据，插入可能要扩容，性能来看比`LinkedList`更好

**Map**：

`HashMap`： JDK1.8 之前 `HashMap` 由数组+链表组成的，数组是 `HashMap` 的主体，链表则是主要为了解决哈希冲突而存在的（“拉链法”解决冲突）。JDK1.8 以后在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为 8）（将链表转换成红黑树前会判断，如果当前数组的长度小于 64，那么会选择先进行数组扩容，而不是转换为红黑树）时，将链表转化为红黑树，以减少搜索时间。非线程安全，保证线程安全用`ConcurrentHashMap`，**并发控制使用 `synchronized` 和 CAS 来操作**，`synchronized` 只锁定当前链表或红黑二叉树的首节点，这样只要 hash 不冲突，就不会产生并发，效率又提升 N 倍。。

<img src="https://gitee.com/hqinglau/img/raw/master/img/20210908171950.png" alt="image-20210908171949920" style="zoom: 60%;" />

`LinkedHashMap`： `LinkedHashMap` 继承自 `HashMap`，所以它的底层仍然是基于拉链式散列结构即由数组和链表或红黑树组成。另外，`LinkedHashMap` 在上面结构的基础上，增加了一条双向链表，使得上面的结构可以保持键值对的插入顺序。同时通过对链表进行相应的操作，实现了访问顺序相关逻辑

~~`Hashtable`： 数组+链表组成的，数组是 `Hashtable` 的主体，链表则是主要为了解决哈希冲突而存在的，全局锁~~

`TreeMap`： 红黑树（自平衡的排序二叉树）

**comparable** 和 Comparator 的区别：

`Comparable` 接口实际上是出自`java.lang`包 它有一个 `compareTo(Object obj)`方法用来排序

`Comparator`接口实际上是出自 java.util 包它有一个`compare(Object obj1, Object obj2)`方法用来排序，只能使用两个参数版的 `Collections.sort()`

```java
ArrayList<Integer> arrayList = new ArrayList<>();
arrayList.add(3);
arrayList.add(5);
arrayList.add(1);
System.out.println(arrayList); // [3, 5, 1]
Collections.sort(arrayList);
System.out.println(arrayList); // [1, 3, 5]
Collections.sort(arrayList, new Comparator<Integer>() {
    @Override
    public int compare(Integer o1, Integer o2) {
        return o2.compareTo(o1);
    }
});
System.out.println(arrayList); // [5, 3, 1]
```

`Comparable` 接口例子：

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.Comparator;
import java.util.TreeMap;

class Person implements Comparable<Person>{
    private String name;
    private int age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public int compareTo(Person o) {
        if(this.age>o.age) {
            return 1;
        }
        if(this.age<o.age) {
            return -1;
        }
        return 0;
    }

    public Person(String name, int age) {
        Super();
        this.name = name;
        this.age = age;
    }
}

public class Container {
    public static void main(String[] args) {
        TreeMap<Person,String> pdata = new TreeMap<>();
        pdata.put(new Person("张三",11),"zhangsan");
        pdata.put(new Person("李四",8),"lisi");
        for (Person key:pdata.keySet()) {
            System.out.println(key.getAge() + "-" + key.getName());
        }
    }
}
// output:
// 8-李四
// 11-张三
```

`Collections`工具类：

```java
void reverse(List list)//反转
void shuffle(List list)//随机排序
void sort(List list)//按自然排序的升序排序
void sort(List list, Comparator c)//定制排序，由Comparator控制排序逻辑
void swap(List list, int i , int j)//交换两个索引位置的元素
void rotate(List list, int distance)//旋转。当distance为正数时，将list后distance个元素整体移到前面。当distance为负数时，将 list的前distance个元素整体移到后面
    
int binarySearch(List list, Object key)//对List进行二分查找，返回索引，注意List必须是有序的
int max(Collection coll)//根据元素的自然顺序，返回最大的元素。 类比int min(Collection coll)
int max(Collection coll, Comparator c)//根据定制排序，返回最大元素，排序规则由Comparatator类控制。类比int min(Collection coll, Comparator c)
void fill(List list, Object obj)//用指定的元素代替指定list中的所有元素
int frequency(Collection c, Object o)//统计元素出现次数
int indexOfSubList(List list, List target)//统计target在list中第一次出现的索引，找不到则返回-1，类比int lastIndexOfSubList(List source, list target)
boolean replaceAll(List list, Object oldVal, Object newVal)//用新元素替换旧元素
```



**参考**：

[JavaGuide](https://snailclimb.gitee.io/javaguide/#/)