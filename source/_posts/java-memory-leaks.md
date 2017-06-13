---
layout: post
title:  "Java内存泄露排查方法"
date:   2017-06-13 21:17:43
categories: Java
---

## 工具分析

- dump内存

```
jmap -dump:format=b,file=dump.hprof  <pid>
```

- YourKit使用


### 常见内存泄露的原因

---

- 资源未关闭
切记在`finally`中关闭一切继承了`Closeable`接口的对象

```java
try {
	RecordWriter recordWriter;
	recordWriter.write(record);
	recordWriter.close();
} catch (TunnelException e) {
} 
```

这段代码中`RecordWriter`继承了`Closeable`，并在`close`方法中做了很多清理的工作。这段代码平时工作都没有问题，但是在`write`抛异常的情况下，就会导致`close` 方法没有调用，导致资源未释放

---

- 应该单例的地方没有单例
典型例子
OkHttpClient，应该被单例来使用，OkHttpClient内部维护了一个请求连接池

```java
OkHttpClient client = new OkHttpClient();
Response response = client.newCall(request)
```

---

- 单例模式没有考虑并发场景

```java
	public OdpsService getOdpsService() {
		if(null == odpsService){
			odpsService = new OdpsServiceImpl();
		}
		return odpsService;
	}
```

---

- 静态域或者集合

```java
    static final ArrayList list = new ArrayList(100);
```

---

- 未重写或者不正确的`hashCode()`和`equals()`
在HashSet或者HashMap等结构中

```java
import java.util.Map;
public class MemLeak {
   // no hashCode or equals();
    public final String key;
    public MemLeak(String key) {this.key =key;}
    
    public static void main(String args[]) {
        try {
            Map map = System.getProperties();
            for(;;) {
                map.put(new MemLeak("key"), "value");
            }
        } catch(Exception e) {
            e.printStackTrace();
        }
    }
}
```

---

- 其它
classloader泄露、ThreadLocal泄露

---
