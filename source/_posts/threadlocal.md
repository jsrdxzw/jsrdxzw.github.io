---
title: ThreadLocal原理和使用的问题
date: 2022-01-07 11:19:30
excerpt: ThreadLocal原理和使用的问题，解决方案
tags:
- 多线程
categories: 后端
---

## ThreadLocal源码

ThreadLocal可以实现每个线程的隔离，自定义线程级别的变量，使用如下:
```java
private static final ThreadLocal<SimpleDateFormat> formatter = ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyyMMdd HHmm"));
```
可以使用 `get（）` 和 `set（）` 方法来获取默认值或将其值更改为当前线程所存的副本的值，从而避免了线程安全问题。

ThreadLocal使用了ThreadLocalMap去存取值，而每个Thread类都有自己的ThreadLocalMap，ThreadLocalMap底层使用的是
Entry[] table，用来保存每个ThreadLocal对应的value。

具体的源码分析如下：

```java
// 定义在ThreadLocal里面，被每个Thread所引用：ThreadLocal.ThreadLocalMap threadLocals
static class ThreadLocalMap {

    /**
     * The entries in this hash map extend WeakReference, using
     * its main ref field as the key (which is always a
     * ThreadLocal object).  Note that null keys (i.e. entry.get()
     * == null) mean that the key is no longer referenced, so the
     * entry can be expunged from table.  Such entries are referred to
     * as "stale entries" in the code that follows.
     * Entry的Key为弱引用，而value是强引用，会导致内存泄漏
     */
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }

    private void set(ThreadLocal<?> key, Object value) {

        // 每个ThreadLocal作为一个key，求hash之后取余，设置value
        Entry[] tab = table;
        int len = tab.length;
        int i = key.threadLocalHashCode & (len-1);

        for (Entry e = tab[i];
             e != null;
             e = tab[i = nextIndex(i, len)]) {
            ThreadLocal<?> k = e.get();

            // 下标相同，则直接赋值，新值覆盖旧值，一般一个thread的一个ThreadLocal实例，key是相同的
            if (k == key) {
                e.value = value;
                return;
            }

            if (k == null) {
                replaceStaleEntry(key, value, i);
                return;
            }
        }

        // 第一次设置值，则新建Entry
        tab[i] = new Entry(key, value);
        int sz = ++size;
        // 清理key为null的情况，设置value为null，防止内存泄漏
        if (!cleanSomeSlots(i, sz) && sz >= threshold)
            rehash();
    }
}
```

ThreadLocal本身的hashcode作为数组下标，value作为值，存在ThreadLocalMap的Entry[]中，可以保证多线程安全
![ThreadLocal源码分析图](/img/threadlocal-1.png)

## ThreadLocal内存泄漏

Entry的Key是ThreadLocal，它是弱引用的
> 如果一个对象只具有弱引用，那就类似于可有可无的生活用品。
> 弱引用与软引用的区别在于：只具有弱引用的对象拥有更短暂的生命周期。
> 在垃圾回收器线程扫描它 所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。
> 不过，由于垃圾回收器是一个优先级很低的线程， 因此不一定会很快发现那些只具有弱引用的对象。 
> 弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java 虚拟机就会把这个弱引用加入到与之关联的引用队列中。

一旦ThreadLocal没有被外部引用，那么垃圾回收就会回收掉Key(ThreadLocal)，这样就会导致ThreadLocalMap中key为null，
而value还存在着强引用，只有thead线程退出以后,value的强引用链条才会断掉。
但如果当前线程迟迟不结束的话，这些key为null的Entry的value就会一直存在，造成内存泄漏
> Thread Ref -> Thread -> ThreadLocalMap -> Entry -> value

### 防止内存泄漏的措施
1. 每次使用完ThreadLocal都调用它的remove()方法清除数据

## 线程池中使用ThreadLocal的问题
ThreadLocal是使用Thread中的ThreadLocalMap进行存储的，如果都是新线程使用起来没有问题，但是如果是线程池，还是会
有一些问题的，下面看一段代码

```java
private static final ThreadLocal currentUser = ThreadLocal.withInitial(() -> null);
@GetMapping("wrong")public Map wrong(@RequestParam("userId") Integer userId) { 
    //设置用户信息之前先查询一次ThreadLocal中的用户信息 
    String before = Thread.currentThread().getName() + ":" + currentUser.get(); 
    //设置用户信息到ThreadLocal 
    currentUser.set(userId); 
    //设置用户信息之后再查询一次ThreadLocal中的用户信息 
    String after = Thread.currentThread().getName() + ":" + currentUser.get(); 
    //汇总输出两次查询结果 
    Map result = new HashMap(); 
    result.put("before", before); 
    result.put("after", after); 
    return result;
}
```
这里面每次请求，第一次获取的值都会是null吗？
其实是不一定的，可能before会有值！
原因是Tomcat使用了线程池，线程池会重用固定的几个线程，一旦线程重用，那么很可能首次从`ThreadLocal`获取的值是之前其他用户的请求遗留的值。
这时，ThreadLocal 中的用户信息就是其他用户的信息。

### 解决方案
在代码运行结束，需要显示的清除
```java
@GetMapping("right")
public Map right(@RequestParam("userId") Integer userId) { 
    String before = Thread.currentThread().getName() + ":" + currentUser.get(); 
    currentUser.set(userId); 
    try { 
        String after = Thread.currentThread().getName() + ":" + currentUser.get(); 
        Map result = new HashMap(); 
        result.put("before", before); 
        result.put("after", after); return result; 
    } finally { 
        //在finally代码块中删除ThreadLocal中的数据，确保数据不串 
        currentUser.remove(); 
    }
}
```

这样这个线程运行结束对应的ThreadLocal数据也清除了
