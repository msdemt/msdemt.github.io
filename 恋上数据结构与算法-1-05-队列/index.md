# 恋上数据结构与算法-1-05-队列


## 队列（Queue）

队列是一种特殊的线性表，只能在头尾两端进行操作
- 队尾（rear）：只能从队尾添加元素，一般叫做 enQueue ，入队
- 队头（front）：只能从队头移除元素，一般叫做 deQueue ，出队
- 先进先出的原则，First In First Out，FIFO

![队列](../../../../images/恋上算法与数据结构/1-05-队列/20200915-队列.png)  

### 队列的接口设计

```java
int size();  //元素的数量
boolean isEmpty();  //是否为空
void enQueue(E element);  //入队
E deQueue();  //出队
E front();  //获取队列的头元素
void clear();  //清空
```

队列的内部实现是否可以直接利用以前学过的数据结构？
- 动态数组、链表
- 优先使用**双向链表**，因为队列主要是往**头尾**操作元素

### 自定义队列的实现

使用双向链表实现队列，首先将以前的自定义双向链表拷贝过来

```java
package org.msdemt.demo.list;

public interface List<E> {
    static final int ELEMENT_NOT_FOUND = -1;

    /**
     * 清除所有元素
     */
    void clear();

    /**
     * 元素的数量
     *
     * @return
     */
    int size();

    /**
     * 是否为空
     *
     * @return
     */
    boolean isEmpty();

    /**
     * 是否包含某个元素
     *
     * @param element
     * @return
     */
    boolean contains(E element);

    /**
     * 添加元素到尾部
     *
     * @param element
     */
    void add(E element);

    /**
     * 获取index位置的元素
     *
     * @param index
     * @return
     */
    E get(int index);

    /**
     * 设置index位置的元素
     *
     * @param index
     * @param element
     * @return 原来的元素ֵ
     */
    E set(int index, E element);

    /**
     * 在index位置插入一个元素
     *
     * @param index
     * @param element
     */
    void add(int index, E element);

    /**
     * 删除index位置的元素
     *
     * @param index
     * @return
     */
    E remove(int index);

    /**
     * 查看元素的索引
     *
     * @param element
     * @return
     */
    int indexOf(E element);
}
```

```java
package org.msdemt.demo.list;

public abstract class AbstractList<E> implements List<E> {
    /**
     * 元素的数量
     */
    protected int size;

    /**
     * 元素的数量
     *
     * @return
     */
    public int size() {
        return size;
    }

    /**
     * 是否为空
     *
     * @return
     */
    public boolean isEmpty() {
        return size == 0;
    }

    /**
     * 是否包含某个元素
     *
     * @param element
     * @return
     */
    public boolean contains(E element) {
        return indexOf(element) != ELEMENT_NOT_FOUND;
    }

    /**
     * 添加元素到尾部
     *
     * @param element
     */
    public void add(E element) {
        add(size, element);
    }

    protected void outOfBounds(int index) {
        throw new IndexOutOfBoundsException("Index:" + index + ", Size:" + size);
    }

    protected void rangeCheck(int index) {
        if (index < 0 || index >= size) {
            outOfBounds(index);
        }
    }

    protected void rangeCheckForAdd(int index) {
        if (index < 0 || index > size) {
            outOfBounds(index);
        }
    }
}
```

```java
package org.msdemt.demo.list;

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

利用双向链表实现队列

```java
package org.msdemt.demo;

import org.msdemt.demo.list.LinkedList;
import org.msdemt.demo.list.List;

/**
 * 使用双向链表实现队列
 *
 * @param <E>
 */
public class Queue<E> {
    private List<E> list = new LinkedList<>();

    public int size() {
        return list.size();
    }

    public boolean isEmpty() {
        return list.isEmpty();
    }

    public void clear() {
        list.clear();
    }

    public void enQueue(E element) {
        list.add(element);
    }

    public E deQueue() {
        return list.remove(0);
    }

    public E front() {
        return list.get(0);
    }
}
```

### java官方的队列

java.util.Queue

## 双端队列（Deque）

**双端队列**是能在头尾两端添加、删除的队列
- 英文 deque 是 double ended queue 的简称

![双端队列](../../../../images/恋上算法与数据结构/1-05-队列/20200915-双端队列.png)  

### 双端队列的接口设计

```java
int size();  //元素的数量
boolean isEmpty();  //是否为空
void enQueueRear(E element);  //从队尾入队
E deQueueFront();  //从队头出队
void enQueueFront(E element);  //从队头入队
E deQueueRear();  //从队尾出队
E front();  //获取队列的头元素
E rear();  //获取队列的尾元素
```

### 自定义双端队列的实现

使用LinkedList实现双端队列

```java
package org.msdemt.demo;

import org.msdemt.demo.list.LinkedList;
import org.msdemt.demo.list.List;

/**
 * 使用双向链表实现双端队列
 */
public class Deque<E> {

    private List<E> list = new LinkedList<>();

    public int size() {
        return list.size();
    }

    public boolean isEmpty() {
        return list.isEmpty();
    }

    public void clear() {
        list.clear();
    }

    /**
     * 队尾入队列
     *
     * @param element
     */
    public void enQueueRear(E element) {
        list.add(element);
    }

    /**
     * 队头出队列
     *
     * @return
     */
    public E deQueueFront() {
        return list.remove(0);
    }

    /**
     * 队头入队列
     *
     * @param element
     */
    public void enQueueFront(E element) {
        list.add(0, element);
    }

    /**
     * 队尾出队列
     *
     * @return
     */
    public E deQueueRear() {
        return list.remove(list.size() - 1);
    }

    public E front() {
        return list.get(0);
    }

    public E rear() {
        return list.get(list.size() - 1);
    }

}
```

## 循环队列（Circle Queue）

其实队列底层也可以使用**动态数组**实现，并且各项接口也可以优化到 O(1) 的时间复杂度，这个用数组实现并且优化之后的队列也叫做：**循环队列**

**循环队列**底层用数组实现

![循环队列1](../../../../images/恋上算法与数据结构/1-05-队列/20200915-循环队列1.png)  

![循环队列2](../../../../images/恋上算法与数据结构/1-05-队列/20200915-循环队列2.png)  

动态数组实现循环队列

```java
package org.msdemt.demo.circle;

public class CircleQueue<E> {

    private int front;
    private int size;
    private E[] elements;
    private static final int DEFAULT_CAPACITY = 10;

    public CircleQueue() {
        elements = (E[]) new Object[DEFAULT_CAPACITY];
    }

    public int size() {
        return size;
    }

    public boolean isEmpty() {
        return size == 0;
    }

    public void clear() {
        for (int i = 0; i < size; i++) {
            elements[index(i)] = null;
        }
        front = 0;
        size = 0;
    }

    public void enQueue(E element) {
        ensureCapacity(size + 1);
        elements[index(size)] = element;
        size++;
    }

    public E deQueue() {
        E frontElement = elements[front];
        elements[front] = null;
        front = index(1);
        size--;
        return frontElement;
    }

    public E front() {
        return elements[front];
    }

    /**
     * 获取index索引位置在数组中真实的坐标
     *
     * @param index
     * @return
     */
    private int index(int index) {
        index += front;
        return index - (index >= elements.length ? elements.length : 0);
        //return (front + index) % elements.length;
    }

    /**
     * 保证数组中有capacity的容量
     *
     * @param capacity
     */
    private void ensureCapacity(int capacity) {
        int oldCapacity = elements.length;
        if (oldCapacity >= capacity) return;

        // 新容量为旧容量的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        E[] newElements = (E[]) new Object[newCapacity];
        for (int i = 0; i < size; i++) {
            newElements[i] = elements[index(i)];
        }
        elements = newElements;

        // 重置front
        front = 0;
    }

    @Override
    public String toString() {
        StringBuilder string = new StringBuilder();
        string.append("capcacity=").append(elements.length)
                .append(" size=").append(size)
                .append(" front=").append(front)
                .append(", [");
        for (int i = 0; i < elements.length; i++) {
            if (i != 0) {
                string.append(", ");
            }

            string.append(elements[i]);
        }
        string.append("]");
        return string.toString();
    }
}
```


**循环双端队列**：可以进行两端添加、删除操作的循环队列

动态数组实现循环双端队列

```java
package org.msdemt.demo.circle;

public class CircleDeque<E> {

    private int front;
    private int size;
    private E[] elements;
    private static final int DEFAULT_CAPACITY = 10;

    public CircleDeque() {
        elements = (E[]) new Object[DEFAULT_CAPACITY];
    }

    public int size() {
        return size;
    }

    public boolean isEmpty() {
        return size == 0;
    }

    public void clear() {
        for (int i = 0; i < size; i++) {
            elements[index(i)] = null;
        }
        front = 0;
        size = 0;
    }

    /**
     * 从尾部入队
     *
     * @param element
     */
    public void enQueueRear(E element) {
        ensureCapacity(size + 1);

        elements[index(size)] = element;
        size++;
    }

    /**
     * 从头部出队
     *
     * @param
     */
    public E deQueueFront() {
        E frontElement = elements[front];
        elements[front] = null;
        front = index(1);
        size--;
        return frontElement;
    }

    /**
     * 从头部入队
     *
     * @param element
     */
    public void enQueueFront(E element) {
        ensureCapacity(size + 1);

        front = index(-1);
        elements[front] = element;
        size++;
    }

    /**
     * 从尾部出队
     *
     * @param
     */
    public E deQueueRear() {
        int rearIndex = index(size - 1);
        E rear = elements[rearIndex];
        elements[rearIndex] = null;
        size--;
        return rear;
    }

    public E front() {
        return elements[front];
    }

    public E rear() {
        return elements[index(size - 1)];
    }

    @Override
    public String toString() {
        StringBuilder string = new StringBuilder();
        string.append("capcacity=").append(elements.length)
                .append(" size=").append(size)
                .append(" front=").append(front)
                .append(", [");
        for (int i = 0; i < elements.length; i++) {
            if (i != 0) {
                string.append(", ");
            }

            string.append(elements[i]);
        }
        string.append("]");
        return string.toString();
    }

    private int index(int index) {
        index += front;
        if (index < 0) {
            return index + elements.length;
        }
        return index - (index >= elements.length ? elements.length : 0);
    }

    /**
     * 保证要有capacity的容量
     *
     * @param capacity
     */
    private void ensureCapacity(int capacity) {
        int oldCapacity = elements.length;
        if (oldCapacity >= capacity) return;

        // 新容量为旧容量的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        E[] newElements = (E[]) new Object[newCapacity];
        for (int i = 0; i < size; i++) {
            newElements[i] = elements[index(i)];
        }
        elements = newElements;

        // 重置front
        front = 0;
    }
}
```


## % 运算符优化

尽量避免使用乘（*）、除（/）、模（%）、浮点数运算，效率低下

将 `(front + index) % elements.length` 优化为 `index - (index >= elements.length ? elements.length : 0)`


已知 `n>=0, m>0`
- `n%m` 等价于 `n-(m>n?0:m)` 的前提条件：`n<2m`

```java
public static void main(String[] args) {
	
	int n = 13;
	int m = 7;
	
	if (n >= m) {
		System.out.println(n - m);
	} else {
		System.out.println(n);
	}
	
	// m > 0, n >= 0, n < 2m
	System.out.println(n - (n >= m ? m : 0));
	
	System.out.println(n % m);
}
```
