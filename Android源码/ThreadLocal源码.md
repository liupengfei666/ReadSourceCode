
### ThreadLocal原理
1. ThreadLocal是一种线程本地存储机制，能够保证多线程环境下访问共享变量的安全，可以利用该机制将数据缓存在某个线程内部，该
线程可以在任意时刻、任意方法中获取缓存数据，是一种以空间换时间的设计思想
2. ThreadLocal底层是通过ThreadLocalMap来实现的，每个Thread对象中都存在一个ThreadLocalMap,Map的key为ThreadLocal
对象，Map的value为需要缓存的值
3. 如果在线程池中使用ThreadLocal会造成内存泄漏，因为当ThreadLocal对象使用完后，应该要把设置的key-value，也就是Entry
对象进行回收，但线程池中的线程不会回收，而线程对象是通过强引用指向ThreadLocalMap，ThreadLocalMap也是通过强引用指向
Entry对象，线程不会回收，Entry对象也就不会回收，从而出现内存泄漏。*解决办法是：在使用了ThreadLocal对象之后，手动调用
ThreadLocal的remove方法，手动清除Entry对象*

### ThreadLocal源码相关
ThreadLocal主要看一下set、get方法
```
    public void set(T value) {
        //获取当前线程
        Thread t = Thread.currentThread();
        //通过当前线程获取线程中的ThreadLocalMap对象
        ThreadLocalMap map = getMap(t);
        if (map != null) // 更新值
            map.set(this, value);
        else //第一次创建
            createMap(t, value);
    }
    
    // 获取ThreadLocalMap
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
    
    //创建ThreadLocalMap,它是Thread中的一个全局变量
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```
以上是set函数相关，每个线程中都有一个ThreadLocalMap,它是线程私有的，这样就保证了其它线程不会影响到它里面的数据。
```
    /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     * 1. 返回线程中以当前threadlocal为key的值
     * 2. 如果以ThreadLocal为key获取到的值为空，则进行初始化并返回
     *
     * @return the current thread's value of this thread-local
     */
    public T get() {
        //获取当前线程
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) { //获取到数据
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        //没有获取到数据，进行初始化
        return setInitialValue();
    }
    
    /**
     * Variant of set() to establish initialValue. Used instead
     * of set() in case user has overridden the set() method.
     * set()的一个变体，如果用户重写了set（）,则用此方法代替
     *
     * @return the initial value
     */
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```
最后看一下ThreadLocalMap里面存储数据是一个什么样的数据结构
```
    //可以看到它是一个Entry数组，并不像名字一样是一个map结构
    private Entry[] table;
    
    //Entry它继承一个软引用防止内存泄漏
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    
    /**
     * 通过这个方法来删除那些key为null的数据，通过翻译来看一下，比较重要的方法基本都能在它的注释中获取到重要信息
     * Expunge a stale entry by rehashing any possibly colliding entries
     * lying between staleSlot and the next null slot. 
     * 通过重新散列位于 staleSlot 和下一个空槽之间的任何可能冲突的条目来删除过时的条目
     * This also expunges any other stale entries encountered before the trailing null.  
     * 这也会删除在尾随 null 之前遇到的任何其他陈旧条目
     * See Knuth, Section 6.4
     *
     * @param staleSlot index of slot known to have null key
     * 空键插槽的 staleSlot index 位置
     * @return the index of the next null slot after staleSlot
     * (all between staleSlot and this slot will have been checked
     * for expunging).
     * staleSlot 之后的下一个空槽的索引（所有在 staleSlot 和这个槽之间的都将被检查是否被删除）
     */
    private int expungeStaleEntry(int staleSlot) {
        Entry[] tab = table;
        int len = tab.length;

        // expunge entry at staleSlot 删除staleSlot位置的entry
        tab[staleSlot].value = null;
        tab[staleSlot] = null;
        size--;

        // Rehash until we encounter null 再哈希，直到遇到null值
        Entry e;
        int i;
        // 注意一下nextIndex和preIndex，它们是循环取值的方式
        for (i = nextIndex(staleSlot, len);
             (e = tab[i]) != null;
             i = nextIndex(i, len)) {
            ThreadLocal<?> k = e.get();
            if (k == null) { // 如果key为空就清楚当前插槽数据
                e.value = null;
                tab[i] = null;
                size--;
            } else { // 否则进行再哈希
                int h = k.threadLocalHashCode & (len - 1);
                if (h != i) {
                    tab[i] = null;

                    // Unlike Knuth 6.4 Algorithm R, we must scan until
                    // null because multiple entries could have been stale.
                    while (tab[h] != null)
                        h = nextIndex(h, len);
                    tab[h] = e;
                }
            }
        }
        return i;
    }
```
看源码可以知道以下信息：
1. 初始容量是16，并且必须是2的幂
2. 加载因子默认是2/3
3. 通过key，也就是ThreadLocal的哈希值与上Entry的长度减一来确定位置，即key.hashCode & (table.length - 1)
4. Entry继承一个弱引用，会通过expungeStaleEntry()来删除那些key为null的数据

这就是简要的ThreadLocal源码分析，要想详细还得看源码！！！
