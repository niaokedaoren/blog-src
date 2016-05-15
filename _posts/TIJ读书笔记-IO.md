title: TIJ读书笔记--IO
date: 2016-02-28 10:34:08
categories: 
- java
tags:
- io
---

时间飞逝，转眼就是2016，由于2015下半年事情太多，就把读书这事给松懈了，现在捡起来继续。

平时用java io主要是读写文件，网络传输方面基本不涉及。用java io API给我的最大感受就是读写一下文件，为啥要new那么多类，一层包一层，好混乱。这几天，读了java编程思想的文件IO这一章，稍微弄明白其内在逻辑，写篇博文总结总结。本文相关类依赖图都出自[深入分析 Java I/O 的工作机制](https://www.ibm.com/developerworks/cn/java/j-lo-javaio/)

### 基于字节流的API: InputStream 和 OutputStream
java io的[API][api-overview]是不断演进的，最开始只有支持byte流读写的API，其基类就是InputStream和OutputStream。简单看一下各个类的关系。
![InputStream][uml-inputstream]
![OutputStream][uml-outputstream]

都是基于字节的，和字符编码没有关系:
```
int read(byte[] b, int off, int len)
void write(byte[] b, int off, int len
```
输入输出的类关系图，很相似，就以OutputStream为例说道说道，直接子类有ByteArrayOutputStream，FileOutputStream，FilterOutputStream，ObjectOutputStream，PipedOutputStream，其中FilterOutputStream类比较特殊，这是一个装饰类，其构造函数是FilterOutputStream(OutputStream out)，这是个典型的装饰模式，所以我们在使用相关API的时候，需要Stream对象一层层包起来，达到一个组合的效果，例如
```
new BufferedInputStream(new FileInputStream(filename))。
```

下面举个文件copy的例子：
```java
    private static void copy(String from, String to) throws IOException {
        FileInputStream in = null;
        FileOutputStream out = null;
        try {
            in = new FileInputStream(from);
            out = new FileOutputStream(to);
            byte[] buf = new byte[1024];
            int n = 0;
            while ((n = in.read(buf)) > 0) {
                out.write(buf, 0, n);
            }
        } finally {
            if (in != null) {
                in.close();
            }

            if (out != null) {
                out.close();
            }
        }
    }
```

### 基于字符的API：Reader 和 Writer
可能是为了国际化，java后来推出了基于字符char的API：
```
int read(char[] cbuf, int off, int len)
void write(char[] cbuf, int off, int len)
```
Reader、Writer的子类关系图和OutputStream、InputStream很相似，都基于装饰模式。
![Reader][uml-reader]
![Writer][uml-writer]

### 适配器API：InputStreamReader 和 OutputStreamWriter
从基于byte的InputStream转到基于字符char的Reader，需要适配器InputStreamReader，如果字节编码格式不是系统默认的，还需要指定编码格式，例如UTF-8等，否则读入乱码，特别是中文。
```
InputStreamReader(InputStream in, String charsetName)
```
同样OutputStreamWriter，也充当着类似的角色。适配器模式和装饰模式类似，调用相关API，也需要一层层包起来，例如 
```
new BufferedReader(new InputStreamReader(new FileInputStream(filename), "UTF-8"))
```

下面举个读取文件直接print的例子
```java
    private static void display(String filename) throws IOException {
        FileInputStream in = null;
        BufferedReader bufReader = null;
        try {
            in = new FileInputStream(filename);
            bufReader = new BufferedReader(new InputStreamReader(in, "UTF-8"));
            String line = null;
            while ((line = bufReader.readLine()) != null) {
                System.out.println(line);
            }
        } finally {
            if (in != null) {
                if (bufReader != null) {
                    bufReader.close();
                } else {
                    in.close();
                }
            }
        }
    }
```

[api-overview]: https://docs.oracle.com/javase/7/docs/api/java/io/package-summary.html
[uml-inputstream]: https://www.ibm.com/developerworks/cn/java/j-lo-javaio/image002.png
[uml-outputstream]: https://www.ibm.com/developerworks/cn/java/j-lo-javaio/image004.png
[uml-reader]: https://www.ibm.com/developerworks/cn/java/j-lo-javaio/image008.png
[uml-writer]: https://www.ibm.com/developerworks/cn/java/j-lo-javaio/image006.png
