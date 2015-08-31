title: TIJ 读书笔记 -- 复用类
date: 2015-08-30 16:15:38
categories: java
tags: 继承,组合,final,类加载
---

### 缘起
平时使用java，写的最多也就是mapreduce代码，除了刚开始会带来一点新鲜感，等到熟悉map、reduce这一套之后，剩下的就是机械重复。这种厌倦感重新燃起了深入学习java的欲望。正好家里有一本java编程思想，就抓起来看了第七章，略有一点小感悟，就作为读书笔记写下来。希望自己能坚持看下去、写下去。

-----
### 组合和继承
感觉平时还是用组合比较多，继承基本不用（除了继承一下Mapper和Reducer，这种条条框框的代码，用继承，确实可以给使用者带来不少便利）。我觉得组合最大的好处就是给了复用代码的程序员更大的定制空间：
1. 可以选择性的暴露原有的接口；
2. 重新定制一下原有的接口（当然要真正达到以假乱真的效果，还是需要继承一下，装饰模式就是这么搞的）。

另外，对于继承，父类想要暴露出来的接口，应该是public，因为二次开发想要继承的子类很有可能是出现在其他包下面。

### 语言细节
#### 构造函数调用顺序
构造函数的初始化顺序，一般都是先父类，后子类。
```java
class Game {
    Game() {
        println("Game");
    }
}

class BoardGame extends Game {
    BoardGame(int size) {
        // The compiler auto-insert super() here.
        println("BoardGame");
    }
}

public class Chess extends BoardGame {
    Chess() {
        super(11);
        println("Chess");
    }

    public static void main(String[] args) {
        new Chess();
        /* output:
            Game
            BoardGame
            Chess
        */
    }
}
```
在生成Chess对象，首先会调用父类BoardGame（在BoardGame的构造方法中，编译器会自动加入父类Game的默认构造方法调用），发现BoardGame还有父类Game，所以整个初始化顺序就是：
Game -> BoardGame -> Chess

#### 类的加载顺序
类的加载发生在第一次new对象或调用static变量（方法）时。其变量的初始化顺序是先static变量，后实例变量；先父类，后子类。这个顺序是比较合理的，static变量是独立于类的实例的，所以在类加载的时候率先初始化，而实例变量等到真正需要new一个实例的时候再初始化；同样，子类是依赖于父类的，所以父类先初始化。这个过程很好验证，下面给个例子说明。
```java
class Insect {
    Insect() {
        println("Insect: constructor");
    }

    {
        println("Insect: instance initialization");
    }

    static {
        println("Insect: static initialization");
    }
}

class Beetle extends Insect {
    static int loadTimes = 0;
    Beetle() {
        println("Beetle: constructor");
    }

    static {
        println("Beetle: static initialization");
        loadTimes++;
    }
}
```

* Insect的初始化顺序
```java
new Insect();
```
> Insect: static initialization
> Insect: instance initialization
> Insect: constructor

* Insect和Beetle的初始化顺序
```java
new Beetle();
```
> Insect: static initialization
> Beetle: static initialization
> Insect: instance initialization
> Insect: constructor
> Beetle: constructor

#### final 保留字段的用法
final修饰符，大概有三个用途，修饰变量、方法、类。

##### 修饰变量
final修饰变量，那么变量就成了常量，初始化之后不能“修改”，对于基本类型无法修改值，对于非基本类型，无法修改引用指向的对象，但是对象的内容还是可以修改的。
```java
public class FinalData {
    final int n = 4;
    final int[] array = new int[n];

    public static void main(String[] args) {
        FinalData fd = new FinalData();
        // fd.n = 5; // The final field FinalData.n cannot be assigned.
        // fd.array = new int[4]; // The final field FinalData.array cannot be assigned
        println("array[0] = " + fd.array[0]);
        fd.array[0] = 4;
        println("array[0] = " + fd.array[0]);

        /* output:
            array[0] = 0
            array[0] = 4
        */
    }
}
```
final字段是必须初始化完成的，即便不在构造函数之前，也要在构造函数中完成。
```java
public class BlankFinal {
    private final int i = 0; // Initialized final
    private final int j; // Blank final
    private final Poppet p; // Blank final reference

    // Initialize all blank finals in the constructor.
    public BlankFinal(int x) {
        j = x;
        p = new Poppet(x);
    }

    public static void main(String[] args) {
        new BlankFinal(10);
    }
}
```

##### 修饰方法和类
用final修饰的方法不能被继承后override；final类则不能被继承，String类就是final类。这个平时基本没怎么用过。TIJ的作者也表示慎用final来修饰方法和类。