---
title: ThreadLocal源码分析
tags:
  - ThreadLocal
categories:
  - 面试专题
top: false
date: 2019-04-15 11:00:25
---

## ThreadLocal 原理分析



### 数据结构图
![数据结构图](https://user-images.githubusercontent.com/42650061/45798359-3d2d3500-bcdc-11e8-8bee-d40bb3332cd5.jpg)


### 流程图
![流程图](https://user-images.githubusercontent.com/42650061/45798346-27b80b00-bcdc-11e8-8736-0171ed25f74d.jpg)


### 源码解析

> 基本上上诉整个流程就是ThreadLocal 如何取值，如何设置值。下面会进行细致的源码分析

#### 数据存储结构

> 就如数据结构图所示 ，ThreadLocal 本身并不会存储数据，每一个Thread对象都会存储一个ThreaLocal.ThreadLocalMap 属性 , ThreadLocalMap 会在使用ThreadLocal的时候初始化一个长度为INITIAL_CAPACITY 16的数组 ，数组中Entry对象.

> 其中 Entry 的数据结构 为 key value 结构 其中 key为 初始化的 ThreadLocal对象 即ThreadLocal threadLocal = new ThreadLocal() value 则为 set()的值 

```java
    static class Entry extends WeakReference<ThreadLocal<?>> {
                /** The value associated with this ThreadLocal. */
                Object value;
    
                Entry(ThreadLocal<?> k, Object v) {
                    super(k);
                    value = v;
                }
            }
```


#### set 操作

- set

```java
        public void set(T value) {
            //获取当前执行线程
            Thread t = Thread.currentThread();
            //从当前thread 中取出 threadlocals的值
            ThreadLocalMap map = getMap(t);
            if (map != null)
                //设置当前threadLocal对象到 ThreadLocalMap中
                map.set(this, value);
            else
                //创建一个ThreadLocalMap 
                createMap(t, value);
        }
```


- getMap

```java
        ThreadLocalMap getMap(Thread t) {
            //t 表示  Thread.currentThread()
            return t.threadLocals;
        }
```

- createMap

```java
        void createMap(Thread t, T firstValue) {
            //初始化一个 ThreadLocalMap
            t.threadLocals = new ThreadLocalMap(this, firstValue);
        }

```
```java
     ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
         		//初始化一个16长度的 Entry数组
                table = new Entry[INITIAL_CAPACITY];
         		//根据 threadLocal对象的hashcode 计算出应该落在哪个数组下标上
                int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
         		//设置当前的下标对象
                table[i] = new Entry(firstKey, firstValue);
                size = 1;
                setThreshold(INITIAL_CAPACITY);
            }
```

map.set(this, value)精华 

```java
    private void set(ThreadLocal<?> key, Object value) {
                Entry[] tab = table;
                int len = tab.length;
       		 //计算key的下标
                int i = key.threadLocalHashCode & (len-1);
        
    // e = tab[i = nextIndex(i, len)] 这个是用来解决hash冲突的，如果i小标的hash值不为空，则在i+1上设置值		
                for (Entry e = tab[i];
                     e != null;
                     e = tab[i = nextIndex(i, len)]) {
                    
                    //获取Entry的key值
                    ThreadLocal<?> k = e.get();
    			
                    //如果key的对象与threadlocal的对象一致
                    if (k == key) {
                        //替换原来的值
                        e.value = value;
                        
                        //结束方法
                        return;
                    }
    				
                    //如果k 已经被GC回收 ，则会进行清空 
                    // Entry 中的key 是弱引用，当堆内存不足的时候回进行GC回收
                    if (k == null) {
                        replaceStaleEntry(key, value, i);
                        return;
                    }
                }
    			
        //执行到这里表示要么是下标数组的对象为空，要么发送了hash冲突了 并且现在的i值是 +1 过了的 ，并且设置值
                tab[i] = new Entry(key, value);
                int sz = ++size;
                if (!cleanSomeSlots(i, sz) && sz >= threshold)
                    rehash();
            }
```
总结一下 ：

- set的时候检查 如果thread没有ThreadLocalMap 则初始话一个长度为16的数组，并计算除当前threadLocal对象的hash 下标 在进行存储
- 如果存在ThreadLocalMap则判断计算出来的下标是否已经存在值
  - 如果存在，则进行判断是否为一样的对象，
    - 如果两个对象不同，则表示发送了hash冲突  则 在计算出来的下标值+1 进行存储 （解决hash冲突的原理）
    - 如果两个对象一致，则表示需要替换原来的值 
    - 如果对象的key 为null 则表示threadlocal``key被回收了 
  - 如果没有发送hash 冲突，则存储当前的值到数组中

#### get操作
```java
      public T get() {
            Thread t = Thread.currentThread();
          	//获取ThreadLocalMap
            ThreadLocalMap map = getMap(t);
            if (map != null) {
                //获取Entry 根据
                ThreadLocalMap.Entry e = map.getEntry(this);
                if (e != null) {
                    @SuppressWarnings("unchecked")
                    T result = (T)e.value;
                    return result;
                }
            }
          	//返回初始值 
            return setInitialValue();
        }
```


map.getEntry(this)
```java
    private Entry getEntry(ThreadLocal<?> key) {
        //获取hash 下标
                int i = key.threadLocalHashCode & (table.length - 1);
                Entry e = table[i];
        //判断当前是否key值是否相等
                if (e != null && e.get() == key)
                    return e;
                else
                    //如果不相等则肯定是发送了hash冲突
                    return getEntryAfterMiss(key, i, e);
            }
```
getEntryAfterMiss
```java

    private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
                Entry[] tab = table;
                int len = tab.length;
    			
        		
                while (e != null) {
                    ThreadLocal<?> k = e.get();
                    //判断是否为该对象
                    if (k == key)
                        //如果相等则返回
                        return e;
                    if (k == null)
                        expungeStaleEntry(i);
                    else
                        //寻找下一个+1的下标
                        i = nextIndex(i, len);
                    e = tab[i];
                }
                return null;
            }
```
总结一下 ：

- 如果不存在ThreadLocalMap或者threadLocal不存在ThreadLocalMap 中则返回初始值
- 如果存在 ThreadLocalMap则会根据下标去查找所对应的值，如果发生了hash冲突则+1继续找，直到数组找完，或者找到key值为当前threadLocal的值.

#### 问题

##### 1、内存泄漏的问题

网上有很多文章在讨论ThreadLocal 内存泄漏的问题，其中有说到是由于ThreadLocal使用了弱引用，JVM

 GC 了threadlocal之后  导致 ThreadLocalMap 中的key 值（指的是threadLocal对象）为null 然后无法定位到 value值。其实总结下来跟这个关系不大，只不过是因为key值为null 了 value其实也没有意义了，其实，ThreadLocalMap的设计中已经考虑到这种情况，也加上了一些防护措施：在ThreadLocal的get(),set(),remove()的时候都会清除线程ThreadLocalMap里所有key为null的value 

ThreadLocal内存泄漏的根源是：由于ThreadLocalMap的生命周期跟Thread一样长，如果没有手动删除对应key就会导致内存泄漏，而不是因为弱引用。

##### 2、线程池使用的问题 

   ThreadLocalMap 是存放在线程内部的，使用线程池的时候，由于线程对象没有被销毁，所以会存在新的线程运行程序会已经在ThreadLocalMap中存在，那么当新的线程 set的时候会替换掉原来的值（使用相同的ThreadLocal对象）。



参考地址 ： https://allenwu.itscoder.com/threadlocal-source