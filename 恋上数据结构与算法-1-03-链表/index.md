# 恋上数据结构与算法-1-03-链表


## 链表（Linked List）

动态数组有个明显的缺点：可能会造成内存空间的大量浪费（每次扩容 1.5 倍）

能否用到多少就申请多少内存？链表可以做到这一点。

**链表**是一种**链式存储**的线性表，所有元素的内存地址不一定是连续的。

![链表](../../../../images/恋上算法与数据结构/1-03-链表/20200909-链表.png)  

## 自定义链表

### 自定义链表的接口设计

自定义动态数组与链表的对外接口是一样的，所以可以使用接口（`List<E>`）进行统一声明。

自定义动态数组和链表中存在可重用的公共代码，所以可以使用抽象类（`AbstractList<E>`）存放公共代码，同时该抽象类实现接口（`List<E>`）

自定义动态数组和链表继承抽象类（`AbstractList<E>`）

抽象类（`AbstractList<E>`）对外是不可见的，所以对外界可见的元素放在接口（`List<E>`）中，如 `ELEMENT_NOT_FOUND` 的声明。

![20200909-自定义链表的接口设计](../../../../images/恋上算法与数据结构/1-03-链表/20200909-自定义链表的接口设计.png)  


### 单向链表

![单向链表的结构](../../../../images/恋上算法与数据结构/1-03-链表/20200909-单向链表的结构.png)  

自定义数组和链表的公共接口

```java
public interface List<E> {

    static final int ELEMENT_NOT_FOUND = -1;

    /**
     * 清除所有元素
     */
    void clear();

    /**
     * 元素的数量
     * @return
     */
    int size();

    /**
     * 是否为空
     * @return
     */
    boolean isEmpty();

    /**
     * 是否包含某个元素
     * @param element
     * @return
     */
    boolean contains(E element);

    /**
     * 添加元素到尾部
     * @param element
     */
    void add(E element);

    /**
     * 获取index位置的元素
     * @param index
     * @return
     */
    E get(int index);

    /**
     * 设置index位置的元素
     * @param index
     * @param element
     * @return
     */
    E set(int index, E element);

    /**
     * 在index位置插入一个元素
     * @param index
     * @param element
     */
    void add(int index, E element);

    /**
     * 删除index位置的元素
     * @param index
     * @return
     */
    E remove(int index);

    /**
     * 查看元素的索引
     * @param element
     * @return
     */
    int indexOf(E element);
}
```

自定义动态数组和链表的抽象父类

```java
public abstract class AbstractList<E> implements List<E> {
    /**
     * 元素的数量
     */
    protected int size;

    /**
     * 获取元素的数量
     * @return
     */
    @Override
    public int size(){
        return size;
    }

    /**
     * 是否为空
     * @return
     */
    @Override
    public boolean isEmpty(){
        return size == 0;
    }

    /**
     * 是否包含某个元素
     * @param element
     * @return
     */
    @Override
    public boolean contains(E element){
        return indexOf(element) != ELEMENT_NOT_FOUND;
    }

    /**
     * 添加元素到尾部
     * @param element
     */
    @Override
    public void add(E element){
        add(size, element);
    }

    protected void outOfBounds(int index){
        throw new IndexOutOfBoundsException("Index: " + index + ", Size:" + size);
    }

    protected void rangeCheck(int index){
        if(index < 0 || index >= size){
            outOfBounds(index);
        }
    }

    protected void rangeCheckForAdd(int index){
        if(index < 0 || index > size){
            outOfBounds(index);
        }
    }
}
```

自定义动态数组

```java
public class ArrayList<E> extends AbstractList<E> {

    private E[] elements;
    private static final int DEFAULT_CAPACITY = 10;

    public ArrayList(int capacity) {
        capacity = (capacity < DEFAULT_CAPACITY) ? DEFAULT_CAPACITY : capacity;
        elements = (E[]) new Object[capacity];
    }

    public ArrayList() {
        this(DEFAULT_CAPACITY);
    }

    /**
     * 清除所有元素
     */
    @Override
    public void clear() {
        for (int i = 0; i < size; i++) {
            elements[i] = null;
        }
        size = 0;
    }

    /**
     * 获取index位置的元素
     * 该方法的时间复杂度为O(1)
     *
     * @param index
     * @return
     */
    @Override
    public E get(int index) {
        rangeCheck(index);
        return elements[index];  // O(1)  数组元素访问复杂度为O(1)
    }

    /**
     * 设置index位置的元素
     * 该方法的时间复杂度为O(1)
     *
     * @param index
     * @param element
     * @return
     */
    @Override
    public E set(int index, E element) {
        rangeCheck(index);
        E old = elements[index];
        elements[index] = element;
        return old;
    }

    /**
     * 在index位置插入一个元素
     * <p>
     * 时间复杂度，不包括ensureCapacity()
     * 最好： O(1)  往最后面添加元素
     * 最坏： O(n)  往0位置添加元素
     * 平均： O(n)  1+2+...+n=2/n，省略常数，得到评价复杂度为O(n)
     * size是数据规模
     *
     * @param index
     * @param element
     */
    @Override
    public void add(int index, E element) {
        rangeCheckForAdd(index);
        ensureCapacity(size + 1);

        for (int i = size; i > index; i--) {
            elements[i] = elements[i - 1];
        }
        elements[index] = element;
        size++;
    }

    /**
     * 删除index位置的元素
     * <p>
     * 时间复杂度：
     * 最好： O(1)
     * 最坏： O(n)
     * 平均： O(n)
     *
     * @param index
     * @return
     */
    @Override
    public E remove(int index) {
        rangeCheck(index);

        E old = elements[index];
        for (int i = index + 1; i < size; i++) {
            elements[i - 1] = elements[i];
        }
        elements[--size] = null;
        return old;
    }

    /**
     * 查看元素的索引
     *
     * @param element
     * @return
     */
    @Override
    public int indexOf(E element) {
        if (element == null) {
            for (int i = 0; i < size; i++) {
                if (elements[i] == null) return i;
            }
        } else {
            for (int i = 0; i < size; i++) {
                if (element.equals(elements[i])) return i;
            }
        }
        return ELEMENT_NOT_FOUND;
    }

    /**
     * 保证要有capacity的容量
     *
     * @param capacity
     */
    private void ensureCapacity(int capacity) {
        int oldCapacity = elements.length;
        if (oldCapacity >= capacity) return;

        //新容量为旧容量的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        E[] newElements = (E[]) new Object[newCapacity];
        for (int i = 0; i < size; i++) {
            newElements[i] = elements[i];
        }
        elements = newElements;

        System.out.println(oldCapacity + "扩容为：" + newCapacity);
    }

    @Override
    public String toString() {
        // size=3, [99, 88, 77]
        StringBuilder string = new StringBuilder();
        string.append("size=").append(size).append(", [");
        for (int i = 0; i < size; i++) {
            if (i != 0) {
                string.append(", ");
            }

            string.append(elements[i]);

			//if (i != size - 1) {
			//	string.append(", ");
			//}
        }
        string.append("]");
        return string.toString();
    }
}
```

自定义单向链表

```java
import org.msdemt.demo.AbstractList;

/**
 * 单向链表
 */
public class SingleLinkedList<E> extends AbstractList<E> {

    private Node<E> first;

    private static class Node<E> {
        E element;
        Node<E> next;

        public Node(E element, Node<E> next) {
            this.element = element;
            this.next = next;
        }
    }

    @Override
    public void clear() {
        size = 0;
        first = null;
    }

    /**
     * 时间复杂度:
     * 最好：O(1)
     * 最坏：O(n)
     * 平均：O(n)
     *
     * @param index
     * @return
     */
    @Override
    public E get(int index) {
        return node(index).element;
    }

    /**
     * 时间复杂度:
     * 最好：O(1)
     * 最坏：O(n)
     * 平均：O(n)
     *
     * @param index
     * @param element
     * @return
     */
    @Override
    public E set(int index, E element) {
        Node<E> node = node(index);
        E old = node.element;
        node.element = element;
        return old;
    }

    /**
     * 时间复杂度:
     * 最好：O(1)
     * 最坏：O(n)
     * 平均：O(n)
     * 
     * 网上说链表添加、删除的复杂度是O(1)，指的是添加时、删除时的那一刻的复杂度，
     * 即移动指针，我们封装的对外的接口的整体情况不是O(1)
     * @param index
     * @param element
     */
    @Override
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        if (index == 0) {
            first = new Node<>(element, first);
        } else {
            Node<E> prev = node(index - 1);
            prev.next = new Node<>(element, prev.next);
        }
        size++;
    }

    /**
     * 时间复杂度:
     * 最好：O(1)
     * 最坏：O(n)
     * 平均：O(n)
     *
     * @param index
     * @return
     */
    @Override
    public E remove(int index) {
        rangeCheck(index);

        Node<E> node = first;
        if (index == 0) {
            first = first.next;
        } else {
            Node<E> prev = node(index - 1);
            node = prev.next;
            prev.next = node.next;
        }
        size--;
        return node.element;
    }

    @Override
    public int indexOf(E element) {
        if (element == null) {
            Node<E> node = first;
            for (int i = 0; i < size; i++) {
                if (node.element == null) return i;
                node = node.next;
            }
        } else {
            Node<E> node = first;
            for (int i = 0; i < size; i++) {
                if (element.equals(node.element)) return i;
                node = node.next;
            }
        }
        return ELEMENT_NOT_FOUND;
    }

    /**
     * 获取index位置对应的节点对象
     * 最好 O(1)
     * 最坏 O(n)
     * @param index
     * @return
     */
    private Node<E> node(int index) {
        rangeCheck(index);

        Node<E> node = first;
        for (int i = 0; i < index; i++) {
            node = node.next;
        }
        return node;
    }

    @Override
    public String toString() {
        StringBuilder string = new StringBuilder();
        string.append("size=").append(size).append(", [");
        Node<E> node = first;
        for (int i = 0; i < size; i++) {
            if (i != 0) {
                string.append(", ");
            }
            string.append(node.element);
            node = node.next;
        }
        string.append("]");

        //另一种遍历的方式
        //Node<E> node1 = first;
        //while (node1 != null) {
        //	node1 = node1.next;
        //}
        return string.toString();
    }
}
```

动态数组、链表复杂度对比

![动态数组和链表复杂度对比](../../../../images/恋上算法与数据结构/1-03-链表/20200914-动态数组和链表复杂度对比.png)  

#### 均摊复杂度

动态数组 add(E element) 的复杂度分析

```java
/**
 * 添加元素到尾部
 *
 * @param element
 */
@Override
public void add(E element) {
    add(size, element);
}
```

该方法是往数组的最后面添加元素，此时不需要挪动元素，此时最好复杂度为O(1)，当产生扩容是，会有最坏复杂度

```java
/**
 * 保证要有capacity的容量
 *
 * @param capacity
 */
private void ensureCapacity(int capacity) {
    int oldCapacity = elements.length;
    if (oldCapacity >= capacity) return;

    //新容量为旧容量的1.5倍
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    E[] newElements = (E[]) new Object[newCapacity];
    for (int i = 0; i < size; i++) {
        newElements[i] = elements[i];
    }
    elements = newElements;

    System.out.println(oldCapacity + "扩容为：" + newCapacity);
}
```

扩容时会将所有元素挪到新的数组，所以此时的复杂度为O(n)。

绝大数情况的复杂度为O(1)，当扩容时的复杂度为O(n)，此时平均的复杂度为多少呢？

1 + 1 + 1 + ... + 1 + n

**平均复杂度**为 O(1)

此时**均摊复杂度**也为O(1)

![均摊复杂度](../../../../images/恋上算法与数据结构/1-03-链表/20200914-均摊复杂度.png)  

什么情况下使用均摊复杂度？

经过连续的多次复杂度比较低的情况后，出现个别复杂度比较高的情况

#### 虚拟头节点（了解）

有时候为了让代码更加精简，统一所有节点的处理逻辑，可以在最前面增加一个虚拟的头节点（不存储数据）

![虚拟头节点](../../../../images/恋上算法与数据结构/1-03-链表/20200914-虚拟头节点.png)  

对上文中的自定义单向链表添加虚拟头节点

```java
import org.msdemt.demo.AbstractList;

/**
 * 单向链表增加虚拟头节点
 */
public class SingleLinkedList2<E> extends AbstractList<E> {

    private Node<E> first;

    private static class Node<E> {
        E element;
        Node<E> next;

        public Node(E element, Node<E> next) {
            this.element = element;
            this.next = next;
        }
    }

    public SingleLinkedList2() {
        first = new Node<>(null, null); //虚拟头节点
    }

    @Override
    public void clear() {
        size = 0;
        first = null;
    }

    @Override
    public E get(int index) {
        return node(index).element;
    }

    @Override
    public E set(int index, E element) {
        Node<E> node = node(index);
        E old = node.element;
        ;
        node.element = element;
        return old;
    }

    @Override
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        Node<E> prev = index == 0 ? first : node(index - 1);
        prev.next = new Node<>(element, prev.next);
        size++;
    }

    @Override
    public E remove(int index) {
        rangeCheck(index);

        Node<E> prev = index == 0 ? first : node(index - 1);
        Node<E> node = prev.next;
        prev.next = node.next;
        size--;
        return node.element;
    }

    @Override
    public int indexOf(E element) {
        if (element == null) {
            Node<E> node = first;
            for (int i = 0; i < size; i++) {
                if (node.element == null) return i;
                node = node.next;
            }
        } else {
            Node<E> node = first;
            for (int i = 0; i < size; i++) {
                if (element.equals(node.element)) return i;
                node = node.next;
            }
        }
        return ELEMENT_NOT_FOUND;
    }

    /**
     * 获取index位置对应的节点对象
     *
     * @param index
     * @return
     */
    private Node<E> node(int index) {
        rangeCheck(index);

        Node<E> node = first.next;
        for (int i = 0; i < index; i++) {
            node = node.next;
        }
        return node;
    }

    @Override
    public String toString() {
        StringBuilder string = new StringBuilder();
        string.append("size=").append(size).append(", [");
        Node<E> node = first.next;
        for (int i = 0; i < size; i++) {
            if (i != 0) {
                string.append(", ");
            }

            string.append(node.element);

            node = node.next;
        }
        string.append("]");

		//Node<E> node1 = first;
        //while (node1 != null) {
        //
        //
        //	node1 = node1.next;
        //}
        return string.toString();
    }
}
```

#### 动态数组的缩容

如果内存使用比较紧张，动态数组有比较多的剩余空间，可以考虑进行缩容操作
- 比如剩余空间占总容量的一半时，就进行缩容

删除元素时触发缩容操作

```java
/**
 * 具有动态缩容功能的动态数组
 */
public class ArrayList2<E> extends AbstractList<E> {

    private E[] elements;
    private static final int DEFAULT_CAPACITY = 10;

    private ArrayList2(int capacity) {
        capacity = (capacity < DEFAULT_CAPACITY) ? DEFAULT_CAPACITY : capacity;
        elements = (E[]) new Object[capacity];
    }

    private ArrayList2() {
        this(DEFAULT_CAPACITY);
    }

    @Override
    public void clear() {
        for (int i = 0; i < size; i++) {
            elements[i] = null;
        }
        size = 0;

        //缩容参考
        if (elements != null && elements.length > DEFAULT_CAPACITY) {
            elements = (E[]) new Object[DEFAULT_CAPACITY];
        }
    }

    @Override
    public E get(int index) {
        rangeCheck(index);
        return elements[index];
    }

    @Override
    public E set(int index, E element) {
        rangeCheck(index);

        E old = elements[index];
        elements[index] = element;
        return old;
    }

    @Override
    public void add(int index, E element) {
        rangeCheckForAdd(index);
        ensureCapacity(size + 1);
        for (int i = size; i > index; i--) {
            elements[i] = elements[i - 1];
        }
        elements[index] = element;
        size++;
    }

    @Override
    public E remove(int index) {
        rangeCheck(index);

        E old = elements[index];
        for (int i = index + 1; i < size; i++) {
            elements[i - 1] = elements[i];
        }
        elements[--size] = null;

        //缩容参考
        trim();

        return old;
    }

    @Override
    public int indexOf(E element) {
        if (element == null) {
            for (int i = 0; i < size; i++) {
                if (elements[i] == null) return i;
            }
        } else {
            for (int i = 0; i < size; i++) {
                if (element.equals(elements[i])) return i;
            }
        }
        return ELEMENT_NOT_FOUND;
    }

    private void ensureCapacity(int capacity) {
        int oldCapacity = elements.length;
        if (oldCapacity >= capacity) return;

        //新容量为旧容量的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        //新容量为旧容量的2倍
        //int newCapacity = oldCapacity << 1;

        E[] newElements = (E[]) new Object[newCapacity];
        for (int i = 0; i < size; i++) {
            newElements[i] = elements[i];
        }
        elements = newElements;
        System.out.println(oldCapacity + "扩容为：" + newCapacity);
    }

    private void trim() {
        int oldCapacity = elements.length;
        int newCapacity = oldCapacity >> 1;
        if (size >= newCapacity || oldCapacity <= DEFAULT_CAPACITY) return;

        E[] newElements = (E[]) new Object[newCapacity];
        for (int i = 0; i < size; i++) {
            newElements[i] = elements[i];
        }
        elements = newElements;
        System.out.println(oldCapacity + "缩容为：" + newCapacity);
    }

    @Override
    public String toString() {
        // size=3, [99, 88, 77]
        StringBuilder string = new StringBuilder();
        string.append("size=").append(size).append(", [");
        for (int i = 0; i < size; i++) {
            if (i != 0) {
                string.append(", ");
            }

            string.append(elements[i]);

			//if (i != size - 1) {
            //	string.append(", ");
            //}
        }
        string.append("]");
        return string.toString();
    }
}
```

如果扩容倍数、缩容时机设计不得当，有可能导致**复杂度震荡**。

```java
private void ensureCapacity(int capacity) {
    int oldCapacity = elements.length;
    if (oldCapacity >= capacity) return;

    //新容量为旧容量的1.5倍
    //int newCapacity = oldCapacity + (oldCapacity >> 1);
    //新容量为旧容量的2倍
    int newCapacity = oldCapacity << 1;

    E[] newElements = (E[]) new Object[newCapacity];
    for (int i = 0; i < size; i++) {
        newElements[i] = elements[i];
    }
    elements = newElements;
    System.out.println(oldCapacity + "扩容为：" + newCapacity);
}

private void trim() {
    int oldCapacity = elements.length;
    int newCapacity = oldCapacity >> 1;
    if (size > newCapacity || oldCapacity <= DEFAULT_CAPACITY) return;

    E[] newElements = (E[]) new Object[newCapacity];
    for (int i = 0; i < size; i++) {
        newElements[i] = elements[i];
    }
    elements = newElements;
    System.out.println(oldCapacity + "缩容为：" + newCapacity);
}
```

假设，扩容时为原来的2倍，缩容时减少一半，同时缩容时若size等于newCapacity也会出发缩容（`if (size >= newCapacity || oldCapacity <= DEFAULT_CAPACITY) return;` 改为 `if (size > newCapacity || oldCapacity <= DEFAULT_CAPACITY) return;`）

![复杂度震荡1](../../../../images/恋上算法与数据结构/1-03-链表/20200914-复杂度震荡1.png) 

![复杂度震荡2](../../../../images/恋上算法与数据结构/1-03-链表/20200914-复杂度震荡2.png)  


假设动态数组长度为4，此时容量也为4，添加一个元素55，会出发扩容操作，长度变为8，这个操作的复杂度为O(n)，再删除添加的元素55，会出发缩容操作，这个删除操作的时间复杂度为O(n)，不断添加删除位置5的元素，复杂度会保持在O(n)

复杂度平常很低，针对某个操作频繁进行，导致复杂度一直处于很高的状态，这种情况叫**复杂度震荡**。

解决办法：扩容的倍数和缩容的倍数相乘不要等于1


### 双向链表

使用双向链表可以提升链表的综合性能

![双向链表](../../../../images/恋上算法与数据结构/1-03-链表/20200914-双向链表.png)  

自定义双向链表实现

```java
/**
 * 双向链表
 */
public class LinkedList<E> extends AbstractList<E> {
    private Node<E> first;
    private Node<E> last;

    private static class Node<E> {
        E element;
        Node<E> prev;
        Node<E> next;

        public Node(Node<E> prev, E element, Node<E> next) {
            this.prev = prev;
            this.element = element;
            this.next = next;
        }

        @Override
        public String toString() {
            StringBuilder sb = new StringBuilder();

            if (prev != null) {
                sb.append(prev.element);
            } else {
                sb.append("null");
            }

            sb.append("_").append(element).append("_");

            if (next != null) {
                sb.append(next.element);
            } else {
                sb.append("null");
            }

            return sb.toString();
        }
    }

    @Override
    public void clear() {
        size = 0;
        first = null;
        last = null;
    }

    @Override
    public E get(int index) {
        return node(index).element;
    }

    @Override
    public E set(int index, E element) {
        Node<E> node = node(index);
        E old = node.element;
        node.element = element;
        return old;
    }

    @Override
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        // size == 0
        // index == 0
        if (index == size) { // 往最后面添加元素
            Node<E> oldLast = last;
            last = new Node<>(oldLast, element, null);
            if (oldLast == null) { // 这是链表添加的第一个元素
                first = last;
            } else {
                oldLast.next = last;
            }
        } else {
            Node<E> next = node(index);
            Node<E> prev = next.prev;
            Node<E> node = new Node<>(prev, element, next);
            next.prev = node;

            if (prev == null) { // index == 0
                first = node;
            } else {
                prev.next = node;
            }
        }

        size++;
    }

    @Override
    public E remove(int index) {
        rangeCheck(index);

        Node<E> node = node(index);
        Node<E> prev = node.prev;
        Node<E> next = node.next;

        if (prev == null) { // index == 0
            first = next;
        } else {
            prev.next = next;
        }

        if (next == null) { // index == size - 1
            last = prev;
        } else {
            next.prev = prev;
        }

        size--;
        return node.element;
    }

    @Override
    public int indexOf(E element) {
        if (element == null) {
            Node<E> node = first;
            for (int i = 0; i < size; i++) {
                if (node.element == null) return i;

                node = node.next;
            }
        } else {
            Node<E> node = first;
            for (int i = 0; i < size; i++) {
                if (element.equals(node.element)) return i;

                node = node.next;
            }
        }
        return ELEMENT_NOT_FOUND;
    }

    /**
     * 获取index位置对应的节点对象
     *
     * @param index
     * @return
     */
    private Node<E> node(int index) {
        rangeCheck(index);

        if (index < (size >> 1)) {
            Node<E> node = first;
            for (int i = 0; i < index; i++) {
                node = node.next;
            }
            return node;
        } else {
            Node<E> node = last;
            for (int i = size - 1; i > index; i--) {
                node = node.prev;
            }
            return node;
        }
    }

    @Override
    public String toString() {
        StringBuilder string = new StringBuilder();
        string.append("size=").append(size).append(", [");
        Node<E> node = first;
        for (int i = 0; i < size; i++) {
            if (i != 0) {
                string.append(", ");
            }

            string.append(node);
            node = node.next;
        }
        string.append("]");
        return string.toString();
    }
}
```

#### 双向链表 vs 单向链表

![双向链表vs单向链表](../../../../images/恋上算法与数据结构/1-03-链表/20200914-双向链表vs单向链表.png)  

#### 双向链表 vs 动态数组

**动态数组**：开辟、销毁内存空间的次数相对较少，但可能造成内存空间浪费（可以通过缩容解决）

**双向链表**：开辟、销毁内存空间的次数相对较多，但不会造成内存空间的浪费

- 如果频繁在尾部进行添加、删除操作，动态数组、双向链表均可选择
- 如果频繁在头部进行添加、删除操作，建议选择使用双向链表
- 如果有频繁的（在任意位置）添加、删除操作，建议选择使用双向链表
- 如果有频繁的查询操作（随机访问操作），建议选择使用动态数组

有了双向链表，单向链表是否就没有任何用处了？

并非如此，在哈希表的设计中就用到了单链表

### 单向循环链表

![单向循环链表](../../../../images/恋上算法与数据结构/1-03-链表/20200914-单向循环链表.png)  

单向循环链表的实现

```java
import org.msdemt.demo.AbstractList;

/**
 * 单向循环链表
 *
 * @param <E>
 */
public class SingleCircleLinkedList<E> extends AbstractList<E> {

    private Node<E> first;

    private static class Node<E> {
        E element;
        Node<E> next;

        public Node(E element, Node<E> next) {
            this.element = element;
            this.next = next;
        }

        @Override
        public String toString() {
            StringBuilder sb = new StringBuilder();
            sb.append(element).append("_").append(next.element);
            return sb.toString();
        }
    }

    @Override
    public void clear() {
        size = 0;
        first = null;
    }

    @Override
    public E get(int index) {
        return node(index).element;
    }

    @Override
    public E set(int index, E element) {
        Node<E> node = node(index);
        E old = node.element;
        node.element = element;
        return old;
    }

    @Override
    public void add(int index, E element) {
        rangeCheck(index);
        if (index == 0) {
            Node<E> newFirst = new Node<>(element, first);
            Node<E> last = (size == 0) ? newFirst : node(size - 1);
            last.next = newFirst;
            first = newFirst;
        } else {
            Node<E> prev = node(index - 1);
            prev.next = new Node<>(element, prev.next);
        }
        size++;
    }

    @Override
    public E remove(int index) {
        rangeCheck(index);

        Node<E> node = first;
        if (index == 0) {
            if (size == 1) {
                first = null;
            } else {
                Node<E> last = node(size - 1);
                first = first.next;
                last.next = first;
            }
        } else {
            Node<E> prev = node(index - 1);
            node = prev.next;
            prev.next = node.next;
        }
        size--;
        return node.element;
    }

    @Override
    public int indexOf(E element) {
        if (element == null) {
            Node<E> node = first;
            for (int i = 0; i < size; i++) {
                if (node.element == null) return i;
                node = node.next;
            }
        } else {
            Node<E> node = first;
            for (int i = 0; i < size; i++) {
                if (element.equals(node.element)) return i;
                node = node.next;
            }
        }
        return ELEMENT_NOT_FOUND;
    }

    private Node<E> node(int index) {
        rangeCheck(index);

        Node<E> node = first;
        for (int i = 0; i < index; i++) {
            node = node.next;
        }
        return node;
    }

    @Override
    public String toString() {
        StringBuilder string = new StringBuilder();
        string.append("size=").append(size).append(", [");
        Node<E> node = first;
        for (int i = 0; i < size; i++) {
            if (i != 0) {
                string.append(", ");
            }
            string.append(node);
            node = node.next;
        }
        string.append("]");
        return string.toString();
    }
}
```

### 双向循环链表

![双向循环链表](../../../../images/恋上算法与数据结构/1-03-链表/20200914-双向循环链表.png)  

双向循环链表的实现

```java
import org.msdemt.demo.AbstractList;

/**
 * 双向循环链表
 *
 * @param <E>
 */
public class CircleLinkedList<E> extends AbstractList<E> {

    private Node<E> first;
    private Node<E> last;
    private Node<E> current;

    private static class Node<E> {
        E element;
        Node<E> prev;
        Node<E> next;

        public Node(Node<E> prev, E element, Node<E> next) {
            this.prev = prev;
            this.element = element;
            this.next = next;
        }

        @Override
        public String toString() {
            StringBuilder sb = new StringBuilder();

            if (prev != null) {
                sb.append(prev.element);
            } else {
                sb.append("null");
            }

            sb.append("_").append(element).append("_");

            if (next != null) {
                sb.append(next.element);
            } else {
                sb.append("null");
            }

            return sb.toString();
        }
    }

    public void reset() {
        current = first;
    }

    public E next() {
        if (current == null) return null;
        current = current.next;
        return current.element;
    }

    public E remove() {
        if (current == null) return null;

        Node<E> next = current.next;
        E element = remove(current);
        if (size == 0) {
            current = null;
        } else {
            current = next;
        }
        return element;
    }


    @Override
    public void clear() {
        size = 0;
        first = null;
        last = null;
    }

    @Override
    public E get(int index) {
        return node(index).element;
    }

    @Override
    public E set(int index, E element) {
        Node<E> node = node(index);
        E old = node.element;
        node.element = element;
        return old;
    }

    @Override
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        if (index == size) {    //往最后添加元素
            Node<E> oldLast = last;
            last = new Node<>(oldLast, element, first);
            if (oldLast == null) {  //链表添加的第一个元素
                first = last;
                first.next = first;
                first.prev = first;
            } else {
                oldLast.next = last;
                first.prev = last;
            }
        } else {
            Node<E> next = node(index);
            Node<E> prev = next.prev;
            Node<E> node = new Node<>(prev, element, next);
            next.prev = node;
            prev.next = node;
            if (next == first) {    //index == 0
                first = node;
            }
        }
        size++;
    }

    @Override
    public E remove(int index) {
        rangeCheck(index);
        return remove(node(index));
    }

    @Override
    public int indexOf(E element) {
        if (element == null) {
            Node<E> node = first;
            for (int i = 0; i < size; i++) {
                if (node.element == null) return i;

                node = node.next;
            }
        } else {
            Node<E> node = first;
            for (int i = 0; i < size; i++) {
                if (element.equals(node.element)) return i;

                node = node.next;
            }
        }
        return ELEMENT_NOT_FOUND;
    }

    private E remove(Node<E> node) {
        if (size == 1) {
            first = null;
            last = null;
        } else {
            Node<E> prev = node.prev;
            Node<E> next = node.next;
            prev.next = next;
            next.prev = prev;

            if (node == first) {  //index == 0
                first = next;
            }

            if (node == last) {   //index =- size -1
                last = prev;
            }
        }
        size--;
        return node.element;
    }

    private Node<E> node(int index) {
        rangeCheck(index);

        if (index < (size >> 1)) {
            Node<E> node = first;
            for (int i = 0; i < index; i++) {
                node = node.next;
            }
            return node;
        } else {
            Node<E> node = last;
            for (int i = size - 1; i > index; i--) {
                node = node.prev;
            }
            return node;
        }
    }

    @Override
    public String toString() {
        StringBuilder string = new StringBuilder();
        string.append("size=").append(size).append(", [");
        Node<E> node = first;
        for (int i = 0; i < size; i++) {
            if (i != 0) {
                string.append(", ");
            }
            string.append(node);
            node = node.next;
        }
        string.append("]");
        return string.toString();
    }
}
```

### 使用循环链表解决约瑟夫问题

![约瑟夫问题](../../../../images/恋上算法与数据结构/1-03-链表/20200914-约瑟夫问题.png)  

如何发挥循环链表的最大威力？

可以考虑增设1个成员变量、3个方法
- `current`：用于指向某个节点
- `void reset()`：让 current 指向头节点 first
- `E next()`：让 current 往后走一步，也就是 current = current.next
- `E remove()`：删除 current 指向的节点，删除成功后让 current 指向下一个节点

![增强循环链表](../../../../images/恋上算法与数据结构/1-03-链表/20200914-增强循环链表.png)  

上文双向循环链表已增设了1个成员变量、3个方法

也可以在单向循环链表增设如上1个成员变量、3个方法来解决约瑟夫问题。

```java
static void josephus() {
	CircleLinkedList<Integer> list = new CircleLinkedList<>();
	for (int i = 1; i <= 8; i++) {
		list.add(i);
	}
	
	// 指向头结点（指向1）
	list.reset();
	
	while (!list.isEmpty()) {
		list.next();
		list.next();
		System.out.println(list.remove());
	}
}

public static void main(String[] args) {
    josephus();
}
```

### 静态链表

前面所学习的链表，是依赖于指针（引用）实现的

有些编程语言是没有指针的，如早期的 BASIC、FORTRAN 语言

没有指针的情况下，如何实现链表？
- 可以通过数组来模拟链表，称为静态链表
- 数组的每个元素存放两个数据：值、下个元素的索引
- 数组 0 位置存放的是头结点信息

![静态链表1](../../../../images/恋上算法与数据结构/1-03-链表/20200914-静态链表1.png)  

![静态链表2](../../../../images/恋上算法与数据结构/1-03-链表/20200914-静态链表2.png)  

思考：如果数组的每个元素只能存放1个数据呢？
- 那就使用 2 个数组，1 个数组存放索引关系，1 个数组存放值。

### ArrayList 能否进一步优化？

![arraylist优化](../../../../images/恋上算法与数据结构/1-03-链表/20200915-arraylist优化.png)  

增加或删除首元素时，更改first指针即可。

增加或删除中间元素最多挪动一半的元素


