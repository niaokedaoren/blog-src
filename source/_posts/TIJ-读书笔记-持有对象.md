title: TIJ 读书笔记 -- 持有对象
date: 2015-09-28 22:58:47
categories: java
tags: 容器,Collections,sort,Comparable,Comparator
---

### Overview
![Collections UML][overview-uml]
这一章主要讲java容器类，以及相关类的一些基本用法，看完之后对这些java容器之间关系有了一定的把握。总的来说，java容器分为两大类：
* Collection
该接口定义了线性容器，List、Set、Queue都是继承自它。
* Map
该接口定义了key-value对的容器。

同时Collections类里面定义了一堆静态函数，提供辅助功能，方便用户操作容器类，例如sort、二分查找、shuffle等功能。

Java容器类最大的特点就是接口定义和具体实现是解耦的，这个很方便第三方提供其它的优化实现，来替代java sdk中的实现，例如著名的guava库。

看书，除了温故，当然还有知新。我不喜欢泛泛总结，下面还是举些例子。
### Comparable vs Comparator
以前也看过，但是老是映像不深，总觉的差不多，因为两个都是接口。现在谈谈本次看书的体会：
* Comparable
需要实现compareTo方法，这个接口像是一个特质（用scala里的trait解释，会更好），一个类实现了这个接口，就说明这个类的对象具有可比性这个特质，这样在调用Collections的sort方法时，就不需要额外提供其它的比较器。
* Comparator
需要实现compare方法，这个接口更像是一种策略模式，它是独立于需要比较的类；不同的实现可以给同一个类提供不同的比较策略，使用起来会更加灵活。

```java
class User implements Comparable<User> {
    final int id;
    final String name;
    
    User(int id, String name) {
        this.id = id;
        this.name = name;
    }

    public int compareTo(User o) {
        if (o == null) {
            return -1;
        }
        
        return this.id - o.id;
    }
    
    public String toString() {
        return "(" + id + ", " + name + ")";
    }
}
```
现在有User类，实现了Comparable接口，所以具有User类对象之间具有可比性，现在的实现是按照id来排序的。

测试代码
```java
public class ComparableVsComparator {
    
    private static List<User> generateTestData() {
        List<User> users = new ArrayList<User>();
        users.add(new User(2, "Jack"));
        users.add(new User(1, "Tom"));
        users.add(new User(3, "Lucy"));
        return users;
    }
    
    private static void testComparable() {
        List<User> users = ComparableVsComparator.generateTestData();
        Collections.sort(users);
        System.out.println(users);
    }

    public static void main(String[] args) {
        System.out.println("Test comparable: ");
        ComparableVsComparator.testComparable();
    }
/* Output
 Test comparable: 
 [(1, Tom), (2, Jack), (3, Lucy)]
 */
}
```
输出结果是按照user id来升序排列的，一切都显得很顺利，但是突然有需求，需要对user name进行字母序排序，这下Comparator可以拿出来一展身手了，原来的代码完全不用修改,只需要另外提供一个字母序比较的Comparator实现。
```java
class UserComparatorViaName implements Comparator<User> {
    public int compare(User o1, User o2) {
        return o1.name.compareTo(o2.name);
    }
}
```

测试代码
```java
...
private static void testComparator() {
    List<User> users = ComparableVsComparator.generateTestData();
    Collections.sort(users, new UserComparatorViaName());
    System.out.println(users);
}

public static void main(String[] args) {
    System.out.println("Test comparator: ");
    ComparableVsComparator.testComparator();
}
/* Output
 Test comparator: 
 [(2, Jack), (3, Lucy), (1, Tom)]
 */
```

### Queue and Stack
因为List下的ArrayList、LinkedList，以及Set下的HashSet、Map下的HashMap平时都比较用的多，就不举例子了。下面就着重介绍一下Queue和Stack，官方API推荐使用双端队列接口[Deque](http://docs.oracle.com/javase/7/docs/api/java/util/Deque.html)的一种实现[ArrayDeque](http://docs.oracle.com/javase/7/docs/api/java/util/ArrayDeque.html)来模拟Queue和Stack的行为，官方文档中说ArrayDeque要比LinkedList速度要快，但同样是线程不安全的。
> Resizable-array implementation of the Deque interface. Array deques have no capacity restrictions; they grow as necessary to support usage. They are not thread-safe; in the absence of external synchronization, they do not support concurrent access by multiple threads. Null elements are prohibited. This class is likely to be faster than Stack when used as a stack, and faster than LinkedList when used as a queue.

* Queue
```java
    public static void testQueue(int n) {
        Deque<String> queue = new ArrayDeque<String>();
        for (int i=0; i<n; i++) {
            queue.addLast(generator.next()); // push
        }
        System.out.println("Queue: " + queue);
        System.out.print("Pop order:");
        while (queue.isEmpty() == false) {
            String head = queue.removeFirst(); // pop
            System.out.print(" " + head);
        }
        System.out.println();
    }
```

* Stack
```java
    public static void testStack(int n) {
        Deque<String> stack = new ArrayDeque<String>();
        for (int i=0; i<n; i++) {
            stack.addFirst(generator.next()); // push
        }
        System.out.println("Stack: " + stack);
        System.out.print("Pop order:");
        while (stack.isEmpty() == false) {
            String head = stack.removeFirst(); // pop
            System.out.print(" " + head);
        }
        System.out.println();
    }
```

[overview-uml]: http://7xkcol.com1.z0.glb.clouddn.com/java/tij/holding/collections_uml.png