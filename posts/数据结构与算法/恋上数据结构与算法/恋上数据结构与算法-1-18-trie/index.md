# 恋上数据结构与算法-1-18-Trie


## 需求

如何判定一堆不重复的字符串是否以某个前缀开头？
- 用Set\Map存储字符串
- 遍历所有字符串进行判断
- 时间复杂度
  - O(n)

有没有更优的数据结构实现前缀搜索？
- Trie

## Trie

Trie 也叫做字典树、前缀树（Prefix Tree）、单词查找树

Trie 搜索字符串的效率主要跟字符串的长度有关

假设使用 Trie 存储 cat、dog、doggy、does、cast、add 六个单词

![Trie存储数据结构](/images/恋上算法与数据结构/1-18-Trie/20200924-Trie存储数据结构.png)  

## Trie 接口设计参考

只存储字符串

```java
int size();
boolean isEmpty();
void clear();
boolean contains(String str);
void add(String str);
void remove(String str);
boolean startsWith(String prefix)
```

存储字符串和值

```java
int size();
boolean isEmpty();
void clear();
boolean contains(String str);
V add(String str, V value);
V remove(String str);
boolean startsWith(String prefix)
```

## 实现

```java
package com.mj;

import java.util.HashMap;

public class Trie<V> {
	private int size;
	private Node<V> root;
	
	public int size() {
		return size;
	}

	public boolean isEmpty() {
		return size == 0;
	}

	public void clear() {
		size = 0;
		root = null;
	}

	public V get(String key) {
		Node<V> node = node(key);
		return node != null && node.word ? node.value : null;
	}

	public boolean contains(String key) {
		Node<V> node = node(key);
		return node != null && node.word;
	}

	public V add(String key, V value) {
		keyCheck(key);
		
		// 创建根节点
		if (root == null) {
			root = new Node<>(null);
		}

		Node<V> node = root;
		int len = key.length();
		for (int i = 0; i < len; i++) {
			char c = key.charAt(i); 
			boolean emptyChildren = node.children == null;
			Node<V> childNode = emptyChildren ? null : node.children.get(c);
			if (childNode == null) {
				childNode = new Node<>(node);
				childNode.character = c;
				node.children = emptyChildren ? new HashMap<>() : node.children;
				node.children.put(c, childNode);
			}
			node = childNode;
		}
		
		if (node.word) { // 已经存在这个单词
			V oldValue = node.value;
			node.value = value;
			return oldValue;
		}
		
		// 新增一个单词
		node.word = true;
		node.value = value;
		size++;
		return null;
	}

	public V remove(String key) {
		// 找到最后一个节点
		Node<V> node = node(key);
		// 如果不是单词结尾，不用作任何处理
		if (node == null || !node.word) return null;
		size--;
		V oldValue = node.value;
		
		// 如果还有子节点
		if (node.children != null && !node.children.isEmpty()) {
			node.word = false;
			node.value = null;
			return oldValue;
		}
		
		// 如果没有子节点
		Node<V> parent = null;
		while ((parent = node.parent) != null) {
			parent.children.remove(node.character);
			if (parent.word || !parent.children.isEmpty()) break;
			node = parent;
		}
		
		return oldValue;
	}

	public boolean startsWith(String prefix) {
		return node(prefix) != null;
	}
	
	private Node<V> node(String key) {
		keyCheck(key);
		
		Node<V> node = root;
		int len = key.length();
		for (int i = 0; i < len; i++) {
			if (node == null || node.children == null || node.children.isEmpty()) return null;
			char c = key.charAt(i); 
			node = node.children.get(c);
		}
		
		return node;
	}
	
	private void keyCheck(String key) {
		if (key == null || key.length() == 0) {
			throw new IllegalArgumentException("key must not be empty");
		}
	}
	
	private static class Node<V> {
		Node<V> parent;
		HashMap<Character, Node<V>> children;
		Character character;
		V value;
		boolean word; // 是否为单词的结尾（是否为一个完整的单词）
		public Node(Node<V> parent) {
			this.parent = parent;
		}
	}
}
```






## 总结

Trie的优点：搜索前缀的效率主要跟前缀的长度有关

Trie的缺点：需要耗费大量内存，因此还有待改进

更多Trie相关的数据结构和算法
- Double-array Trie、Suffix Tree、Patricia Tree、Crit-bit Tree、AC自动机


## 数据结构总结

复杂度
- 时间复杂度
- 空间复杂度

线性数据结构
- 动态数组（ArrayList）
- 链表（LinkedList）
  - 单向链表
  - 双向链表
  - 循环链表
  - 静态链表
- 栈（Stack）
- 队列（Queue）
  - 双端队列（Deque）
  - 循环队列
- 哈希表（HashTable）

树形数据结构
- 二叉树（BinaryTree）、二叉搜索树（BinarySearchTree、BST）
- 平衡二叉搜索树（BalancedBinarySearchTree、BBST）
  - AVL树（AVLTree）、红黑树（RebBlackTree）
- B树（B-Tree）
- 集合（TreeSet）、映射（TreeMap）
- 哈夫曼树
- Trie

线性+树形数据结构
- 集合（HashSet）
- 映射（HashMap、LinkedHashMap）
- 二叉堆（BinaryHeap）
- 优先级队列（PriorityQueue）
