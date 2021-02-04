# 恋上数据结构与算法-1-02-动态数组


## 数据结构

数据结构是计算机存储、组织数据的方式

![20200909-数据结构概览](/images/恋上算法与数据结构/1-02-动态数组/20200909-数据结构概览.png)  

数据结构可视化网站

https://visualgo.net/zh
## 线性表

**线性表**是具有 n 个**相同类型元素**的有限**序列**（n>=0）

线性表的特点：具有**索引**

![20200909-线性表](/images/恋上算法与数据结构/1-02-动态数组/20200909-线性表.png)  

a<sub>1</sub> 是首节点（首元素），a<sub>n</sub> 是尾节点（尾元素）

a<sub>1</sub> 是 a<sub>2</sub> 的**前驱**，a<sub>2</sub> 是 a<sub>1</sub> 的**后继**。

常见的线性表
- 数组
- 链表
- 栈
- 队列
- 哈希表（散列表）


## 数组（Array）

**数组**是一种**顺序存储**的线性表，所有元素的内存地址是连续的。

```java
int[] array = new int[]{11, 22, 33};
```

![20200909-数组内存结构](/images/恋上算法与数据结构/1-02-动态数组/20200909-数组内存结构.png)  

在很多编程语言中，数组都有个致命的缺点：无法动态修改容量。

在实际开发中，我们更希望数组的容量是可以动态改变的。


### 自定义动态数组

自己设计一个动态数组，模仿 java 官方的动态数组 ArrayList 

#### 动态数组（Dynamic Array）接口设计

我们期望设计的动态数组具有如下接口

```java
int size();  //元素的数量
boolean isEmpty();  //是否为空
boolean contains(E element);  //是否包含某个元素
void add(E element);  //添加元素到最后
E get(int index);  //返回index位置对应的元素
E set(int index, E element);  //设置index位置的元素
void add(int index, E element);  //往index位置添加元素
E remove(int index);  //删除index位置对应的元素
int indexOf(E element);  //查看元素的位置
void clear();  //清除所有元素
```

#### 自定义动态数组 ArrayList 实现

```java
/**
 * 自定义动态数组ArrayList
 */
public class ArrayList<E> {

    /**
     * 元素的数量
     */
    private int size;
    /**
     * 所有的元素
     */
    private E[] elements;

    private static final int DEFAULT_CAPACITY = 10;
    private static final int ELEMENT_NOT_FOUND = -1;

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
    public void clear() {
        for (int i = 0; i < size; i++) {
            elements[i] = null;
        }
        size = 0;
    }

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

    /**
     * 获取index位置的元素
     *
     * @param index
     * @return
     */
    public E get(int index) {
        rangeCheck(index);
        return elements[index];
    }

    /**
     * 设置index位置的元素
     *
     * @param index
     * @param element
     * @return
     */
    public E set(int index, E element) {
        rangeCheck(index);

        E old = elements[index];
        elements[index] = element;
        return old;
    }

    /**
     * 在index位置插入一个元素
     *
     * @param index
     * @param element
     */
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
     *
     * @param index
     * @return
     */
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

    public int indexOf2(E element) {
        for (int i = 0; i < size; i++) {
            if (valEquals(element, elements[i])) return i;
        }
        return ELEMENT_NOT_FOUND;
    }

    private boolean valEquals(Object v1, Object v2) {
        return v1 == null ? v2 == null : v1.equals(v2);
    }

    /**
     * 保证有capacity的容量
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

    private void rangeCheck(int index) {
        if (index < 0 || index >= size) {
            outOfBounds(index);
        }
    }

    private void rangeCheckForAdd(int index) {
        if (index < 0 || index > size) {
            outOfBounds(index);
        }
    }

    private void outOfBounds(int index) {
        throw new IndexOutOfBoundsException("Index: " + index + ", Size: " + size);
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder();
        sb.append("size=").append(size).append(", [");
        for (int i = 0; i < size; i++) {
            if (i != 0) {
                sb.append(", ");
            }
            sb.append(elements[i]);

            //不建议采用此方式拼接，因为此方式多一个减法运算
            /*
            if(i != size -1){
                sb.append(", ");
            }
            */
        }
        sb.append("]");
        return sb.toString();
    }
}
```





