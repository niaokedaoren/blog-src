title: TIJ 读书笔记 -- 接口
date: 2015-09-07 22:27:36
categories: java
tags: 接口,策略模式,适配模式
---

### 一个例子
接口很好很强大，如果泛泛而谈，我感觉很虚，不实在，那么就从这个小例子谈谈自己的理解吧。在机器学习算法里面，有些会涉及各种类型的距离计算，例如计算与给定点a距离最近的点b，我们可以用欧氏距离，也可以使用马氏距离，或者是向量cos角度值等等。

很容易发现，计算最近点的算法流程是基本不变的，而变化的部分是距离计算方式，那么我们可以把距离计算方式独立出来，作为一个接口。这个接口可以被实现成不同计算形式，从而做到灵活扩展，但是算法流程部分代码是不需要变更的。这样从某种程度上就做到了“对修改关闭，对扩展开放。”这一经典的做法，就是大名鼎鼎的策略模式。

#### 初体验
下面看看具体代码(已省去逻辑细节)：
* 最近邻算法流程
```java
public class OneNearestNeighbour {
    private List<DataPoint> dataset;
    private DistanceCalculator distanceCalculator;

    public OneNearestNeighbour(List<DataPoint> dataset, DistanceCalculator calculator) {
        this.dataset = dataset;
        distanceCalculator = calculator;
    }

    public DataPoint getNearestNeighbour(DataPoint a) {
        DataPoint result = null;
        double min = Double.MAX_VALUE;
        for (DataPoint b : dataset) {
            double distance = distanceCalculator.distance(a, b);
            if (distance < min) {
                result = b;
                min = distance;
            }
        }

        return result;
    }
}

```

* 距离计算接口
```java
public interface DistanceCalculator {
    double distance(DataPoint a, DataPoint b);
}
```

* 计算欧氏距离
```java
public class EuclideanDistance implements DistanceCalculator {

    public double distance(DataPoint a, DataPoint b) {
        double result = 0;
        // ...
        return result;
    }
}
```

这样就可以用欧氏距离计算最近邻了。
```java
public static void main(String[] args) {
    // ... 
    DistanceCalculator calculator = new EuclideanDistance();
    OneNearestNeighbour oneNN = new OneNearestNeighbour(dataset, calculator);
    oneNN.getNearestNeighbour(point);
}
```

#### 进一步
现在我们厌倦了欧氏距离，想试试马氏距离的效果，所需做的就是用马氏距离实现接口*DistanceCalculator*，算法流程无需变更。
```java
public class MahalanobisDistance implements DistanceCalculator {

    public double distance(DataPoint a, DataPoint b) {
        double result = 0;
        // ...
        return result;
    }
    
    public static void main(String[] args) {
        // ...
        DistanceCalculator calculator = new MahalanobisDistance();
        OneNearestNeighbour oneNN = new OneNearestNeighbour(dataset, calculator);
        oneNN.getNearestNeighbour(point);
    }
}
```

#### 再进一步
有些时候，我们无需从零开始实现接口*DistanceCalculator*，像上面的两个距离计算方式。例如，我们现在已经有两个点相似度的计算类*PointsSimilarity*：
```java
public class PointsSimilarity {
    double similarity(DataPoint a, DataPoint b) {
        double result = 0;
        // ...
        return result;
    }
}
```
把*PointsSimilarity*类借助适配模式，包装成接口*DistanceCalculator*的实现，就可以巧妙的融入到最近邻算法当中。
```java
public class PointsSimilarityAdapter implements DistanceCalculator {

    private PointsSimilarity pointsSimilarity;

    public PointsSimilarityAdapter(PointsSimilarity pointsSimilarity) {
        this.pointsSimilarity = pointsSimilarity;
    }

    public double distance(DataPoint a, DataPoint b) {
        return 1 / (1 + pointsSimilarity.similarity(a, b));
    }
    
}
```

#### 反思
其实在上面这个例子中，接口*DistanceCalculator*完全可以用一个抽象类来代替，可以达到一模一样的效果。但是如果类*PointsSimilarity*在适配的时候，需要继承另一个类，那样用抽象类就没法适配了。

接口和抽象类都可以做到不能被直接实例化，通过抽象方法来规范子类的行为，相当于所有子类都会遵守这个“契约”，所以子类可以向上泛化成父类，对调用者隐藏具体的实现。而接口与抽象类相比，最大的优势就是可以多重继承。

TIJ的作者在本章的最后给出了一个忠告：
> 恰当的原则应该是优先使用类而不是接口。从类开始，如果接口的必需性变得非常明确，那么就进行重构。接口是一种工具，但是它容易被滥用。

这句话我理解不深，在以后的工作学习中，多琢磨，多尝试。