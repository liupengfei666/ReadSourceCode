### ArrayList原理
先讲原理
1. ArrayList底层是通过数组来实现的，通常来说查找快，增删慢
2. 查找快是在知道下标的情况下，如果不知道下标，但是知道value的话也是需要一个个遍历即O(n)的复杂度
3. 增删慢是因为数组在中间增删的时候需要移动数据，有一个拷贝移动的过程。但是如果是最后一个元素的话也是O(1)的复杂度
### ArrayList特点及源码
1. 实现了List接口，它是一个有序可重复的集合，允许null值
2. 实现了Cloneable接口，允许被复制
3. 实现了Serializable表明可以被序列化
```java
    public class ArrayList<E> extends AbstractList<E>
            implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```
4. 默认容量为10，如果确定元素个数，最好使用传参的方式创建集合
```java
    private static final int DEFAULT_CAPACITY = 10;

    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) { //有初始容量
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) { //初始容量为0
            this.elementData = EMPTY_ELEMENTDATA;
        } else { //非法容量
            throw new IllegalArgumentException("Illegal Capacity: "+
                initialCapacity);
        }
    }
```
5. ArrayList扩容为当前容量的1.5倍
```java
    private void grow(int minCapacity) { //扩容方法
        // overflow-conscious code 之前数组长度
        int oldCapacity = elementData.length; 
        // 新数组长度，为之前数组的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        //判断新的数组长度是否能够装下
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        //判断新数组长度是否超过最大长度
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```
### ArrayList的几个方法
#### 1. 增加元素
```java
    /**
     * Appends the specified element to the end of this list.
     * 在列表最后追加元素
     *
     * @param e element to be appended to this list
     * @return <tt>true</tt> (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }

    private void ensureCapacityInternal(int minCapacity) {
        //初始赋值时没有给elementData数组设置容量，这儿会判断会取minCapacity与默认值中较大的
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        //确定容量
        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);//上文有介绍
    }
```
#### 2.增加元素到指定位置
```java
    public void add(int index, E element) {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1, size - index);
        elementData[index] = element;
        size++;
    }
```
这个方法主要就是在中间插入元素，index位置之后的所有元素都往后移动一位。这就是为什么ArrayList插入慢的原因。
#### 3.增加数据集合
无论是addAll(Collection<? extends E> c)还是addAll(int index, Collection<? extends E> c)，都是通过系统方法
System.arraycopy函数完成的，与上面的增加元素到指定位置类似，就不列举源码了。不同的是都需要判断扩容，并且增加集合到指定
位置还要判断是否越界。
#### 4.移除元素
```java
    public E remove(int index) {
        if (index >= size) //判断是否越界
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        modCount++;
        //获取到需要移除的元素
        E oldValue = (E) elementData[index];

        //计算需要移动多少元素
        int numMoved = size - index - 1;
        if (numMoved > 0) //如果需要移动元素，则通过系统函数完成
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        //将最后的一个元素置空，让GC进行回收
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;//返回移除的元素
    }
```
#### 5.移除指定元素
这个函数看注释就比较明白了：删除列表中第一个与给定元素相同的元素，如果存在删除后返回true，如果不存在则列表不变
```java
    /**
     * Removes the first occurrence of the specified element from this list,
     * if it is present.  If the list does not contain the element, it is
     * unchanged.  More formally, removes the element with the lowest index
     * <tt>i</tt> such that
     * <tt>(o==null&nbsp;?&nbsp;get(i)==null&nbsp;:&nbsp;o.equals(get(i)))</tt>
     * (if such an element exists).  Returns <tt>true</tt> if this list
     * contained the specified element (or equivalently, if this list
     * changed as a result of the call).
     *
     * @param o element to be removed from this list, if present
     * @return <tt>true</tt> if this list contained the specified element
     */
    public boolean remove(Object o) {
        if (o == null) {
            //找到第一个为null的元素删除，如果存在的话
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            //找到第一个与o相同的元素删除，如果存在的话
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
    
    //跳过边界检查且不返回已删除值的私有删除方法。
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index, numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
```
#### 6.删除一块区间内的元素
```java
    protected void removeRange(int fromIndex, int toIndex) {
        // Android-changed: Throw an IOOBE if toIndex < fromIndex as documented.
        // All the other cases (negative indices, or indices greater than the size
        // will be thrown by System#arrayCopy.
        if (toIndex < fromIndex) { //判断下标合法性
            throw new IndexOutOfBoundsException("toIndex < fromIndex");
        }

        modCount++;
        int numMoved = size - toIndex;
        //通过系统方法进行移动
        System.arraycopy(elementData, toIndex, elementData, fromIndex,
                         numMoved);

        // clear to let GC do its work
        int newSize = size - (toIndex-fromIndex);
        //移动完成后把后面的元素置空
        for (int i = newSize; i < size; i++) {
            elementData[i] = null;
        }
        size = newSize;
    }
```
#### 7.删除一个集合中的元素
```java
    public boolean removeAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, false);
    }

    //如果complement为false，则移除elementData和c中都包含的数据
    //如果complement为true，则移除elementData中除c以外的元素
    private boolean batchRemove(Collection<?> c, boolean complement) {
        //创建一个局部变量，防止被修改
        final Object[] elementData = this.elementData;
        int r = 0, w = 0;
        boolean modified = false;
        try {
            //进行遍历
            for (; r < size; r++)
                //如果complement为true，则记录c与elementData的交集，否则记录c在elementData中的补集
                if (c.contains(elementData[r]) == complement)
                    elementData[w++] = elementData[r];
        } finally {
            // Preserve behavioral compatibility with AbstractCollection,
            // even if c.contains() throws.
            //如果上面出错了会走这儿，系统会将r后面的元素放到w后面
            if (r != size) {
                System.arraycopy(elementData, r, elementData, w, size - r);
                w += size - r;
            }
            if (w != size) { //如果w与size不相等说明需要删除元素
                // clear to let GC do its work
                for (int i = w; i < size; i++)
                    elementData[i] = null;
                modCount += size - w;
                size = w;
                modified = true;
            }
        }
        return modified;
    }
```
另外获取元素，修改元素，清空等函数比较简单就不记录了。

