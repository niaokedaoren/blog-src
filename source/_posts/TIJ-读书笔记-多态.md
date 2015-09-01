title: TIJ 读书笔记 -- 多态
date: 2015-09-01 22:40:12
categories: java
tags: 多态
---

多态是TIJ的第八章，记录一些琐碎的语言细节和陷阱，平时更喜欢用interface的形式来实现多态，那样感觉会更顺眼，纯属个人喜好，O(∩_∩)O哈哈~

### 一个简单例子
```java
package thinking.in.java.polymorphism;

import static thinking.in.java.util.Print.*;

public abstract class Cycle {
    public static void echo(Cycle cycle) {
        println(cycle.name() + " has " + cycle.wheels() + " wheels.");
    }

    public String name() {
        return getClass().getSimpleName();
    }

    public abstract int wheels();

    public static void main(String[] args) {
        Cycle.echo(new Bicycle());
        Cycle.echo(new Tricycle());
        /* output:
         Bicycle has 2 wheels.
         Tricycle has 3 wheels.
         */
    }
}

class Bicycle extends Cycle {
    public int wheels() {
        return 2;
    }
}

class Tricycle extends Cycle {
    public int wheels() {
        return 3;
    }
}
```
在java的世界里，父类中除了static和final方法（private方法也是final方法，这个设定是很合理的，因为private方法是无法被继承的），其它方法都可以被子类继承（或者override），这些方法运行时都是需要动态绑定到具体对象类型，相当于c++里的方法加了virtual字段。那么，为什么static、final、private方法不需要做额外的动态绑定开销呢？原因很简单，static方法是属于类的，和具体对象无关，而final和private方法是无法继承覆盖的，其对应的对象类型是固定的。一般，编译器会对final方法进行展开处理，来提高效率，就像c++里的inline函数。不过对于类的方法能不加final，就尽量不加，但是能private的，尽量private。

现在说说上面这个简单例子，*Cycle.echo()*方法传入的都是Cycle的子类类型，都可以upcast到Cycle类型，但是具体运行到name()和wheels()方法时候，都进行了动态绑定，执行的是子类的方法。这样做的好处是在加入新的子类，e.g. Unicycle时候，基类Cycle根本不需要做任何改变，执行echo方法，如果传入的是Unicycle的对象，就会动态绑定到新编写的方法。这样就可以做到“对修改关闭，对扩展开放了”。

### Covariant Return
子类在覆盖基类的方法时，方法的签名是必须一致的，但是return的类型不一定要一致，子类返回的是父类返回类型的子类型也是可以的，因为这完全不妨碍upcast。
```java
class Grain {
    public String toString() { return "Grain"; }
}

class Wheat extends Grain {
    public String toString() { return "Wheat"; }
}

class Mill {
    Grain process() { return new Grain(); }
}

class WheatMill extends Mill {
    @Override
    Wheat process() { return new Wheat(); }
}

public class CovariantReturn {
    public static void main(String[] args) {
        Mill m = new Mill();
        Grain g = m.process();
        println(g);

        m = new WheatMill();
        g = m.process();
        println(g);
        /* output: 
         Grain
         Wheat
         */
    }
}
```
*Mill的process()和子类WheatMill的process()就是存在这个差异。*

### Pitfall：构造器中调用动态绑定函数
最好还是不要那样做，有可能产生匪夷所思的错误。下面看个TIJ书上的例子：
```java
class Glyph {
    void draw() { println("Glyph.draw()"); }

    Glyph() {
        println("Glyph() before draw()");
        draw();
        println("Glyph() after draw()");
    }
}

class RoundGlyph extends Glyph {
    private int radius = 1;

    @Override
    void draw() { println("RoundGlyph.draw(), radius = " + radius); }

    RoundGlyph() {
        println("RoundGlyph(), radius = " + radius);
    }
}

public class PolyConstructors {
    public static void main(String[] args) {
        new RoundGlyph();
        /* output: 
         Glyph() before draw()
         RoundGlyph.draw(), radius = 0
         Glyph() after draw()
         RoundGlyph(), radius = 1
         */
    }
}
```
在运行*new RoundGlyph();*这个语句时候，先要调用父类Glyph的构造器，但是Glyph()的draw()方法是动态绑定到RoundGlyph的对象上的，而该对象尚未构造完成，所有的实例域都被初始化为二进制0，所以打印出来的radius是0。