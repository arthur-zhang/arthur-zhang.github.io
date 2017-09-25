---
layout: post
title:  "Java Class 文件之 UTF8 编码"
date:   2017-09-22 21:17:43
categories: "JVM utf8"
---

最近看了一点kvm，其中有一段校验utf8字符串的代码，觉得比较有意思

以下描述取自 [阮一峰](http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html)

UTF8编码方法如下

UTF-8最大的一个特点，就是它是一种`变长`的编码方式。它可以使用1~4个字节表示一个符号，根据不同的符号而变化字节长度。

UTF-8的编码规则很简单，只有二条：

1. 对于单字节的符号，字节的第一位设为0，后面7位为这个符号的unicode码。因此对于英语字母，UTF-8编码和ASCII码是相同的。
2. 对于n字节的符号（n>1），第一个字节的前n位都设为1，第n+1位设为0，后面字节的前两位一律设为10。剩下的没有提及的二进制位，全部为这个符号的unicode码。
下表总结了编码规则，字母x表示可用编码的位。

| Unicode符号范围 |  UTF-8编码方式  |
| --- | --- |
|0000 0000-0000 007F  | 0xxxxxxx |
|0000 0080-0000 07FF  | 110xxxxx 10xxxxxx |
|0000 0800-0000 FFFF  | 1110xxxx 10xxxxxx 10xxxxxx |
|0001 0000-0010 FFFF  | 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx |



我们来看一下kvm中`preverifier`程序中的一段utf8字符检验代码（常量池中的CONSTANT_Utf8类型）

```c
static void
verifyUTF8String(char *bytes, unsigned short length) {
    unsigned int i;
    for (i = 0; i < length; i++) {
        unsigned int c = ((unsigned char *) bytes)[i];

        if (c == 0) /* no embedded zeros */
            goto failed;
        if (c < 128)
            continue;
        switch (c >> 4) {
            // 0~7  0xxx xxxx 一定是ok的
            default:
                break;

            // 这五个一定是不ok的
            case 0x8: // 1000xxxx
            case 0x9: // 1001xxxx
            case 0xA: // 1010xxxx
            case 0xB: // 1011xxxx
            case 0xF: // 1111xxxx
                goto failed;

            // 两个字节表示
            case 0xC: // 1100xxxx
            case 0xD: // 1101xxxx
                /* 110xxxxx  10xxxxxx */
                i++;
                if (i >= length)
                    goto failed;
                // 判断第二字节是否为10xxxxxx，和0xC0(0x11000000)进行与操作，如果等于0x80(0x100000000),则表示最高两位为10
                if ((bytes[i] & 0xC0) == 0x80)
                    break;
                goto failed;

            // 三个字节表示
            case 0xE:
                /* 1110xxxx 10xxxxxx 10xxxxxx */
                i += 2;
                if (i >= length)
                    goto failed;
                // 判断第二 第三字节是否为10xxxxxx，和0xC0(0x11000000)进行与操作，如果等于0x80(0x100000000),则表示最高两位为10
                if (((bytes[i - 1] & 0xC0) == 0x80) &&
                    ((bytes[i] & 0xC0) == 0x80))
                    break;
                goto failed;
        } /* end of switch */
    }
    return;

    failed:
        JAVA_ERROR(context, "Bad utf string");
}
```

上面的代码其实是一段非常经典的代码，我找了很多utf8编码的实现，大体逻辑都跟这个类似，还是比较巧妙的。

细心的观众，可能会注意到一个问题

> 你前面说了，utf8是用1~4个字节来构成的，但是上面的代码，明明就只有判断1~3个字节的逻辑，如果有四个字节，岂不是这里的逻辑会错乱？

确实，这个我纠结了一会，一开始还以为是kvm太古老，utf编码支持得不行，我看了下最新版里面Java处理utf8的逻辑，也是一样的，google了一番，发现了原因

来自[维基百科](https://zh.wikipedia.org/wiki/UTF-8)

> Java
在通常用法下，Java程序语言在通过InputStreamReader和OutputStreamWriter读取和写入串的时候支持标准UTF-8。但是，Java也支持一种非标准的变体UTF-8，供对象的序列化，Java本地界面和在class文件中的嵌入常数时使用的modified UTF-8

注意到有一个东西叫`modified UTF-8`，java的`DataInput`中文档有比较详细的介绍 [modified-utf-8](http://docs.oracle.com/javase/7/docs/api/java/io/DataInput.html#modified-utf-8)


The differences between this format and the standard UTF-8 format are the following:

- The null byte '\u0000' is encoded in 2-byte format rather than 1-byte, so that the encoded strings never have embedded nulls.
- Only the 1-byte, 2-byte, and 3-byte formats are used.
- Supplementary characters are represented in the form of surrogate pairs.

翻译成人话就是:

modified UTF-8跟标准utf8的区别在于：

- 空\u0000用两个字节表示，而不是一个字节，使用双字节的0xc0 0x80，而不是单字节的0x00。这保证了在已编码字符串中没有嵌入空字节。因为C语言等语言程序中，单字节空字符是用来标志字符串结尾的。当已编码字符串放到这样的语言中处理，一个嵌入的空字符将把字符串一刀两断。
- 只有1-2-3字节三种，没有4字节表示的utf8，UTF-8中需要4字节的字符在变种UTF-8中变成需要6字节
- 没看懂，看了翻译的也没看懂，囧

参考文章：

http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html
http://docs.oracle.com/javase/7/docs/api/java/io/DataInput.html#modified-utf-8
https://zh.wikipedia.org/wiki/UTF-8

    


