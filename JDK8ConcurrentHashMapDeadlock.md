JDK1.8中，其内部实现变化较大，内部对不再使用1.7版本的Segment锁，而是使用synchronized + CAS（Unsafe类）实现来更高效的对map中每个Node的细粒度独占锁定并更新。

新实现中，对元素的更新操作代码变化较大。比如下面方法的使用，稍不注意就会产生*死锁*。
```java
public V computeIfAbsent(K key, Function<? super K, ? extends V> mappingFunction)
```
```java
       ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>(16);
       map.computeIfAbsent(
           "AaAa",
           key ->  map.computeIfAbsent("BBBB", key2 -> 42)
       );
```
引用自：*https://stackoverflow.com/questions/43861945/deadlock-in-concurrenthashmap*

执行上面的代码片段会产生死锁。当map中不存在key="AaAa"时，`computeIfAbsent`会插入该key，并将以下lamda函数的返回值(42)作为它的value。而这个lamda函数其实会继续去对key="BBBB"的Node进行同样操作，并设置value=42。但是由于这里的“AaAa”和“BBBB”这个字符串的hashCode一样，导致执行出现死锁(https://stackoverflow.com/questions/43861945/deadlock-in-concurrenthashmap)。

```java
   key ->  map.computeIfAbsent("BBBB", key2 -> 42);
```

这篇文章 https://www.jianshu.com/p/59bd27e137e1 认为是computeIfAbsent方法中的CAS操作造成的，synchronized是可重入的锁，两次去获取同一个Node的锁不会阻塞，为什么CAS会造成这个问题？我太不认同他的说法，还需要进一步分析。

我在一个名为ConcurrentMapBug的类中测试以上代码块，通过命令
` jstack  -l pid`获取到线程的执行堆栈内容如下：

```
"main" #1 prio=5 os_prio=0 tid=0x0062e000 nid=0x614 runnable [0x005ef000]
   java.lang.Thread.State: RUNNABLE
        at java.util.concurrent.ConcurrentHashMap.computeIfAbsent(ConcurrentHashMap.java:1718)
        at concurrent.map.ConcurrentMapBug.lambda$main$1(ConcurrentMapBug.java:13)
        at concurrent.map.ConcurrentMapBug$$Lambda$1/10634667.apply(Unknown Source)
        at java.util.concurrent.ConcurrentHashMap.computeIfAbsent(ConcurrentHashMap.java:1660)
        - locked <0x049a9c60> (a java.util.concurrent.ConcurrentHashMap$ReservationNode)
        at concurrent.map.ConcurrentMapBug.main(ConcurrentMapBug.java:11)
```
注意到这几行：

```
java.util.concurrent.ConcurrentHashMap.computeIfAbsent(ConcurrentHashMap.java:1660)
        - locked <0x049a9c60> (a java.util.concurrent.ConcurrentHashMap$ReservationNode)
        at concurrent.map.ConcurrentMapBug.main(ConcurrentMapBug.java:11)
```
说是在JDK代码中第1660行处被lock了，查看ConcurrentHashMap源码：

```java
public V computeIfAbsent(K key, Function<? super K, ? extends V> mappingFunction) { 
        if (key == null || mappingFunction == null)
            throw new NullPointerException();
        int h = spread(key.hashCode());
        V val = null;
        int binCount = 0; #JDK1.8.0_152源代码中1648行，binCount值初始化为0 
        for (Node<K,V>[] tab = table;;) { #1649行
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & h)) == null) { #1653行
                Node<K,V> r = new ReservationNode<K,V>(); #1654行，新建一个占位Node r
                synchronized (r) { #1655行，获取Node r的monitor锁
                    if (casTabAt(tab, i, null, r)) { #1656行，通过CAS方式将Node r插入到map内置的table中
                        binCount = 1;
                        Node<K,V> node = null;
                        try {
                            if ((val = mappingFunction.apply(key)) != null) #1660行
                                node = new Node<K,V>(h, key, val, null);
                        } finally {
                            setTabAt(tab, i, node);
                        }
                    }
                }
                if (binCount != 0)
                    break;
            }
            else if ((fh = f.hash) == MOVED)
                 tab = helpTransfer(tab, f);
            else { #1672行
                boolean added = false;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) { #1676行
                            binCount = 1; #上面例子不会被执行到这里
                            ....
                        }
                        ... #TreeBin
                   ｝
           }
           if (binCount != 0) { #1710行
           #参考1648行，每次执行computeIfAbsent，都会初始化为0，
           #因此第二次调用computeIfAbsent时不会执行到if body里面代码
               if (binCount >= TREEIFY_THRESHOLD)
                   treeifyBin(tab, i);
               if (!added)
                   return val;
               break;
           }
       } #end 1649行的for_loop
       ...
```
``map.computeIfAbsent("AaAa", mapFunction)``会进入1653行的分支。代码中1654行，这里会创建一个占位符作用的Node(***hash=-3***, key=null, value=null, next Node=null)，然后将该Node插入到map中（1656行）。每个执行该方法的线程都可以自己创建一个这样的Node，因此可以多个线程同时来操作同一个key，但是CAS操作只有一个线程能成功，这和以前1.7版本的排他方式不同。

代码中1656行，CAS 是自旋锁，不存在锁的获取，一般场景下效率较高，这里只会执行一次，如果执行成功才会继续执行，这里成功意思是在此时没有其它线程写入相同key到map中。

执行到1660行时，会执行``key ->  map.computeIfAbsent("BBBB", key2 -> 42);``，这里再次调用``computeIfAbsent``，由于"AaAa"和"BBBB"的hash相同，因此会将值存到同一位置，因此执行到1653行时获取到之前插入的占位符Node **f**，**注意fh = f.hash = -3**，此时会执行1672行的分支，在1676行判断**fh >= 0**为false，因此会结束该分支，由于后面再无代码来退出1649行的循环，所以会进入下一个循环，重复以上过程。**代码因此在1649行的循环中反复执行，而不是被阻塞。**

简而言之，``map.computeIfAbsent("AaAa", mapFunction)``在等待mapFunction.apply(key)的返回值，而mapFunction：``key ->  map.computeIfAbsent("BBBB", key2 -> 42);``却进入了一个死循环，永远都不会返回。因此整个代码得执行就被锁住了，但是这算*死锁*吗？似乎和死锁的定义不太一样！

总之，为了避免这个问题，在JDK1.8中使用ConcurrentHashMap时，不要在computeIfAbsent的lambda函数中再去执行更新其它节点value的操作。

这一点其实在该方法的Java doc中就已经提到：

>   Some attempted update operations on this map by other threads may be blocked while computation is in progress, so the computation should be short and simple,  and must not attempt to update any other mappings of this map.

简而言之，不要像上面例子中那样，在一个更新操作中又去对map中其他元素进行更新。