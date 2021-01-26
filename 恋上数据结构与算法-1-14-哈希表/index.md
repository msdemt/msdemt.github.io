# 恋上数据结构与算法-1-14-哈希表


## 哈希表

### TreeMap 分析

时间复杂度（平均）
- 添加、删除、搜索：O(log n)

特点
- key 必须具备可比较性（TreeMap是由红黑树实现的）
- 元素的分布是有顺序的

在实际应用中，很多时候的需求
- Map 中存储的元素不需要讲究顺序
- Map 中的 key 不需要具备可比较性

不考虑顺序、不考虑 key 的可比较性，Map 有更好的实现方案，平均复杂度可以达到 O(1)
- 那就是采取**哈希表**来实现 Map

### 需求

设计一个写字楼通讯录，存放所有公司的通讯信息
- 座机号码作为 key（假设座机号码长度是8位），公司详情（名称、地址等）作为 value
- 添加、删除、搜索的时间复杂度要求是 O(1)

模拟实现

```java
private Company[] companies = new Company[100000000];
public void add(int phone, Company company){
    companies[phone] = company;
}
public void remove(int phone){
    companies[phone] = null;
}
public Company get(int phone){
    return companies[phone];
}
```

![需求图片](../../../../images/恋上算法与数据结构/1-14-哈希表/20200922-需求图片.png)  

存在什么问题？
- 空间复杂度非常大
- 空间利用率极其低，非常浪费内存空间
- 其实数组 companies 就是一个哈希表，典型的**空间换时间**

### 哈希表（Hash Table）

**哈希表**也叫做**散列表**（hash 有“剁碎”的意思）

它是如何实现高效处理数据的？
- put("Jack", 666);
- put("Rose", 777);
- put("Kate", 888);

![哈希表](../../../../images/恋上算法与数据结构/1-14-哈希表/20200922-哈希表.png)  

添加、搜索、删除的流程都是类似的
1. 利用哈希函数生成 key 对应的 index，时间复杂度 O(1)
2. 根据 index 操作定位数组元素，时间复杂度 O(1)

哈希表是**空间换时间**的典型应用

**哈希函数**，也叫做**散列函数**

哈希表内部的数组元素，很多地方也叫 Bucket（桶），整个数组叫 Buckets 或者 Bucket Array

### 哈希冲突（Hash Collision）

**哈希冲突**也叫做**哈希碰撞**
- 2 个不同的 key，经过哈希函数计算出相同的结果
- key1 ≠ key2， hash(key1) = hash(key2)

![哈希冲突](../../../../images/恋上算法与数据结构/1-14-哈希表/20200922-哈希冲突.png)  

解决哈希冲突的常见方法
1. 开放定址法（Open Addressing）
   - 按照一定规则向其他地址探测，直到遇到空桶
2. 再哈希法（Re-Hashing）
   - 设计多个哈希函数
3. 链地址法（Separate Chaining）
   - 比如通过链表将同一 index 的元素串起来

### JDK1.8 的哈希冲突解决方案

![jdk1.8哈希冲突解决方案](../../../../images/恋上算法与数据结构/1-14-哈希表/20200922-jdk1.8哈希冲突解决方案.png)  

默认使用**单向链表**将元素串起来

在添加元素时，可能会由**单向链表**转为**红黑树**来存储元素
- 比如当哈希表容量 >= 64 且单向链表的节点数量大于 8 时

当**红黑树**节点数量少到一定程度时，又会转为**单向链表**

JDK1.8中的哈希表是使用**链表+红黑树**解决哈希冲突的

思考：这里为什么使用单链表？
- 每次都是从头节点开始遍历
- 单向链表比双向链表少一个指针，可以节省内存空间

### 哈希函数

哈希表中哈希函数的实现步骤大概如下
1. 先生成 key 的哈希值（必须是整数）
2. 再让 key 的哈希值跟数组的大小进行相关运算，生成一个索引值

```java
public int hash(Object key){
    return hash_code(key) % table.length;
}
```

为了提高效率，可以使用 & 位运算取代 % 运算（前提：将数组长度设计为 2 的幂，即 2<sup>n</sup>）

```java
public int hash(Object key){
    return hash_code(key) & (table.length - 1);
}
```

![哈希函数](../../../../images/恋上算法与数据结构/1-14-哈希表/20200922-哈希函数.png)  

良好的哈希函数
- 让哈希值更加均匀分布 -> 减少哈希冲突次数 -> 提升哈希表的性能

### 如何生成 key 的哈希值

key 的常见种类有
- 整数、浮点数、字符串、自定义对象
- 不同种类的 key，哈希值的生成方式不一样，但目标是一致的
  - 尽量让每个 key 的哈希值是唯一的
  - 尽量让 key 的所有信息参与运算

在 Java 中，HashMap 的 key 必须实现 hashCode、equals 方法，也允许 key 为 null

Java 平台的哈希值必须是 int 类型，即 4 个字节，32 位

#### 整数的哈希值

整数值当作哈希值
- 比如 10 的哈希值就是 10

```java
public static int hashCode(int value){
    return value;
}
```

#### 浮点数的哈希值

将存储的二进制格式转为整数值

```java
public static int hashCode(float value){
    return floatToIntBits(value);
}
```

#### Long 的哈希值

Long 类型是 8 字节，64 位，超过了 int 的长度

```java
public static int hashCode(long value){
    return (int)(value ^ (value >>> 32));
}
```

`>>>` : 无符号右移

`>>>` 和 `^` 的作用是？
- 高 32 bit 和 低 32 bit 混合计算出 32 bit 的哈希值
- 充分利用所有信息计算出哈希值

![异或运算](../../../../images/恋上算法与数据结构/1-14-哈希表/20200922-异或运算.png)  

#### Double 的哈希值

Long 类型是 8 字节，64 位，超过了 int 的长度

```java
public static int hashCode(double value){
    long bits = doubleToLongBits(value);
    return (int)(bits ^ (bits >>> 32));
}
```

#### 字符串的哈希值

整数 5489 是如何计算出来的？
- 5 * 10<sup>3</sup> + 4 * 10<sup>2</sup> + 8 * 10<sup>1</sup> + 9 * 10<sup>0</sup>

字符串是由若干字符串组成的
- 比如字符串 jack，由 j、a、c、k 四个字符组成（字符的本质就是一个整数）
- 因此，jack 的哈希值可以表示为 j * n<sup>3</sup> + a * n<sup>2</sup> + c * n<sup>1</sup> + k * n<sup>0</sup>
- 在 JDK 中，乘数 n 为 31，为什么使用 31 ？
  - 31 是一个奇素数，jvm 会将 `31 * i` 优化成 `(i << 5) - i`

```java
String string = "jack";
int hashCode = 0;
int len = string.length();
for(int i=0; i<len; i++){
    char c = string.charAt(i);
    hashCode = 31 * hashCode + c;
}
```

```java
String string = "jack";
int hashCode = 0;
int len = string.length();
for(int i=0; i<len; i++){
    char c = string.charAt(i);
    hashCode = (hashCode << 5) - hashCode + c;
}
```

#### 关于 31 的探讨

31 * i = (2^5 - 1) * i = i * 2^5 - i = (i<<5) - i

31 不仅仅是符合 2^n - 1 ，它是个奇素数（既是奇数，又是素数，也就是质数）
- 素数和其他数相乘的结果比其他方式更容易产生唯一性，减少哈希冲突
- 最终选择31是经过观测分布结果后的选择

#### 自定义对象的哈希值

```java
public class Person {
	private int age;   // 10  20
	private float height; // 1.55 1.67
	private String name; // "jack" "rose"
	
	public Person(int age, float height, String name) {
		this.age = age;
		this.height = height;
		this.name = name;
	}
	
	@Override
	/**
	 * 用来比较2个对象是否相等
	 */
	public boolean equals(Object obj) {
		// 内存地址
		if (this == obj) return true;
		if (obj == null || obj.getClass() != getClass()) return false;
		// if (obj == null || !(obj instanceof Person)) return false;
		
		// 比较成员变量
		Person person = (Person) obj;
		return person.age == age
				&& person.height == height
				&& (person.name == null ? name == null : person.name.equals(name));
	}
	
	@Override
	public int hashCode() {
		int hashCode = Integer.hashCode(age);
		hashCode = hashCode * 31 + Float.hashCode(height);
		hashCode = hashCode * 31 + (name != null ? name.hashCode() : 0);
		return hashCode;
	}
}
```

思考：哈希值太大，整型溢出怎么办？
- 不用作任何处理

#### 自定义对象作为 key

自定义对象作为 key，最好同时重写 hashCode、equals 方法
- equals: 用以判定 2 个 key 是否为同一个 key
  - 自反性：对于任何非 null 的 x， x.equals(x) 必须返回 true
  - 对称性：对于任何非 null 的 x、y，如果 y.equals(x) 返回 true，x.equals(y) 必须返回 true
  - 传递性：对于任何非 null 的 x、y、z，如果 x.equals(y)、y.equals(z) 返回 true，那么 x.equals(z) 必须返回 true
  - 一致性：对于任何非 null 的 x、y，只要 equals 的比较操作在对象中所用的信息没有被修改，多次调用 x.equals(y) 就会一致地返回 true，或者一致地返回 false
  - 对于任何非 null 的 x，x.equals(null) 必须返回 false
- hashCode: 必须保证 equals 为 true 的 2 个 key 的哈希值一样
- 返回来， hashCode 相等的 key，不一定 equals 为 true

不重写 hashCode 方法只重写 equals 方法会有什么结果？
- 可能导致 2 个 equals 为 true 的 key 同时存在哈希表中

hashCode 在计算索引的时候使用，equals 在哈希冲突时比较节点 key 的时候使用

如果两个对象 equals 为 true，那么它们的 hashCode 必须相等（本例中 equals 和 hashCode 都是使用了 age、height、name 计算，所以能保证该性质）

返回来， hashCode 相等的 key，不一定 equals 为 true























#### 哈希值的进一步处理：扰动计算

```java
private int hash(K key){
    if(key == null) return 0;
    int h = key.hashCode();
    return (h ^ (h >>> 16)) & (table.length - 1);
}
```

### 自定义 HashMap 实现

使用数组+红黑树实现自定义 HashMap

```java
package com.mj.map;

import java.util.LinkedList;
import java.util.Objects;
import java.util.Queue;

import com.mj.printer.BinaryTreeInfo;
import com.mj.printer.BinaryTrees;

@SuppressWarnings({"unchecked", "rawtypes"})
public class HashMap_v0<K, V> implements Map<K, V> {
	private static final boolean RED = false;
	private static final boolean BLACK = true;
	private int size;
	private Node<K, V>[] table;
	private static final int DEFAULT_CAPACITY = 1 << 4;
	
	public HashMap_v0() {
		table = new Node[DEFAULT_CAPACITY];
	}

	@Override
	public int size() {
		return size;
	}

	@Override
	public boolean isEmpty() {
		return size == 0;
	}

	@Override
	public void clear() {
		if (size == 0) return;
		size = 0;
		for (int i = 0; i < table.length; i++) {
			table[i] = null;
		}
	}

	@Override
	public V put(K key, V value) {
		int index = index(key);
		// 取出index位置的红黑树根节点
		Node<K, V> root = table[index];
		if (root == null) {
			root = new Node<>(key, value, null);
			table[index] = root;
			size++;
			afterPut(root);
			return null;
		}
		
		// 添加新的节点到红黑树上面
		Node<K, V> parent = root;
		Node<K, V> node = root;
		int cmp = 0;
		K k1 = key;
		int h1 = k1 == null ? 0 : k1.hashCode();
		Node<K, V> result = null;
		boolean searched = false; // 是否已经搜索过这个key
		do {
			parent = node;
			K k2 = node.key;
			int h2 = node.hash;
			if (h1 > h2) {
				cmp = 1;
			} else if (h1 < h2) {
				cmp = -1;
			} else if (Objects.equals(k1, k2)) {
				cmp = 0;
			} else if (k1 != null && k2 != null 
					&& k1.getClass() == k2.getClass()
					&& k1 instanceof Comparable
					&& (cmp = ((Comparable) k1).compareTo(k2)) != 0) {

			} else if (searched) { // 已经扫描了
				cmp = System.identityHashCode(k1) - System.identityHashCode(k2);
			} else { // searched == false; 还没有扫描，然后再根据内存地址大小决定左右
				if ((node.left != null && (result = node(node.left, k1)) != null)
						|| (node.right != null && (result = node(node.right, k1)) != null)) {
					// 已经存在这个key
					node = result;
					cmp = 0;
				} else { // 不存在这个key
					searched = true;
					cmp = System.identityHashCode(k1) - System.identityHashCode(k2);
				}
			}
			
			if (cmp > 0) {
				node = node.right;
			} else if (cmp < 0) {
				node = node.left;
			} else { // 相等
				V oldValue = node.value;
				node.key = key;
				node.value = value;
				node.hash = h1;
				return oldValue;
			}
		} while (node != null);

		// 看看插入到父节点的哪个位置
		Node<K, V> newNode = new Node<>(key, value, parent);
		if (cmp > 0) {
			parent.right = newNode;
		} else {
			parent.left = newNode;
		}
		size++;
		
		// 新添加节点之后的处理
		afterPut(newNode);
		return null;
	}

	@Override
	public V get(K key) {
		Node<K, V> node = node(key);
		return node != null ? node.value : null;
	}

	@Override
	public V remove(K key) {
		return remove(node(key));
	}

	@Override
	public boolean containsKey(K key) {
		return node(key) != null;
	}

	@Override
	public boolean containsValue(V value) {
		if (size == 0) return false;
		Queue<Node<K, V>> queue = new LinkedList<>();
		for (int i = 0; i < table.length; i++) {
			if (table[i] == null) continue;
			
			queue.offer(table[i]);
			while (!queue.isEmpty()) {
				Node<K, V> node = queue.poll();
				if (Objects.equals(value, node.value)) return true;
				
				if (node.left != null) {
					queue.offer(node.left);
				}
				if (node.right != null) {
					queue.offer(node.right);
				}
			}
		}
		return false;
	}

	@Override
	public void traversal(Visitor<K, V> visitor) {
		if (size == 0 || visitor == null) return;
		
		Queue<Node<K, V>> queue = new LinkedList<>();
		for (int i = 0; i < table.length; i++) {
			if (table[i] == null) continue;
			
			queue.offer(table[i]);
			while (!queue.isEmpty()) {
				Node<K, V> node = queue.poll();
				if (visitor.visit(node.key, node.value)) return;
				
				if (node.left != null) {
					queue.offer(node.left);
				}
				if (node.right != null) {
					queue.offer(node.right);
				}
			}
		}
	}
	
	public void print() {
		if (size == 0) return;
		for (int i = 0; i < table.length; i++) {
			final Node<K, V> root = table[i];
			System.out.println("【index = " + i + "】");
			BinaryTrees.println(new BinaryTreeInfo() {
				@Override
				public Object string(Object node) {
					return node;
				}
				
				@Override
				public Object root() {
					return root;
				}
				
				@Override
				public Object right(Object node) {
					return ((Node<K, V>)node).right;
				}
				
				@Override
				public Object left(Object node) {
					return ((Node<K, V>)node).left;
				}
			});
			System.out.println("---------------------------------------------------");
		}
	}
	
	private V remove(Node<K, V> node) {
		if (node == null) return null;
		
		size--;
		
		V oldValue = node.value;
		
		if (node.hasTwoChildren()) { // 度为2的节点
			// 找到后继节点
			Node<K, V> s = successor(node);
			// 用后继节点的值覆盖度为2的节点的值
			node.key = s.key;
			node.value = s.value;
			node.hash = s.hash;
			// 删除后继节点
			node = s;
		}
		
		// 删除node节点（node的度必然是1或者0）
		Node<K, V> replacement = node.left != null ? node.left : node.right;
		int index = index(node);
		
		if (replacement != null) { // node是度为1的节点
			// 更改parent
			replacement.parent = node.parent;
			// 更改parent的left、right的指向
			if (node.parent == null) { // node是度为1的节点并且是根节点
				table[index] = replacement;
			} else if (node == node.parent.left) {
				node.parent.left = replacement;
			} else { // node == node.parent.right
				node.parent.right = replacement;
			}
			
			// 删除节点之后的处理
			afterRemove(replacement);
		} else if (node.parent == null) { // node是叶子节点并且是根节点
			table[index] = null;
		} else { // node是叶子节点，但不是根节点
			if (node == node.parent.left) {
				node.parent.left = null;
			} else { // node == node.parent.right
				node.parent.right = null;
			}
			
			// 删除节点之后的处理
			afterRemove(node);
		}
		
		return oldValue;
	}
	
	private Node<K, V> successor(Node<K, V> node) {
		if (node == null) return null;
		
		// 前驱节点在左子树当中（right.left.left.left....）
		Node<K, V> p = node.right;
		if (p != null) {
			while (p.left != null) {
				p = p.left;
			}
			return p;
		}
		
		// 从父节点、祖父节点中寻找前驱节点
		while (node.parent != null && node == node.parent.right) {
			node = node.parent;
		}

		return node.parent;
	}
	
	private Node<K, V> node(K key) {
		Node<K, V> root = table[index(key)];
		return root == null ? null : node(root, key);
	}
	
	private Node<K, V> node(Node<K, V> node, K k1) {
		int h1 = k1 == null ? 0 : k1.hashCode();
		// 存储查找结果
		Node<K, V> result = null;
		int cmp = 0;
		while (node != null) {
			K k2 = node.key;
			int h2 = node.hash;
			// 先比较哈希值
			if (h1 > h2) {
				node = node.right;
			} else if (h1 < h2) {
				node = node.left;
			} else if (Objects.equals(k1, k2)) {
				return node;
			} else if (k1 != null && k2 != null 
				&& k1.getClass() == k2.getClass()
				&& k1 instanceof Comparable
				&& (cmp = ((Comparable) k1).compareTo(k2)) != 0) {
				node = cmp > 0 ? node.right : node.left;
			} else if (node.right != null && (result = node(node.right, k1)) != null) { 
				return result;
			} else { // 只能往左边找
				node = node.left;
			}
//			} else if (node.left != null && (result = node(node.left, k1)) != null) { 
//				return result;
//			} else {
//				return null;
//			}
		}
		return null;
	}
	
	/**
	 * 根据key生成对应的索引（在桶数组中的位置）
	 */
	private int index(K key) {
		if (key == null) return 0;
		int hash = key.hashCode();
		return (hash ^ (hash >>> 16)) & (table.length - 1);
	}
	
	private int index(Node<K, V> node) {
		return (node.hash ^ (node.hash >>> 16)) & (table.length - 1);
	}
	
	/**
	 * 比较key大小
	 * @param k1
	 * @param k2
	 * @param h1 k1的hashCode
	 * @param h2 k2的hashCode
	 * @return
	 */
//	private int compare(K k1, K k2, int h1, int h2) {
//		// 比较哈希值
//		int result = h1 - h2;
//		if (result != 0) return result;
//		
//		// 比较equals
//		if (Objects.equals(k1, k2)) return 0;
//		
//		// 哈希值相等，但是不equals
//		if (k1 != null && k2 != null 
//				&& k1.getClass() == k2.getClass()
//				&& k1 instanceof Comparable) {
//			// 同一种类型并且具备可比较性
//			if (k1 instanceof Comparable) {
//				return ((Comparable) k1).compareTo(k2);
//			}
//		}
//		
//		// 同一种类型，哈希值相等，但是不equals，但是不具备可比较性
//		// k1不为null，k2为null
//		// k1为null，k2不为null
//		return System.identityHashCode(k1) - System.identityHashCode(k2);
//	}

	private void afterRemove(Node<K, V> node) {
		// 如果删除的节点是红色
		// 或者 用以取代删除节点的子节点是红色
		if (isRed(node)) {
			black(node);
			return;
		}
		
		Node<K, V> parent = node.parent;
		if (parent == null) return;
		
		// 删除的是黑色叶子节点【下溢】
		// 判断被删除的node是左还是右
		boolean left = parent.left == null || node.isLeftChild();
		Node<K, V> sibling = left ? parent.right : parent.left;
		if (left) { // 被删除的节点在左边，兄弟节点在右边
			if (isRed(sibling)) { // 兄弟节点是红色
				black(sibling);
				red(parent);
				rotateLeft(parent);
				// 更换兄弟
				sibling = parent.right;
			}
			
			// 兄弟节点必然是黑色
			if (isBlack(sibling.left) && isBlack(sibling.right)) {
				// 兄弟节点没有1个红色子节点，父节点要向下跟兄弟节点合并
				boolean parentBlack = isBlack(parent);
				black(parent);
				red(sibling);
				if (parentBlack) {
					afterRemove(parent);
				}
			} else { // 兄弟节点至少有1个红色子节点，向兄弟节点借元素
				// 兄弟节点的左边是黑色，兄弟要先旋转
				if (isBlack(sibling.right)) {
					rotateRight(sibling);
					sibling = parent.right;
				}
				
				color(sibling, colorOf(parent));
				black(sibling.right);
				black(parent);
				rotateLeft(parent);
			}
		} else { // 被删除的节点在右边，兄弟节点在左边
			if (isRed(sibling)) { // 兄弟节点是红色
				black(sibling);
				red(parent);
				rotateRight(parent);
				// 更换兄弟
				sibling = parent.left;
			}
			
			// 兄弟节点必然是黑色
			if (isBlack(sibling.left) && isBlack(sibling.right)) {
				// 兄弟节点没有1个红色子节点，父节点要向下跟兄弟节点合并
				boolean parentBlack = isBlack(parent);
				black(parent);
				red(sibling);
				if (parentBlack) {
					afterRemove(parent);
				}
			} else { // 兄弟节点至少有1个红色子节点，向兄弟节点借元素
				// 兄弟节点的左边是黑色，兄弟要先旋转
				if (isBlack(sibling.left)) {
					rotateLeft(sibling);
					sibling = parent.left;
				}
				
				color(sibling, colorOf(parent));
				black(sibling.left);
				black(parent);
				rotateRight(parent);
			}
		}
	}

	private void afterPut(Node<K, V> node) {
		Node<K, V> parent = node.parent;
		
		// 添加的是根节点 或者 上溢到达了根节点
		if (parent == null) {
			black(node);
			return;
		}
		
		// 如果父节点是黑色，直接返回
		if (isBlack(parent)) return;
		
		// 叔父节点
		Node<K, V> uncle = parent.sibling();
		// 祖父节点
		Node<K, V> grand = red(parent.parent);
		if (isRed(uncle)) { // 叔父节点是红色【B树节点上溢】
			black(parent);
			black(uncle);
			// 把祖父节点当做是新添加的节点
			afterPut(grand);
			return;
		}
		
		// 叔父节点不是红色
		if (parent.isLeftChild()) { // L
			if (node.isLeftChild()) { // LL
				black(parent);
			} else { // LR
				black(node);
				rotateLeft(parent);
			}
			rotateRight(grand);
		} else { // R
			if (node.isLeftChild()) { // RL
				black(node);
				rotateRight(parent);
			} else { // RR
				black(parent);
			}
			rotateLeft(grand);
		}
	}
	
	private void rotateLeft(Node<K, V> grand) {
		Node<K, V> parent = grand.right;
		Node<K, V> child = parent.left;
		grand.right = child;
		parent.left = grand;
		afterRotate(grand, parent, child);
	}
	
	private void rotateRight(Node<K, V> grand) {
		Node<K, V> parent = grand.left;
		Node<K, V> child = parent.right;
		grand.left = child;
		parent.right = grand;
		afterRotate(grand, parent, child);
	}
	
	private void afterRotate(Node<K, V> grand, Node<K, V> parent, Node<K, V> child) {
		// 让parent称为子树的根节点
		parent.parent = grand.parent;
		if (grand.isLeftChild()) {
			grand.parent.left = parent;
		} else if (grand.isRightChild()) {
			grand.parent.right = parent;
		} else { // grand是root节点
			table[index(grand)] = parent;
		}
		
		// 更新child的parent
		if (child != null) {
			child.parent = grand;
		}
		
		// 更新grand的parent
		grand.parent = parent;
	}

	private Node<K, V> color(Node<K, V> node, boolean color) {
		if (node == null) return node;
		node.color = color;
		return node;
	}
	
	private Node<K, V> red(Node<K, V> node) {
		return color(node, RED);
	}
	
	private Node<K, V> black(Node<K, V> node) {
		return color(node, BLACK);
	}
	
	private boolean colorOf(Node<K, V> node) {
		return node == null ? BLACK : node.color;
	}
	
	private boolean isBlack(Node<K, V> node) {
		return colorOf(node) == BLACK;
	}
	
	private boolean isRed(Node<K, V> node) {
		return colorOf(node) == RED;
	}
	
	private static class Node<K, V> {
		int hash;
		K key;
		V value;
		boolean color = RED;
		Node<K, V> left;
		Node<K, V> right;
		Node<K, V> parent;
		public Node(K key, V value, Node<K, V> parent) {
			this.key = key;
			this.hash = key == null ? 0 : key.hashCode();
			this.value = value;
			this.parent = parent;
		}
		
		public boolean hasTwoChildren() {
			return left != null && right != null;
		}
		
		public boolean isLeftChild() {
			return parent != null && this == parent.left;
		}
		
		public boolean isRightChild() {
			return parent != null && this == parent.right;
		}
		
		public Node<K, V> sibling() {
			if (isLeftChild()) {
				return parent.right;
			}
			
			if (isRightChild()) {
				return parent.left;
			}
			
			return null;
		}
		
		@Override
		public String toString() {
			return "Node_" + key + "_" + value;
		}
	}
}
```

## 扩容

### 装填因子

装填因子（Load Factor）：节点总数量/哈希表桶数组长度，也叫做负载因子

在 jdk1.8 的 HashMap 中，如果装填因子超过0.75，就扩容为原来的2倍

### 具有扩容功能的哈希表

```java
package com.mj.map;

import java.util.LinkedList;
import java.util.Objects;
import java.util.Queue;

import com.mj.printer.BinaryTreeInfo;
import com.mj.printer.BinaryTrees;

@SuppressWarnings({"unchecked", "rawtypes"})
public class HashMap<K, V> implements Map<K, V> {
	private static final boolean RED = false;
	private static final boolean BLACK = true;
	private int size;
	private Node<K, V>[] table;
	private static final int DEFAULT_CAPACITY = 1 << 4;
	private static final float DEFAULT_LOAD_FACTOR = 0.75f;
	
	public HashMap() {
		table = new Node[DEFAULT_CAPACITY];
	}

	@Override
	public int size() {
		return size;
	}

	@Override
	public boolean isEmpty() {
		return size == 0;
	}

	@Override
	public void clear() {
		if (size == 0) return;
		size = 0;
		for (int i = 0; i < table.length; i++) {
			table[i] = null;
		}
	}

	@Override
	public V put(K key, V value) {
		resize();
		
		int index = index(key);
		// 取出index位置的红黑树根节点
		Node<K, V> root = table[index];
		if (root == null) {
			root = createNode(key, value, null);
			table[index] = root;
			size++;
			fixAfterPut(root);
			return null;
		}
		
		// 添加新的节点到红黑树上面
		Node<K, V> parent = root;
		Node<K, V> node = root;
		int cmp = 0;
		K k1 = key;
		int h1 = hash(k1);
		Node<K, V> result = null;
		boolean searched = false; // 是否已经搜索过这个key
		do {
			parent = node;
			K k2 = node.key;
			int h2 = node.hash;
			if (h1 > h2) {
				cmp = 1;
			} else if (h1 < h2) {
				cmp = -1;
			} else if (Objects.equals(k1, k2)) {
				cmp = 0;
			} else if (k1 != null && k2 != null 
					&& k1 instanceof Comparable
					&& k1.getClass() == k2.getClass()
					&& (cmp = ((Comparable)k1).compareTo(k2)) != 0) {
			} else if (searched) { // 已经扫描了
				cmp = System.identityHashCode(k1) - System.identityHashCode(k2);
			} else { // searched == false; 还没有扫描，然后再根据内存地址大小决定左右
				if ((node.left != null && (result = node(node.left, k1)) != null)
						|| (node.right != null && (result = node(node.right, k1)) != null)) {
					// 已经存在这个key
					node = result;
					cmp = 0;
				} else { // 不存在这个key
					searched = true;
					cmp = System.identityHashCode(k1) - System.identityHashCode(k2);
				}
			}
			
			if (cmp > 0) {
				node = node.right;
			} else if (cmp < 0) {
				node = node.left;
			} else { // 相等
				V oldValue = node.value;
				node.key = key;
				node.value = value;
				node.hash = h1;
				return oldValue;
			}
		} while (node != null);

		// 看看插入到父节点的哪个位置
		Node<K, V> newNode = createNode(key, value, parent);
		if (cmp > 0) {
			parent.right = newNode;
		} else {
			parent.left = newNode;
		}
		size++;
		
		// 新添加节点之后的处理
		fixAfterPut(newNode);
		return null;
	}

	@Override
	public V get(K key) {
		Node<K, V> node = node(key);
		return node != null ? node.value : null;
	}

	@Override
	public V remove(K key) {
		return remove(node(key));
	}

	@Override
	public boolean containsKey(K key) {
		return node(key) != null;
	}

	@Override
	public boolean containsValue(V value) {
		if (size == 0) return false;
		Queue<Node<K, V>> queue = new LinkedList<>();
		for (int i = 0; i < table.length; i++) {
			if (table[i] == null) continue;
			
			queue.offer(table[i]);
			while (!queue.isEmpty()) {
				Node<K, V> node = queue.poll();
				if (Objects.equals(value, node.value)) return true;
				
				if (node.left != null) {
					queue.offer(node.left);
				}
				if (node.right != null) {
					queue.offer(node.right);
				}
			}
		}
		return false;
	}

	@Override
	public void traversal(Visitor<K, V> visitor) {
		if (size == 0 || visitor == null) return;
		
		Queue<Node<K, V>> queue = new LinkedList<>();
		for (int i = 0; i < table.length; i++) {
			if (table[i] == null) continue;
			
			queue.offer(table[i]);
			while (!queue.isEmpty()) {
				Node<K, V> node = queue.poll();
				if (visitor.visit(node.key, node.value)) return;
				
				if (node.left != null) {
					queue.offer(node.left);
				}
				if (node.right != null) {
					queue.offer(node.right);
				}
			}
		}
	}
	
	public void print() {
		if (size == 0) return;
		for (int i = 0; i < table.length; i++) {
			final Node<K, V> root = table[i];
			System.out.println("【index = " + i + "】");
			BinaryTrees.println(new BinaryTreeInfo() {
				@Override
				public Object string(Object node) {
					return node;
				}
				
				@Override
				public Object root() {
					return root;
				}
				
				@Override
				public Object right(Object node) {
					return ((Node<K, V>)node).right;
				}
				
				@Override
				public Object left(Object node) {
					return ((Node<K, V>)node).left;
				}
			});
			System.out.println("---------------------------------------------------");
		}
	}
	
	protected Node<K, V> createNode(K key, V value, Node<K, V> parent) {
		return new Node<>(key, value, parent);
	}
	
	protected void afterRemove(Node<K, V> willNode, Node<K, V> removedNode) { }
	
	private void resize() {
		// 装填因子 <= 0.75
		if (size / table.length <= DEFAULT_LOAD_FACTOR) return;
		
		Node<K, V>[] oldTable = table;
		table = new Node[oldTable.length << 1];

		Queue<Node<K, V>> queue = new LinkedList<>();
		for (int i = 0; i < oldTable.length; i++) {
			if (oldTable[i] == null) continue;
			
			queue.offer(oldTable[i]);
			while (!queue.isEmpty()) {
				Node<K, V> node = queue.poll();
				if (node.left != null) {
					queue.offer(node.left);
				}
				if (node.right != null) {
					queue.offer(node.right);
				}
				
				// 挪动代码得放到最后面
				moveNode(node);
			}
		}
	}
	
	private void moveNode(Node<K, V> newNode) {
		// 重置
		newNode.parent = null;
		newNode.left = null;
		newNode.right = null;
		newNode.color = RED;
		
		int index = index(newNode);
		// 取出index位置的红黑树根节点
		Node<K, V> root = table[index];
		if (root == null) {
			root = newNode;
			table[index] = root;
			fixAfterPut(root);
			return;
		}
		
		// 添加新的节点到红黑树上面
		Node<K, V> parent = root;
		Node<K, V> node = root;
		int cmp = 0;
		K k1 = newNode.key;
		int h1 = newNode.hash;
		do {
			parent = node;
			K k2 = node.key;
			int h2 = node.hash;
			if (h1 > h2) {
				cmp = 1;
			} else if (h1 < h2) {
				cmp = -1;
			} else if (k1 != null && k2 != null 
					&& k1 instanceof Comparable
					&& k1.getClass() == k2.getClass()
					&& (cmp = ((Comparable)k1).compareTo(k2)) != 0) {
			} else {
				cmp = System.identityHashCode(k1) - System.identityHashCode(k2);
			}
			
			if (cmp > 0) {
				node = node.right;
			} else if (cmp < 0) {
				node = node.left;
			}
		} while (node != null);

		// 看看插入到父节点的哪个位置
		newNode.parent = parent;
		if (cmp > 0) {
			parent.right = newNode;
		} else {
			parent.left = newNode;
		}
		
		// 新添加节点之后的处理
		fixAfterPut(newNode);
	}
	
	protected V remove(Node<K, V> node) {
		if (node == null) return null;
		
		Node<K, V> willNode = node;
		
		size--;
		
		V oldValue = node.value;
		
		if (node.hasTwoChildren()) { // 度为2的节点
			// 找到后继节点
			Node<K, V> s = successor(node);
			// 用后继节点的值覆盖度为2的节点的值
			node.key = s.key;
			node.value = s.value;
			node.hash = s.hash;
			// 删除后继节点
			node = s;
		}
		
		// 删除node节点（node的度必然是1或者0）
		Node<K, V> replacement = node.left != null ? node.left : node.right;
		int index = index(node);
		
		if (replacement != null) { // node是度为1的节点
			// 更改parent
			replacement.parent = node.parent;
			// 更改parent的left、right的指向
			if (node.parent == null) { // node是度为1的节点并且是根节点
				table[index] = replacement;
			} else if (node == node.parent.left) {
				node.parent.left = replacement;
			} else { // node == node.parent.right
				node.parent.right = replacement;
			}
			
			// 删除节点之后的处理
			fixAfterRemove(replacement);
		} else if (node.parent == null) { // node是叶子节点并且是根节点
			table[index] = null;
		} else { // node是叶子节点，但不是根节点
			if (node == node.parent.left) {
				node.parent.left = null;
			} else { // node == node.parent.right
				node.parent.right = null;
			}
			
			// 删除节点之后的处理
			fixAfterRemove(node);
		}
		
		// 交给子类去处理
		afterRemove(willNode, node);
		
		return oldValue;
	}
	
	private Node<K, V> successor(Node<K, V> node) {
		if (node == null) return null;
		
		// 前驱节点在左子树当中（right.left.left.left....）
		Node<K, V> p = node.right;
		if (p != null) {
			while (p.left != null) {
				p = p.left;
			}
			return p;
		}
		
		// 从父节点、祖父节点中寻找前驱节点
		while (node.parent != null && node == node.parent.right) {
			node = node.parent;
		}

		return node.parent;
	}
	
	private Node<K, V> node(K key) {
		Node<K, V> root = table[index(key)];
		return root == null ? null : node(root, key);
	}
	
	private Node<K, V> node(Node<K, V> node, K k1) {
		int h1 = hash(k1);
		// 存储查找结果
		Node<K, V> result = null;
		int cmp = 0;
		while (node != null) {
			K k2 = node.key;
			int h2 = node.hash;
			// 先比较哈希值
			if (h1 > h2) {
				node = node.right;
			} else if (h1 < h2) {
				node = node.left;
			} else if (Objects.equals(k1, k2)) {
				return node;
			} else if (k1 != null && k2 != null 
					&& k1 instanceof Comparable
					&& k1.getClass() == k2.getClass()
					&& (cmp = ((Comparable)k1).compareTo(k2)) != 0) {
				node = cmp > 0 ? node.right : node.left;
			} else if (node.right != null && (result = node(node.right, k1)) != null) { 
				return result;
			} else { // 只能往左边找
				node = node.left;
			}
		}
		return null;
	}
	
	/**
	 * 根据key生成对应的索引（在桶数组中的位置）
	 */
	private int index(K key) {
		return hash(key) & (table.length - 1);
	}
	
	private int hash(K key) {
		if (key == null) return 0;
		int hash = key.hashCode();
		return hash ^ (hash >>> 16);
	}
	
	private int index(Node<K, V> node) {
		return node.hash & (table.length - 1);
	}

	private void fixAfterRemove(Node<K, V> node) {
		// 如果删除的节点是红色
		// 或者 用以取代删除节点的子节点是红色
		if (isRed(node)) {
			black(node);
			return;
		}
		
		Node<K, V> parent = node.parent;
		if (parent == null) return;
		
		// 删除的是黑色叶子节点【下溢】
		// 判断被删除的node是左还是右
		boolean left = parent.left == null || node.isLeftChild();
		Node<K, V> sibling = left ? parent.right : parent.left;
		if (left) { // 被删除的节点在左边，兄弟节点在右边
			if (isRed(sibling)) { // 兄弟节点是红色
				black(sibling);
				red(parent);
				rotateLeft(parent);
				// 更换兄弟
				sibling = parent.right;
			}
			
			// 兄弟节点必然是黑色
			if (isBlack(sibling.left) && isBlack(sibling.right)) {
				// 兄弟节点没有1个红色子节点，父节点要向下跟兄弟节点合并
				boolean parentBlack = isBlack(parent);
				black(parent);
				red(sibling);
				if (parentBlack) {
					fixAfterRemove(parent);
				}
			} else { // 兄弟节点至少有1个红色子节点，向兄弟节点借元素
				// 兄弟节点的左边是黑色，兄弟要先旋转
				if (isBlack(sibling.right)) {
					rotateRight(sibling);
					sibling = parent.right;
				}
				
				color(sibling, colorOf(parent));
				black(sibling.right);
				black(parent);
				rotateLeft(parent);
			}
		} else { // 被删除的节点在右边，兄弟节点在左边
			if (isRed(sibling)) { // 兄弟节点是红色
				black(sibling);
				red(parent);
				rotateRight(parent);
				// 更换兄弟
				sibling = parent.left;
			}
			
			// 兄弟节点必然是黑色
			if (isBlack(sibling.left) && isBlack(sibling.right)) {
				// 兄弟节点没有1个红色子节点，父节点要向下跟兄弟节点合并
				boolean parentBlack = isBlack(parent);
				black(parent);
				red(sibling);
				if (parentBlack) {
					fixAfterRemove(parent);
				}
			} else { // 兄弟节点至少有1个红色子节点，向兄弟节点借元素
				// 兄弟节点的左边是黑色，兄弟要先旋转
				if (isBlack(sibling.left)) {
					rotateLeft(sibling);
					sibling = parent.left;
				}
				
				color(sibling, colorOf(parent));
				black(sibling.left);
				black(parent);
				rotateRight(parent);
			}
		}
	}

	private void fixAfterPut(Node<K, V> node) {
		Node<K, V> parent = node.parent;
		
		// 添加的是根节点 或者 上溢到达了根节点
		if (parent == null) {
			black(node);
			return;
		}
		
		// 如果父节点是黑色，直接返回
		if (isBlack(parent)) return;
		
		// 叔父节点
		Node<K, V> uncle = parent.sibling();
		// 祖父节点
		Node<K, V> grand = red(parent.parent);
		if (isRed(uncle)) { // 叔父节点是红色【B树节点上溢】
			black(parent);
			black(uncle);
			// 把祖父节点当做是新添加的节点
			fixAfterPut(grand);
			return;
		}
		
		// 叔父节点不是红色
		if (parent.isLeftChild()) { // L
			if (node.isLeftChild()) { // LL
				black(parent);
			} else { // LR
				black(node);
				rotateLeft(parent);
			}
			rotateRight(grand);
		} else { // R
			if (node.isLeftChild()) { // RL
				black(node);
				rotateRight(parent);
			} else { // RR
				black(parent);
			}
			rotateLeft(grand);
		}
	}
	
	private void rotateLeft(Node<K, V> grand) {
		Node<K, V> parent = grand.right;
		Node<K, V> child = parent.left;
		grand.right = child;
		parent.left = grand;
		afterRotate(grand, parent, child);
	}
	
	private void rotateRight(Node<K, V> grand) {
		Node<K, V> parent = grand.left;
		Node<K, V> child = parent.right;
		grand.left = child;
		parent.right = grand;
		afterRotate(grand, parent, child);
	}
	
	private void afterRotate(Node<K, V> grand, Node<K, V> parent, Node<K, V> child) {
		// 让parent称为子树的根节点
		parent.parent = grand.parent;
		if (grand.isLeftChild()) {
			grand.parent.left = parent;
		} else if (grand.isRightChild()) {
			grand.parent.right = parent;
		} else { // grand是root节点
			table[index(grand)] = parent;
		}
		
		// 更新child的parent
		if (child != null) {
			child.parent = grand;
		}
		
		// 更新grand的parent
		grand.parent = parent;
	}

	private Node<K, V> color(Node<K, V> node, boolean color) {
		if (node == null) return node;
		node.color = color;
		return node;
	}
	
	private Node<K, V> red(Node<K, V> node) {
		return color(node, RED);
	}
	
	private Node<K, V> black(Node<K, V> node) {
		return color(node, BLACK);
	}
	
	private boolean colorOf(Node<K, V> node) {
		return node == null ? BLACK : node.color;
	}
	
	private boolean isBlack(Node<K, V> node) {
		return colorOf(node) == BLACK;
	}
	
	private boolean isRed(Node<K, V> node) {
		return colorOf(node) == RED;
	}
	
	protected static class Node<K, V> {
		int hash;
		K key;
		V value;
		boolean color = RED;
		Node<K, V> left;
		Node<K, V> right;
		Node<K, V> parent;
		public Node(K key, V value, Node<K, V> parent) {
			this.key = key;
			int hash = key == null ? 0 : key.hashCode();
			this.hash = hash ^ (hash >>> 16);
			this.value = value;
			this.parent = parent;
		}
		
		public boolean hasTwoChildren() {
			return left != null && right != null;
		}
		
		public boolean isLeftChild() {
			return parent != null && this == parent.left;
		}
		
		public boolean isRightChild() {
			return parent != null && this == parent.right;
		}
		
		public Node<K, V> sibling() {
			if (isLeftChild()) {
				return parent.right;
			}
			
			if (isRightChild()) {
				return parent.left;
			}
			
			return null;
		}
		
		@Override
		public String toString() {
			return "Node_" + key + "_" + value;
		}
	}
}
```

## TreeMap vs HashMap

何时选择 TreeMap
- 元素具备可比较性且要求升序遍历（元素从小到大）

何时选择 HashMap
- 无序遍历

## 关于使用%计算索引

如何使用%来计算索引
- 建议把哈希表的长度设计为素数（质数）
- 可以大大减少哈希冲突

![素数减少哈希冲突](../../../../images/恋上算法与数据结构/1-14-哈希表/20200924-素数减少哈希冲突.png)  

下表列出了不同数据规模对应的最佳素数，特点如下
- 每个素数略小于前一个素数的2倍
- 每个素数尽可能接近2的幂（2<sup>2</sup>）

![上下界的素数选择](../../../../images/恋上算法与数据结构/1-14-哈希表/20200924-上下界的素数选择.png)  

## LinkedHashMap

在HashMap的基础上维护元素的添加顺序，使得遍历的结果是遵从添加顺序的

假设添加顺序是
- 37、21、31、41、97、95、52、42、83

![LinkedHashMap](../../../../images/恋上算法与数据结构/1-14-哈希表/20200924-LinkedHashMap.png)  

删除度为2的节点node时
- 需要注意更换node与前驱\后继节点的连接位置

![删除注意点](../../../../images/恋上算法与数据结构/1-14-哈希表/20200924-删除注意点.png)  

### 实现

```java
package com.mj.map;

import java.util.Objects;

@SuppressWarnings({ "rawtypes", "unchecked" })
public class LinkedHashMap<K, V> extends HashMap<K, V> {
	private LinkedNode<K, V> first;
	private LinkedNode<K, V> last;
	
	@Override
	public void clear() {
		super.clear();
		first = null;
		last = null;
	}
	
	@Override
	public boolean containsValue(V value) {
		LinkedNode<K, V> node = first;
		while (node != null) {
			if (Objects.equals(value, node.value)) return true;
			node = node.next;
		}
		return false;
	}
	
	@Override
	public void traversal(Visitor<K, V> visitor) {
		if (visitor == null) return;
		LinkedNode<K, V> node = first;
		while (node != null) {
			if (visitor.visit(node.key, node.value)) return;
			node = node.next;
		}
	}
	
	@Override
	protected void afterRemove(Node<K, V> willNode, Node<K, V> removedNode) {
		LinkedNode<K, V> node1 = (LinkedNode<K, V>) willNode;
		LinkedNode<K, V> node2 = (LinkedNode<K, V>) removedNode;
		
		if (node1 != node2) {
			// 交换linkedWillNode和linkedRemovedNode在链表中的位置
			// 交换prev
			LinkedNode<K, V> tmp = node1.prev;
			node1.prev = node2.prev;
			node2.prev = tmp;
			if (node1.prev == null) {
				first = node1;
			} else {
				node1.prev.next = node1;
			}
			if (node2.prev == null) {
				first = node2;
			} else {
				node2.prev.next = node2;
			}
			
			// 交换next
			tmp = node1.next;
			node1.next = node2.next;
			node2.next = tmp;
			if (node1.next == null) {
				last = node1;
			} else {
				node1.next.prev = node1;
			}
			if (node2.next == null) {
				last = node2;
			} else {
				node2.next.prev = node2;
			}
		}
		
		LinkedNode<K, V> prev = node2.prev;
		LinkedNode<K, V> next = node2.next;
		if (prev == null) {
			first = next;
		} else {
			prev.next = next;
		}
		
		if (next == null) {
			last = prev;
		} else {
			next.prev = prev;
		}
	}
	
	@Override
	protected Node<K, V> createNode(K key, V value, Node<K, V> parent) {
		LinkedNode node = new LinkedNode(key, value, parent);
		if (first == null) {
			first = last = node;
		} else {
			last.next = node;
			node.prev = last;
			last = node;
		}
		return node;
	}
	
	private static class LinkedNode<K, V> extends Node<K, V> {
		LinkedNode<K, V> prev;
		LinkedNode<K, V> next;
		public LinkedNode(K key, V value, Node<K, V> parent) {
			super(key, value, parent);
		}
	}
}
```



