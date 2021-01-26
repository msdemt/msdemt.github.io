# 恋上数据结构与算法-1-16-优先级队列


## 优先级队列（Priority Queue）

![优先级队列](../../../../images/恋上算法与数据结构/1-16-优先级队列/20200924-优先级队列.png)  

优先级队列也是个队列，因此提供如下接口

```java
int size();  //元素的数量
boolean isEmpty();  //是否为空
void enQueue(E element);  //入队
E deQueue();  //出队
E front();  //获取队列的头元素
void clear();  //清空
```

普通的队列是 FIFO 原则，也就是先进先出

优先级队列则是按照优先级高低进行出队，比如将优先级最高的元素作为队头优先出队


## 优先级队列的底层实现

使用二叉堆实现优先级队列

可以通过 Comparator 或 Comparable 去自定义优先级高低

```java
package com.mj.queue;

import java.util.Comparator;

import com.mj.heap.BinaryHeap;

public class PriorityQueue<E> {
	private BinaryHeap<E> heap;
	
	public PriorityQueue(Comparator<E> comparator) {
		heap = new BinaryHeap<>(comparator);
	}
	
	public PriorityQueue() {
		this(null);
	}
	
	public int size() {
		return heap.size();
	}

	public boolean isEmpty() {
		return heap.isEmpty();
	}
	
	public void clear() {
		heap.clear();
	}

	public void enQueue(E element) {
		heap.add(element);
	}

	public E deQueue() {
		return heap.remove();
	}

	public E front() {
		return heap.get();
	}
}
```

测试

```java
public static void main(String[] args) {
	PriorityQueue<Person> queue = new PriorityQueue<>();
	queue.enQueue(new Person("Jack", 2));
	queue.enQueue(new Person("Rose", 10));
	queue.enQueue(new Person("Jake", 5));
	queue.enQueue(new Person("James", 15));
	
	while (!queue.isEmpty()) {
		System.out.println(queue.deQueue());
	}

}
```

```java
public class Person implements Comparable<Person> {
	private String name;
	private int boneBreak;
	public Person(String name, int boneBreak) {
		this.name = name;
		this.boneBreak = boneBreak;
	}
	
	@Override
	public int compareTo(Person person) {
		return this.boneBreak - person.boneBreak;
	}

	@Override
	public String toString() {
		return "Person [name=" + name + ", boneBreak=" + boneBreak + "]";
	}
}
```



