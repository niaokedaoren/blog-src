---
title: Base64编码简介
date: 2016-05-17 15:14:21
categories:
- java
tags:
- base64
- protobuf
---

#### 缘起
想把protobuf的message按字节数组输出，以前写到hbase里，很自然；但是现在需要写到hdfs上，突然就没法子了。google大法，发现有个base64的东西，问题一下迎刃而解。
1. 输出字节数组：转成base64 string；
2. 读取字节数组：从base64 string反解回来。

#### Dependencies
commons-codec，这个包比较好用，一堆thread safe的静态函数，就能搞定问题。
```xml
<dependency>
  <groupId>commons-codec</groupId>
  <artifactId>commons-codec</artifactId>
  <version>1.10</version>
</dependency>
```

#### 原理
1. 每3个8位二进制码位一组，转换为4个6位二进制码为一组（不足6位时地位补0）。3个8位二进制码和4个6位二进制码长度都是24位。
2. 对获得的4个6位二进制码补位，每个6位二进制码添加两位高位0，组成4个8位二进制码。
3. 将获得的4个8位二进制码转换为4个十进制码。
4. 将获得的十进制码转换为Base64字符表中对应的字符，[对照表](http://aub.iteye.com/blog/1129273)。

另外，当原文的二进制码长度不是24的倍数，最终转换为十进制时也不足4项，这时就需要用=补位。
RFC2045还规定每行位76个字符，每行末尾需添加一个回车换行符(\r\n)，即便是最后一行不够76个字符，也要加换行符。

#### commons-codec 三种模式
1. 标准的RFC2045，每76个字符加上(\r\n)，有补位等号，即chunked的模式
2. 有补位等号，但是不加(\r\n)，即url unsafe模式
3. Url Base64,也就是将“+”和“\”换成了“-”和“_”符号，且不适用补位，即url safe模式


#### 例子
```java
public class Base64Test {
    private static String decode(String base64Str) {
        return new String(Base64.decodeBase64(base64Str));
    }

    public static void main(String[] args) {
        String text = "hello, Base64!";
        String base64Text = Base64.encodeBase64String(text.getBytes());
        String base64TextChunked = new String(Base64.encodeBase64Chunked(text.getBytes()));
        String base64UrlSafeText = Base64.encodeBase64URLSafeString(text.getBytes());

        System.out.println("text: " + text.length());
        System.out.println("text byte: " + text.getBytes().length);
        System.out.println("url-unsafe base64: " + base64Text.length());
        System.out.println("chunked base64: " + base64TextChunked.length());
        System.out.println("url-safe base64: " + base64UrlSafeText.length());

        System.out.println(base64Text);
        System.out.println(base64TextChunked);
        System.out.println(base64UrlSafeText);

        System.out.println(decode(base64Text));
        System.out.println(decode(base64TextChunked));
        System.out.println(decode(base64UrlSafeText));
    }
}
/* output
text: 14
text byte: 14
url-unsafe base64: 20
chunked base64: 22
url-safe base64: 19
aGVsbG8sIEJhc2U2NCE=
aGVsbG8sIEJhc2U2NCE=

aGVsbG8sIEJhc2U2NCE
hello, Base64!
hello, Base64!
hello, Base64!
 */
```
- 分析首三字母 **hel** --> **aGVs**
二进制表示：
h = 01101000
e = 01100101
l = 01101100

| c1 | c2 | c3 | c4 |
| :--: | :--: | :--: | :--: |
| 011010 | 000110 | 010101 | 101100 |
| 26 | 6 | 21 | 44 |
| a | G | V | s |

- 分析末尾两个字母 **4！**
根据规则，3个原始byte一组转换，最后剩余的byte组成一组，
二进制表示：
4 = 00110100
! = 00100001
每6个bit构成base64编码的一个字母，剩余4个bit后面需补充2个0bit，构成第三个字母

| c1 | c2 | c3 |
| :--: | :--: | :--: |
| 001101 | 000010 | 000100 |
| 13 | 2 | 4 |
| N | C | E |

因为只有NCE三个字母，所以需要一个部位等号，凑足4个字母，如果按照标准的base64编码，还需要加上 \r\n

#### 结合protobuf的应用
```java
protected void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
    UserModel.Builder builder = UserModel.newBuilder();
    for (Text entry : values) {
        byte[] pbUserModel = Base64.decodeBase64(entry.toString());
        builder.mergeFrom(pbUserModel);
    }

    UserModel model = builder.build();
    String base64Str = Base64.encodeBase64URLSafeString(model.toByteArray());

    context.write(key, new Text(base64Str));
}
```

#### 利弊
##### 优点
1. 7 Bit ASCII characters are safe for transmission over the network and between different systems
2. SMPT protocol for emails supports only 7 Bit ASCII Characters. Base64 is the commonly used technique to send binary files as attachments in emails.

##### 缺点
1. Base64 encoding bloats the size of the original binary stream by 33 percent
2. Encoding/Decoding process consumes resources and can cause performance problems

reference:
1. https://commons.apache.org/proper/commons-codec/apidocs/org/apache/commons/codec/binary/Base64.html
2. http://aub.iteye.com/blog/1129273
3. http://dev-faqs.blogspot.hk/2012/12/advantages-of-base-64-encoding.html
