# 恋上数据结构与算法-1-15-二叉堆


## 思考

设计一种数据结构，用来存放整数，要求提供 3 个接口
- 添加元素
- 获取最大值
- 删除最大值

![复杂度对比](../../../../images/恋上算法与数据结构/1-15-二叉堆/20200924-复杂度对比.png)  

有没有更优的数据结构？
- 堆
  - 获取最大值：O(1)、删除最大值：O(log n)，添加元素：O(log n)

## Top K 问题

什么是 Top K 问题
- 从海量数据中找出前 K 个数据

比如
- 从 100 万个整数中找出最大的 100 个整数

Top K 问题的解法之一：可以用数据结构“堆”来解决

## 堆（Heap）

堆（Heap）也是一种树状的数据结构（不要跟内存模型中的“堆空间”混淆），常见的堆实现有
- 二叉堆（Binary Heap，完全二叉堆）
- 多叉堆（D-heap、D-ary Heap）
- 索引堆（Index Heap）
- 二项堆（Binomial Heap）
- 斐波那契堆（Fibonacci Heap）
- 左倾堆（Leftist Heap，左式堆）
- 斜堆（Skew Heap）

堆的一个重要性质：任意节点的值总是 >= （<=） 子节点的值
- 如果任意节点的值总是大于等于子节点的值，称为：最大堆、大根堆、大项堆
- 如果任意节点的值总是小于等于子节点的值，称为：最小堆、小根堆、小项堆

![最大堆和最小堆](../../../../images/恋上算法与数据结构/1-15-二叉堆/20200924-最大堆和最小堆.png)  

由此可见，堆中的元素必须具备可比较性（跟二叉搜索树一样）

## 堆的基本接口设计

```java
int size();  //元素的数量
boolean isEmpty();  //是否为空
void clear();  //清空
void add(E element);  //添加元素
E get();  //获得堆顶元素
E remove();  //删除堆顶元素
E replace(E element);  //删除堆顶元素的同时插入一个新元素
```

## 二叉堆（Binary Heap）

二叉堆的逻辑结构就是一棵完全二叉树，所以也叫**完全二叉堆**

鉴于完全二叉树的一些特性，二叉堆的底层（物理结构）一般用数组实现

![二叉堆](../../../../images/恋上算法与数据结构/1-15-二叉堆/20200924-二叉堆.png)  

索引 i 的规律（n 是元素数量）
- 如果 i = 0，它是根节点
- 如果 i > 0，它的父节点的索引为 floor((i-1)/2)
- 如果 2i + 1 <= n - 1，它的左子节点的索引为 2i + 1
- 如果 2i + 1 > n - 1，它无左子节点
- 如果 2i + 2 <= n - 1，它的右子节点的索引为 2i + 2
- 如果 2i + 2 > n - 1，它无右子节点

## 二叉堆的实现

```java
package com.mj.heap;

public interface Heap<E> {
	int size();	// 元素的数量
	boolean isEmpty();	// 是否为空
	void clear();	// 清空
	void add(E element);	 // 添加元素
	E get();	// 获得堆顶元素
	E remove(); // 删除堆顶元素
	E replace(E element); // 删除堆顶元素的同时插入一个新元素
}
```

```java
package com.mj.heap;

import java.util.Comparator;

@SuppressWarnings("unchecked")
public abstract class AbstractHeap<E> implements Heap<E> {
	protected int size;
	protected Comparator<E> comparator;
	
	public AbstractHeap(Comparator<E> comparator) {
		this.comparator = comparator;
	}
	
	public AbstractHeap() {
		this(null);
	}
	
	@Override
	public int size() {
		return size;
	}

	@Override
	public boolean isEmpty() {
		return size == 0;
	}
	
	protected int compare(E e1, E e2) {
		return comparator != null ? comparator.compare(e1, e2) 
				: ((Comparable<E>)e1).compareTo(e2);
	}
}
```

```java
package com.mj.heap;

import java.util.Comparator;

import com.mj.printer.BinaryTreeInfo;

/**
 * 二叉堆（最大堆）
 * @author MJ Lee
 *
 * @param <E>
 */
@SuppressWarnings("unchecked")
public class BinaryHeap<E> extends AbstractHeap<E> implements BinaryTreeInfo {
	private E[] elements;
	private static final int DEFAULT_CAPACITY = 10;
	
	public BinaryHeap(E[] elements, Comparator<E> comparator)  {
		super(comparator);
		
		if (elements == null || elements.length == 0) {
			this.elements = (E[]) new Object[DEFAULT_CAPACITY];
		} else {
			size = elements.length;
			int capacity = Math.max(elements.length, DEFAULT_CAPACITY);
			this.elements = (E[]) new Object[capacity];
			for (int i = 0; i < elements.length; i++) {
				this.elements[i] = elements[i];
			}
			heapify();
		}
	}
	
	public BinaryHeap(E[] elements)  {
		this(elements, null);
	}
	
	public BinaryHeap(Comparator<E> comparator) {
		this(null, comparator);
	}
	
	public BinaryHeap() {
		this(null, null);
	}

	@Override
	public void clear() {
		for (int i = 0; i < size; i++) {
			elements[i] = null;
		}
		size = 0;
	}

	@Override
	public void add(E element) {
		elementNotNullCheck(element);
		ensureCapacity(size + 1);
		elements[size++] = element;
		siftUp(size - 1);
	}

	@Override
	public E get() {
		emptyCheck();
		return elements[0];
	}

	@Override
	public E remove() {
		emptyCheck();
		
		int lastIndex = --size;
		E root = elements[0];
		elements[0] = elements[lastIndex];
		elements[lastIndex] = null;
		
		siftDown(0);
		return root;
	}

	@Override
	public E replace(E element) {
		elementNotNullCheck(element);
		
		E root = null;
		if (size == 0) {
			elements[0] = element;
			size++;
		} else {
			root = elements[0];
			elements[0] = element;
			siftDown(0);
		}
		return root;
	}
	
	/**
	 * 批量建堆
	 */
	private void heapify() {
		// 自上而下的上滤
//		for (int i = 1; i < size; i++) {
//			siftUp(i);
//		}
		
		// 自下而上的下滤
		for (int i = (size >> 1) - 1; i >= 0; i--) {
			siftDown(i);
		}
	}
	
	/**
	 * 让index位置的元素下滤
	 * @param index
	 */
	private void siftDown(int index) {
		E element = elements[index];
		int half = size >> 1;
		// 第一个叶子节点的索引 == 非叶子节点的数量
		// index < 第一个叶子节点的索引
		// 必须保证index位置是非叶子节点
		while (index < half) { 
			// index的节点有2种情况
			// 1.只有左子节点
			// 2.同时有左右子节点
			
			// 默认为左子节点跟它进行比较
			int childIndex = (index << 1) + 1;
			E child = elements[childIndex];
			
			// 右子节点
			int rightIndex = childIndex + 1;
			
			// 选出左右子节点最大的那个
			if (rightIndex < size && compare(elements[rightIndex], child) > 0) {
				child = elements[childIndex = rightIndex];
			}
			
			if (compare(element, child) >= 0) break;

			// 将子节点存放到index位置
			elements[index] = child;
			// 重新设置index
			index = childIndex;
		}
		elements[index] = element;
	}
	
	/**
	 * 让index位置的元素上滤
	 * @param index
	 */
	private void siftUp(int index) {
//		E e = elements[index];
//		while (index > 0) {
//			int pindex = (index - 1) >> 1;
//			E p = elements[pindex];
//			if (compare(e, p) <= 0) return;
//			
//			// 交换index、pindex位置的内容
//			E tmp = elements[index];
//			elements[index] = elements[pindex];
//			elements[pindex] = tmp;
//			
//			// 重新赋值index
//			index = pindex;
//		}
		E element = elements[index];
		while (index > 0) {
			int parentIndex = (index - 1) >> 1;
			E parent = elements[parentIndex];
			if (compare(element, parent) <= 0) break;
			
			// 将父元素存储在index位置
			elements[index] = parent;
			
			// 重新赋值index
			index = parentIndex;
		}
		elements[index] = element;
	}
	
	private void ensureCapacity(int capacity) {
		int oldCapacity = elements.length;
		if (oldCapacity >= capacity) return;
		
		// 新容量为旧容量的1.5倍
		int newCapacity = oldCapacity + (oldCapacity >> 1);
		E[] newElements = (E[]) new Object[newCapacity];
		for (int i = 0; i < size; i++) {
			newElements[i] = elements[i];
		}
		elements = newElements;
	}
	
	private void emptyCheck() {
		if (size == 0) {
			throw new IndexOutOfBoundsException("Heap is empty");
		}
	}
	
	private void elementNotNullCheck(E element) {
		if (element == null) {
			throw new IllegalArgumentException("element must not be null");
		}
	}

	@Override
	public Object root() {
		return 0;
	}

	@Override
	public Object left(Object node) {
		int index = ((int)node << 1) + 1;
		return index >= size ? null : index;
	}

	@Override
	public Object right(Object node) {
		int index = ((int)node << 1) + 2;
		return index >= size ? null : index;
	}

	@Override
	public Object string(Object node) {
		return elements[(int)node];
	}
}
```

### 最大堆 - 添加

![最大堆的添加过程](../../../../images/恋上算法与数据结构/1-15-二叉堆/20200924-最大堆的添加过程.png)  

循环执行以下操作（图中的 80 简称为 node）
- 如果 node > 父节点
  - 与父节点交换位置
- 如果 node < 父节点，或者 node 没有父节点
  - 退出循环

这个过程，叫做**上滤**（Sift Up）
- 时间复杂度：O(log n)

交换位置的优化

![最大堆添加元素交换位置的优化](../../../../images/恋上算法与数据结构/1-15-二叉堆/20200924-最大堆添加元素交换位置的优化.png)  

实现

```java
@Override
public void add(E element) {
	elementNotNullCheck(element);
	ensureCapacity(size + 1);
	elements[size++] = element;
	siftUp(size - 1);
}
```

```java
/**
 * 让index位置的元素上滤
 * @param index
 */
private void siftUp(int index) {
//		E e = elements[index];
//		while (index > 0) {
//			int pindex = (index - 1) >> 1;
//			E p = elements[pindex];
//			if (compare(e, p) <= 0) return;
//			
//			// 交换index、pindex位置的内容
//			E tmp = elements[index];
//			elements[index] = elements[pindex];
//			elements[pindex] = tmp;
//			
//			// 重新赋值index
//			index = pindex;
//		}
	E element = elements[index];
	while (index > 0) {
		int parentIndex = (index - 1) >> 1;
		E parent = elements[parentIndex];
		if (compare(element, parent) <= 0) break;
		
		// 将父元素存储在index位置
		elements[index] = parent;
		
		// 重新赋值index
		index = parentIndex;
	}
	elements[index] = element;
}
```

### 最大堆 - 删除

删除堆顶元素

![最大堆的删除](../../../../images/恋上算法与数据结构/1-15-二叉堆/20200924-最大堆的删除.png)  

1. 用最后一个节点覆盖根节点
2. 删除最后一个节点
3. 循环执行以下操作（图中的43简称为node）
   - 如果 node < 最大的子节点
     - 与最大的子节点交换位置
   - 如果 node >= 最大的子节点，或者 node 没有子节点
     - 退出循环

这个过程，叫做下滤（Sift Down），时间复杂度：O(log n)

同样的，交换位置的操作可以像添加那样进行优化


实现

```java
@Override
public E remove() {
	emptyCheck();
	
	int lastIndex = --size;
	E root = elements[0];
	elements[0] = elements[lastIndex];
	elements[lastIndex] = null;
	
	siftDown(0);
	return root;
}
```

```java
/**
 * 让index位置的元素下滤
 * @param index
 */
private void siftDown(int index) {
	E element = elements[index];
	int half = size >> 1;
	// 第一个叶子节点的索引 == 非叶子节点的数量
	// index < 第一个叶子节点的索引
	// 必须保证index位置是非叶子节点
	while (index < half) { 
		// index的节点有2种情况
		// 1.只有左子节点
		// 2.同时有左右子节点
		
		// 默认为左子节点跟它进行比较
		int childIndex = (index << 1) + 1;
		E child = elements[childIndex];
		
		// 右子节点
		int rightIndex = childIndex + 1;
		
		// 选出左右子节点最大的那个
		if (rightIndex < size && compare(elements[rightIndex], child) > 0) {
			child = elements[childIndex = rightIndex];
		}
		
		if (compare(element, child) >= 0) break;

		// 将子节点存放到index位置
		elements[index] = child;
		// 重新设置index
		index = childIndex;
	}
	elements[index] = element;
}
```

### 最大堆 - 替换

```java
@Override
public E replace(E element) {
	elementNotNullCheck(element);
	
	E root = null;
	if (size == 0) {
		elements[0] = element;
		size++;
	} else {
		root = elements[0];
		elements[0] = element;
		siftDown(0);
	}
	return root;
}
```

### 最大堆 - 批量建堆（Heapify）

![批量建堆](../../../../images/恋上算法与数据结构/1-15-二叉堆/20200924-批量建堆.png)  

批量建堆，有2种做法
- 自上而下的上滤
- 自下而上的下滤


#### 最大堆 - 批量建堆 - 自上而下的上滤

![自下而上的上滤](../../../../images/恋上算法与数据结构/1-15-二叉堆/20200924-自下而上的上滤.png)  

#### 最大堆 - 批量建堆 - 自下而上的下滤

叶子节点下面没有节点了，所以叶子节点没必要下滤

![自下而上的下滤](../../../../images/恋上算法与数据结构/1-15-二叉堆/20200924-自下而上的下滤.png)  

#### 批量建堆效率对比

![批量建堆效率对比](../../../../images/恋上算法与数据结构/1-15-二叉堆/20200924-批量建堆效率对比.png)  

自上而下的上滤相当于一个一个添加，它的效率低于自下而上的下滤

所有节点的深度之和
- 仅仅是叶子节点，就有近 n/2 个，而且每一个叶子节点的深度都是 O(log n) 级别的
- 因此，在叶子节点这一块，就达到了 O(nlog n) 级别
- O(nlog n) 的时间复杂度足以利用排序算法对所有节点进行全排序

所有节点的高度之和
- 假设是满树，节点总个数为 n，树高为 h，那么 n = 2<sup>h</sup> − 1
- 所有节点的树高之和 H(n) = 2<sup>0</sup> ∗ (h − 0) + 2<sup>1</sup> ∗ (h − 1) + 2<sup>2</sup> ∗ (h − 2) + ... + 2<sup>h-1</sup> ∗ [h − (h − 1)]
- H(n) = h ∗ (2<sup>0</sup> + 2<sup>1</sup> + 2<sup>2</sup> + ... + 2<sup>h-1</sup>) - [1 ∗ 2<sup>1</sup> + 2 ∗ 2<sup>2</sup> + 3 ∗ 2<sup>3</sup> + ... + (h − 1) ∗ 2<sup>h-1</sup>]
- H(n) = h ∗ (2<sup>h</sup> − 1) − [(h − 2) ∗ 2<sup>h</sup> + 2]
- H(n) = h ∗ 2<sup>h</sup> − h − h ∗ 2<sup>h</sup> + 2<sup>h+1</sup> − 2
- H(n) = 2<sup>h+1</sup> − h − 2 = 2 ∗ (2<sup>h</sup> − 1) − h = 2n − h = 2n − log<sub>2</sub>(n + 1) = O(n)

公式推导

S(h) = 1 ∗ 2<sup>1</sup> + 2 ∗ 2<sup>2</sup> + 3 ∗ 2<sup>3</sup> + ... + (h − 2) ∗ 2<sup>h-2</sup> + (h − 1) ∗ 2<sup>h-1</sup>

2S(h) = 1 ∗ 2<sup>2</sup> + 2 ∗ 2<sup>3</sup> + 3 ∗ 2<sup>4</sup> + ... + (h − 2) ∗ 2<sup>h-1</sup> + (h − 1) ∗ 2<sup>h</sup>

S(h) – 2S(h) = [2<sup>1</sup> + 2<sup>2</sup> + 2<sup>3</sup> + ... + 2<sup>h-1</sup>] − (h − 1) ∗ 2<sup>h</sup> = (2<sup>h</sup> − 2) − (h − 1) ∗ 2<sup>h</sup>

S(h) = (h − 1) ∗ 2<sup>h</sup> − (2<sup>h</sup> − 2) = (h − 2) ∗ 2<sup>h</sup> + 2

#### 疑惑

![疑惑](../../../../images/恋上算法与数据结构/1-15-二叉堆/20200924-疑惑.png)  

以下方法可以批量建堆吗？
- 自上而下的下滤
- 自下而上的上滤

上述方法不可行

#### 批量建堆的实现

```java
public BinaryHeap(E[] elements, Comparator<E> comparator)  {
	super(comparator);
	
	if (elements == null || elements.length == 0) {
		this.elements = (E[]) new Object[DEFAULT_CAPACITY];
	} else {
		size = elements.length;
		int capacity = Math.max(elements.length, DEFAULT_CAPACITY);
		this.elements = (E[]) new Object[capacity];
		for (int i = 0; i < elements.length; i++) {
			this.elements[i] = elements[i];
		}
		heapify();
	}
}

public BinaryHeap(E[] elements)  {
	this(elements, null);
}
```

```java
/**
 * 批量建堆
 */
private void heapify() {
	// 自上而下的上滤
//		for (int i = 1; i < size; i++) {
//			siftUp(i);
//		}
	
	// 自下而上的下滤
	for (int i = (size >> 1) - 1; i >= 0; i--) {
		siftDown(i);
	}
}
```

#### 如何构建小顶堆

修改比较的逻辑即可

#### Top K 问题

从 n 个整数种，找出最大的前 k 个数（k 远远小于 n）

如果使用排序算法进行全排序，需要 O(nlog n) 的时间复杂度

如果使用二叉堆来解决，可以使用 O(nlog k) 的时间复杂度解决
- 新建一个小顶堆
- 扫描 n 个整数
  - 先将遍历得到的前 k 个数放入堆中
  - 从第 k + 1 个数开始，如果大于堆顶元素，就用 replace 操作（删除堆顶元素，将第k+1个数添加到堆中）
- 扫描完毕后，堆中剩下的就是最大的前 k 个数

![最大前k个数](../../../../images/恋上算法与数据结构/1-15-二叉堆/20200924-最大前k个数.png)  

如果是找出最小的前 k 个数呢？
- 用大顶堆
- 如果小于堆顶元素，就使用 replace 操作



