## LinkedList源码记录
首先还是先讲一下LinkedList的原理，有一个大概的了解
### 一、LinkedList原理
- LinkedList底层是双向链表结构，实现了Deque接口
- LinkedList查询较慢，因为要一个个遍历数据，插入和删除则比较快，只需要修改指针就可以了，但是插入中间位置时首先也需要先遍历
  找到节点位置，然后才能修改指针
- 因为是链表结构，所以LinkedList不要求与数组结构一样空间连续，它可以通过指针利用不连续的空间
### 二、LinkedList继承关系和数据结构
在看源码时，官方给的注释很重要，通常我们从注释就可以了解到这个函数是干什么的，有什么注意事项，对我们阅读源码非常有帮助。在看
LinkedList源码之前首先看一下它的继承关系和底层的数据结构定义：
```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```
可以看到与ArrayList的不同，ArrayList继承AbstractList而LinkedList继承AbstractSequentialList。实际上
AbstractSequentialList又继承了AbstractList，以最大限度的减少实现由顺序访问数据存储（链表）接口所需的工作。对于随机
访问数据（如数组），应优先使用AbstractList而非AbstractSequentialList

它实现了List接口，所以是有序的，可重复的，可以有null的集合；实现了Deque，所以有队列的特性；实现了Cloneable接口，所以它
是可以被复制的；实现了java.io.Serializable，所以它也是可以被序列化的

以下是它底层的数据结构定义
```java
    private static class Node<E> {
        E item;        //数据元素
        Node<E> next;  //指向下一个节点的指针
        Node<E> prev;  //指向上一个节点的指针

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```
### 三、LinkedList几个比较重要的函数定义
看完这几个函数定义之后，就可以快速的浏览它的常用方法了，因为内部也大都调用这几个函数来完成的
```java
    /**
     * Links e as first element. 让添加的元素成为第一个元素，也就是在头节点插入数据
     */
    private void linkFirst(E e) {
        // 头结点
        final Node<E> f = first; 
        //创建一个新的节点，next指向之前的头节点
        final Node<E> newNode = new Node<>(null, e, f);
        //让新节点成为头节点
        first = newNode;
        if (f == null) //如果之前的头节点为空，则给尾结点赋值为新的头节点
            last = newNode;
        else // 如果之前的头节点不为空，则让之前的头节点prev指向新的节点
            f.prev = newNode;
        size++; // 链表长度增加
        modCount++;
    }
```
```java
    /**
     * Links e as last element. 在链表最后添加元素
     */
    void linkLast(E e) {
        //记录尾结点
        final Node<E> l = last;
        //创建新节点，新节点的prev指向之前的尾结点
        final Node<E> newNode = new Node<>(l, e, null);
        //给新的尾节点赋值
        last = newNode;
        if (l == null) //如果链表之前的尾节点为空，则给头节点赋值（相当于插入第一个元素）
            first = newNode;
        else //否则，修改之前尾节点的next指针
            l.next = newNode;
        size++; //长度增加
        modCount++;
    }
```
以下几个看注释了解其作用，详细代码都不解释了，一看就懂
```java
    /**
     * Inserts element e before non-null Node succ.
     * 在不为null的succ节点前插入一个节点
     */
    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }
```
```java
    /**
     * Unlinks non-null first node f.
     * 删除第一个不为null的节点f，并返回其元素值
     */
    private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;
        f.item = null;
        f.next = null; // help GC 这儿注意一下，为了方便GC回收
        first = next;
        if (next == null)
            last = null;
        else
            next.prev = null;
        size--;
        modCount++;
        return element;
    }
```
```java
    /**
     * Unlinks non-null last node l.
     * 删除尾节点l，并返回其元素值
     */
    private E unlinkLast(Node<E> l) {
        // assert l == last && l != null;
        final E element = l.item;
        final Node<E> prev = l.prev;
        l.item = null;
        l.prev = null; // help GC
        last = prev;
        if (prev == null)
            first = null;
        else
            prev.next = null;
        size--;
        modCount++;
        return element;
    }
        /**
         * Unlinks non-null node x.
         * 删除节点x，里面也是操作队列
         */
        E unlink(Node<E> x) {
            ...省略
        }
```
```java
    //获取index位置的节点，可以看到它会判断index是在链表的前半部分还是后半部分
    Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) { //在前半部分，则从头开始遍历
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else { //在后半部分，则从尾部开始遍历
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```
这些代码都比较简单，下面比较常用的函数也是基于这几个简单的方法来实现的

### 四、LinkedList常用函数介绍
1. LinkedList构造函数
```java
    public LinkedList() {
    }

    /**
     * Constructs a list containing the elements of the specified
     * collection, in the order they are returned by the collection's
     * iterator.
     *
     * @param  c the collection whose elements are to be placed into this list
     * @throws NullPointerException if the specified collection is null
     */
    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }
```
可以看到有两个构造函数，一个无参构造，一个通过集合来进行构造。看注释可知通过集合构造时，如果传入的集合为空，则会抛出空指针。
另外没有向ArrayList一样有默认容量的构造函数，因为它是链表结构，不需要内存连续

2. 添加元素
```java
    // 在列表头部插入元素
    public void addFirst(E e) {
        linkFirst(e);
    }
    //在列表尾部增加元素，同add
    public void addLast(E e) {
        linkLast(e);
    }
    //添加元素
    public boolean add(E e) {
        linkLast(e);
        return true;
    }
    //添加集合中所有元素，追加到列表最后面
    public boolean addAll(Collection<? extends E> c) {
        return addAll(size, c);
    }
    //在指定位置添加集合元素
    public boolean addAll(int index, Collection<? extends E> c) {
        checkPositionIndex(index); //检查index合法性
        //集合转数组，如果长度为0，则直接返回失败
        Object[] a = c.toArray();
        int numNew = a.length;
        if (numNew == 0)
        return false;

        Node<E> pred, succ;
        if (index == size) { //如果index与size相等，则为在最后追加
          succ = null;
          pred = last;
        } else { //如果不相等则为在中间插入
          succ = node(index); //根据index找到节点信息
          pred = succ.prev;
        }

        //循环将节点插入
        for (Object o : a) {
          @SuppressWarnings("unchecked") E e = (E) o;
          Node<E> newNode = new Node<>(pred, e, null);
          if (pred == null)
            first = newNode;
          else
            pred.next = newNode;
          pred = newNode;
        }
        
        if (succ == null) { //如果succ为空，则pred就是尾节点
            last = pred;
        } else { //如果不为空，则将succ节点插入到最后一个节点后面
          pred.next = succ;
          succ.prev = pred;
        }

        size += numNew; //更新链表长度
        modCount++;
        return true;
    }

    //在指定位置插入元素，里面的方法都在之前介绍过
    public void add(int index, E element) {
        checkPositionIndex(index);

        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }
```
以及在1.5和1.6版本之后增加的添加元素的方法，也都比较简单,内部的函数调用也在前面介绍过
```java
    public boolean offer(E e) { //增加（追加）元素
        return add(e);
    }
    //在头部插入元素
    public boolean offerFirst(E e) {
        addFirst(e);
        return true;
    }
    //在尾部追加元素
    public boolean offerLast(E e) {
        addLast(e);
        return true;
    }
    //在头部插入元素
    public void push(E e) {
        addFirst(e);
    }
```
3. 获取元素
```java
    //获取第一个元素，如果不存在则抛出NoSuchElementException
    public E getFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return f.item;
    }
    //获取最后一个元素，如果不存在则抛出NoSuchElementException
    public E getLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return l.item;
    }
    //通过index获取节点内的元素
    public E get(int index) {
        checkElementIndex(index);
        return node(index).item;
    }
    //获取第一个元素，如果头节点为空则返回null
    public E peek() {
        final Node<E> f = first;
        return (f == null) ? null : f.item;
    }
    //获取第一个元素，与peek不同的是，poll返回元素的同时会把元素从列表中移除
    public E poll() {
        final Node<E> f = first;
        return (f == null) ? null : unlinkFirst(f);
    }
    //获取第一个元素，同getFirst()
    public E element() {
        return getFirst();
    }
    //同peek
    public E peekFirst() {
        final Node<E> f = first;
        return (f == null) ? null : f.item;
    }
    //与peekFirst相反，它获取的是链表中的最后一个元素
    public E peekLast() {
        final Node<E> l = last;
        return (l == null) ? null : l.item;
    }
    //同poll
    public E pollFirst() {
        final Node<E> f = first;
        return (f == null) ? null : unlinkFirst(f);
    }
    //与pollFirst相反，获取最后一个元素，并且会把它从链表中移除
    public E pollLast() {
        final Node<E> l = last;
        return (l == null) ? null : unlinkLast(l);
    }
```
4. 移除元素
```java
    //移除第一个元素并返回，如果头节点为null则抛出NoSuchElementException
    public E removeFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return unlinkFirst(f);
    }
    //移除最后一个元素并返回，如果尾节点为null则抛出NoSuchElementException
    public E removeLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return unlinkLast(l);
    }
    //移除元素，如果元素不存在则什么也不会改变
    public boolean remove(Object o) {
        if (o == null) { //元素为空的情况
          for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
              unlink(x);
              return true;
            }
          }
        } else { //元素不为空的情况
          for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
              unlink(x);
              return true;
            }
          }
        }
        return false;
    }
    //移除index位置元素
    public E remove(int index) {
        checkElementIndex(index);
        return unlink(node(index));
    }
    //移除第一个元素，同removeFirst
    public E remove() {
        return removeFirst();
    }
    //同removeFirst
    public E pop() {
        return removeFirst();
    }
    //移除第一个与元素o相同的元素，同remove
    public boolean removeFirstOccurrence(Object o) {
        return remove(o);
    }
    //从尾节点往前遍历，移除第一个节点内元素与o相同的节点
    public boolean removeLastOccurrence(Object o) {
        if (o == null) {
          for (Node<E> x = last; x != null; x = x.prev) {
            if (x.item == null) {
              unlink(x);
              return true;
            }
          }
        } else {
          for (Node<E> x = last; x != null; x = x.prev) {
            if (o.equals(x.item)) {
              unlink(x);
              return true;
            }
          }
        }
        return false;
    }
```
5. 更新元素
```java
    public E set(int index, E element) {
        //检测index合法性
        checkElementIndex(index);
        //找到index位置节点
        Node<E> x = node(index);
        //获取到之前的节点
        E oldVal = x.item;
        //修改节点内元素值
        x.item = element;
        //返回旧的节点
        return oldVal;
    }
```

### 五、LinkedList迭代器
```java
    /**
     * Returns a list-iterator of the elements in this list (in proper
     * sequence), starting at the specified position in the list.
     * Obeys the general contract of {@code List.listIterator(int)}.<p>
     *
     * The list-iterator is <i>fail-fast</i>: if the list is structurally
     * modified at any time after the Iterator is created, in any way except
     * through the list-iterator's own {@code remove} or {@code add}
     * methods, the list-iterator will throw a
     * {@code ConcurrentModificationException}.  Thus, in the face of
     * concurrent modification, the iterator fails quickly and cleanly, rather
     * than risking arbitrary, non-deterministic behavior at an undetermined
     * time in the future.
     *
     * @param index index of the first element to be returned from the
     *              list-iterator (by a call to {@code next})
     * @return a ListIterator of the elements in this list (in proper
     *         sequence), starting at the specified position in the list
     * @throws IndexOutOfBoundsException {@inheritDoc}
     * @see List#listIterator(int)
     */
    public ListIterator<E> listIterator(int index) {
        checkPositionIndex(index);
        return new ListItr(index);
    }
```
看官方注释，我们就可以知道他的作用了。它是从index位置开始返回一个迭代器.关于内部的迭代器类也不复杂就不过多介绍了，我们也简
单看一下，也是一些基本的链表操作
```java
    private class ListItr implements ListIterator<E> {
        private Node<E> lastReturned; //上一次返回的节点
        private Node<E> next;  //下一个节点
        private int nextIndex; //下一个节点索引
        private int expectedModCount = modCount; //期望的修改次数

        ListItr(int index) { 
            // assert isPositionIndex(index);
            //如果所给的索引与size相等，则说明是最后一个节点，next返回null，否则找到index处的节点
            next = (index == size) ? null : node(index);
            nextIndex = index;
        }
        //是否还有下一个节点
        public boolean hasNext() {
            return nextIndex < size;
        }
        //获取下一个节点元素
        public E next() {
            checkForComodification();
            if (!hasNext())
                throw new NoSuchElementException();

            lastReturned = next;
            next = next.next;
            nextIndex++;
            return lastReturned.item;
        }
        //是否有上一个节点
        public boolean hasPrevious() {
            return nextIndex > 0;
        }
        //获取上一个节点元素
        public E previous() {
            checkForComodification();
            if (!hasPrevious())
                throw new NoSuchElementException();

            lastReturned = next = (next == null) ? last : next.prev;
            nextIndex--;
            return lastReturned.item;
        }

        public int nextIndex() {
            return nextIndex;
        }

        public int previousIndex() {
            return nextIndex - 1;
        }
        //移除节点
        public void remove() {
            checkForComodification();
            if (lastReturned == null)
                throw new IllegalStateException();

            Node<E> lastNext = lastReturned.next;
            unlink(lastReturned);
            if (next == lastReturned)
                next = lastNext;
            else
                nextIndex--;
            lastReturned = null;
            expectedModCount++;
        }
        //修改当前遍历的节点元素值
        public void set(E e) {
            if (lastReturned == null)
                throw new IllegalStateException();
            checkForComodification();
            lastReturned.item = e;
        }
        //增加一个节点
        public void add(E e) {
            checkForComodification();
            lastReturned = null;
            if (next == null)
                linkLast(e);
            else
                linkBefore(e, next);
            nextIndex++;
            expectedModCount++;
        }
        //循环遍历
        public void forEachRemaining(Consumer<? super E> action) {
            Objects.requireNonNull(action);
            while (modCount == expectedModCount && nextIndex < size) {
                action.accept(next.item);
                lastReturned = next;
                next = next.next;
                nextIndex++;
            }
            checkForComodification();
        }
        //判断是否被其它线程改过，如果改过则抛异常，这就是我们有时候会遇到的问题（一个线程在使用迭代器遍历，另一个修改了它的值）
        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```
以上就是对LinkedList的简单介绍