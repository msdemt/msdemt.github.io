# 恋上数据结构与算法-1-12-集合


## 集合（set）

集合的特点
- 不存放重复的元素
- 常用于去重
  - 存放新增IP，统计新增IP量
  - 存放词汇，统计词汇量
  - ......

集合的接口

```java
public interface Set<E> {
	int size();
	boolean isEmpty();
	void clear();
	boolean contains(E element);
	void add(E element);
	void remove(E element);
	void traversal(Visitor<E> visitor);
	
	public static abstract class Visitor<E> {
		boolean stop;
		public abstract boolean visit(E element);
	}
}
```

思考：集合的内部实现能否直接利用以前学过的数据结构？
- 动态数组
- 链表
- 二叉搜索树（AVL树、红黑树）: 元素必须具备可比较性（如果元素不具备可比较性，又想要红黑树的性能，可以使用哈希表）

## 集合的实现

集合的接口

```java
package com.mj.set;

public interface Set<E> {
	int size();
	boolean isEmpty();
	void clear();
	boolean contains(E element);
	void add(E element);
	void remove(E element);
	void traversal(Visitor<E> visitor);
	
	public static abstract class Visitor<E> {
		boolean stop;
		public abstract boolean visit(E element);
	}
}
```

### 使用链表实现集合

自定义链表相关代码

```java
package com.mj.list;

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
	 * @return 原来的元素ֵ
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

```java
package com.mj.list;

public abstract class AbstractList<E> implements List<E>  {
	/**
	 * 元素的数量
	 */
	protected int size;
	/**
	 * 元素的数量
	 * @return
	 */
	public int size() {
		return size;
	}

	/**
	 * 是否为空
	 * @return
	 */
	public boolean isEmpty() {
		 return size == 0;
	}

	/**
	 * 是否包含某个元素
	 * @param element
	 * @return
	 */
	public boolean contains(E element) {
		return indexOf(element) != ELEMENT_NOT_FOUND;
	}

	/**
	 * 添加元素到尾部
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
package com.mj.list;

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

自定义链表实现集合 ListSet

```java
package com.mj.set;

import com.mj.list.LinkedList;
import com.mj.list.List;

public class ListSet<E> implements Set<E> {
	private List<E> list = new LinkedList<>();
	
	@Override
	public int size() {
		return list.size();
	}

	@Override
	public boolean isEmpty() {
		return list.isEmpty();
	}

	@Override
	public void clear() {
		list.clear();
	}

	@Override
	public boolean contains(E element) {
		return list.contains(element);
	}

	@Override
	public void add(E element) {
		int index = list.indexOf(element);
		if (index != List.ELEMENT_NOT_FOUND) { // 存在就覆盖
			list.set(index, element);
		} else { // 不存在就添加
			list.add(element);
		}
	}

	@Override
	public void remove(E element) {
		int index = list.indexOf(element);
		if (index != List.ELEMENT_NOT_FOUND) {
			list.remove(index);
		}
	}

	@Override
	public void traversal(Visitor<E> visitor) {
		if (visitor == null) return;
		
		int size = list.size();
		for (int i = 0; i < size; i++) {
			if (visitor.visit(list.get(i))) return;
		}
	}

}
```

### 使用红黑树实现集合

自定义红黑树相关代码

```java
package com.mj.tree;

import java.util.LinkedList;
import java.util.Queue;

public class BinaryTree<E> {
	protected int size;
	protected Node<E> root;
	
	public int size() {
		return size;
	}

	public boolean isEmpty() {
		return size == 0;
	}

	public void clear() {
		root = null;
		size = 0;
	}
	
	public void preorder(Visitor<E> visitor) {
		if (visitor == null) return;
		preorder(root, visitor);
	}
	
	private void preorder(Node<E> node, Visitor<E> visitor) {
		if (node == null || visitor.stop) return;
		
		visitor.stop = visitor.visit(node.element);
		preorder(node.left, visitor);
		preorder(node.right, visitor);
	}
	
	public void inorder(Visitor<E> visitor) {
		if (visitor == null) return;
		inorder(root, visitor);
	}
	
	private void inorder(Node<E> node, Visitor<E> visitor) {
		if (node == null || visitor.stop) return;
		
		inorder(node.left, visitor);
		if (visitor.stop) return;
		visitor.stop = visitor.visit(node.element);
		inorder(node.right, visitor);
	}
	
	public void postorder(Visitor<E> visitor) {
		if (visitor == null) return;
		postorder(root, visitor);
	}
	
	private void postorder(Node<E> node, Visitor<E> visitor) {
		if (node == null || visitor.stop) return;
		
		postorder(node.left, visitor);
		postorder(node.right, visitor);
		if (visitor.stop) return;
		visitor.stop = visitor.visit(node.element);
	}
	
	public void levelOrder(Visitor<E> visitor) {
		if (root == null || visitor == null) return;
		
		Queue<Node<E>> queue = new LinkedList<>();
		queue.offer(root);
		
		while (!queue.isEmpty()) {
			Node<E> node = queue.poll();
			if (visitor.visit(node.element)) return;
			
			if (node.left != null) {
				queue.offer(node.left);
			}
			
			if (node.right != null) {
				queue.offer(node.right);
			}
		}
	}
	
	public boolean isComplete() {
		if (root == null) return false;
		Queue<Node<E>> queue = new LinkedList<>();
		queue.offer(root);
		
		boolean leaf = false;
		while (!queue.isEmpty()) {
			Node<E> node = queue.poll();
			if (leaf && !node.isLeaf()) return false;

			if (node.left != null) {
				queue.offer(node.left);
			} else if (node.right != null) {
				return false;
			}
			
			if (node.right != null) {
				queue.offer(node.right);
			} else { // 后面遍历的节点都必须是叶子节点
				leaf = true;
			}
		}
		
		return true;
	}
	
	public int height() {
		if (root == null) return 0;
		
		// 树的高度
		int height = 0;
		// 存储着每一层的元素数量
		int levelSize = 1;
		Queue<Node<E>> queue = new LinkedList<>();
		queue.offer(root);
		
		while (!queue.isEmpty()) {
			Node<E> node = queue.poll();
			levelSize--;
			
			if (node.left != null) {
				queue.offer(node.left);
			}
			
			if (node.right != null) {
				queue.offer(node.right);
			}

			if (levelSize == 0) { // 意味着即将要访问下一层
				levelSize = queue.size();
				height++;
			}
		}
		
		return height;
	}
	
	public int height2() {
		return height(root);
	}
	
	private int height(Node<E> node) {
		if (node == null) return 0;
		return 1 + Math.max(height(node.left), height(node.right));
	}
	
	protected Node<E> createNode(E element, Node<E> parent) {
		return new Node<>(element, parent);
	}

	protected Node<E> predecessor(Node<E> node) {
		if (node == null) return null;
		
		// 前驱节点在左子树当中（left.right.right.right....）
		Node<E> p = node.left;
		if (p != null) {
			while (p.right != null) {
				p = p.right;
			}
			return p;
		}
		
		// 从父节点、祖父节点中寻找前驱节点
		while (node.parent != null && node == node.parent.left) {
			node = node.parent;
		}

		// node.parent == null
		// node == node.parent.right
		return node.parent;
	}
	
	protected Node<E> successor(Node<E> node) {
		if (node == null) return null;
		
		// 前驱节点在左子树当中（right.left.left.left....）
		Node<E> p = node.right;
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

	public static abstract class Visitor<E> {
		boolean stop;
		/**
		 * @return 如果返回true，就代表停止遍历
		 */
		public abstract boolean visit(E element);
	}
	
	protected static class Node<E> {
		E element;
		Node<E> left;
		Node<E> right;
		Node<E> parent;
		public Node(E element, Node<E> parent) {
			this.element = element;
			this.parent = parent;
		}
		
		public boolean isLeaf() {
			return left == null && right == null;
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
		
		public Node<E> sibling() {
			if (isLeftChild()) {
				return parent.right;
			}
			
			if (isRightChild()) {
				return parent.left;
			}
			
			return null;
		}
	}
}
```

```java
package com.mj.tree;

import java.util.Comparator;

@SuppressWarnings("unchecked")
public class BST<E> extends BinaryTree<E> {
	private Comparator<E> comparator;
	
	public BST() {
		this(null);
	}
	
	public BST(Comparator<E> comparator) {
		this.comparator = comparator;
	}

	public void add(E element) {
		elementNotNullCheck(element);
		
		// 添加第一个节点
		if (root == null) {
			root = createNode(element, null);
			size++;

			// 新添加节点之后的处理
			afterAdd(root);
			return;
		}
		
		// 添加的不是第一个节点
		// 找到父节点
		Node<E> parent = root;
		Node<E> node = root;
		int cmp = 0;
		do {
			cmp = compare(element, node.element);
			parent = node;
			if (cmp > 0) {
				node = node.right;
			} else if (cmp < 0) {
				node = node.left;
			} else { // 相等
				node.element = element;
				return;
			}
		} while (node != null);

		// 看看插入到父节点的哪个位置
		Node<E> newNode = createNode(element, parent);
		if (cmp > 0) {
			parent.right = newNode;
		} else {
			parent.left = newNode;
		}
		size++;
		
		// 新添加节点之后的处理
		afterAdd(newNode);
	}
	
	/**
	 * 添加node之后的调整
	 * @param node 新添加的节点
	 */
	protected void afterAdd(Node<E> node) { }
	
	/**
	 * 删除node之后的调整
	 * @param node 被删除的节点 或者 用以取代被删除节点的子节点（当被删除节点的度为1）
	 */
	protected void afterRemove(Node<E> node) { }

	public void remove(E element) {
		remove(node(element));
	}

	public boolean contains(E element) {
		return node(element) != null;
	}
	
	private void remove(Node<E> node) {
		if (node == null) return;
		
		size--;
		
		if (node.hasTwoChildren()) { // 度为2的节点
			// 找到后继节点
			Node<E> s = successor(node);
			// 用后继节点的值覆盖度为2的节点的值
			node.element = s.element;
			// 删除后继节点
			node = s;
		}
		
		// 删除node节点（node的度必然是1或者0）
		Node<E> replacement = node.left != null ? node.left : node.right;
		
		if (replacement != null) { // node是度为1的节点
			// 更改parent
			replacement.parent = node.parent;
			// 更改parent的left、right的指向
			if (node.parent == null) { // node是度为1的节点并且是根节点
				root = replacement;
			} else if (node == node.parent.left) {
				node.parent.left = replacement;
			} else { // node == node.parent.right
				node.parent.right = replacement;
			}
			
			// 删除节点之后的处理
			afterRemove(replacement);
		} else if (node.parent == null) { // node是叶子节点并且是根节点
			root = null;
			
			// 删除节点之后的处理
			afterRemove(node);
		} else { // node是叶子节点，但不是根节点
			if (node == node.parent.left) {
				node.parent.left = null;
			} else { // node == node.parent.right
				node.parent.right = null;
			}
			
			// 删除节点之后的处理
			afterRemove(node);
		}
	}
	
	private Node<E> node(E element) {
		Node<E> node = root;
		while (node != null) {
			int cmp = compare(element, node.element);
			if (cmp == 0) return node;
			if (cmp > 0) {
				node = node.right;
			} else { // cmp < 0
				node = node.left;
			}
		}
		return null;
	}
	
	/**
	 * @return 返回值等于0，代表e1和e2相等；返回值大于0，代表e1大于e2；返回值小于于0，代表e1小于e2
	 */
	private int compare(E e1, E e2) {
		if (comparator != null) {
			return comparator.compare(e1, e2);
		}
		return ((Comparable<E>)e1).compareTo(e2);
	}
	
	private void elementNotNullCheck(E element) {
		if (element == null) {
			throw new IllegalArgumentException("element must not be null");
		}
	}
}
```

```java
package com.mj.tree;

import java.util.Comparator;

public class BBST<E> extends BST<E> {
	public BBST() {
		this(null);
	}
	
	public BBST(Comparator<E> comparator) {
		super(comparator);
	}

	protected void rotateLeft(Node<E> grand) {
		Node<E> parent = grand.right;
		Node<E> child = parent.left;
		grand.right = child;
		parent.left = grand;
		afterRotate(grand, parent, child);
	}
	
	protected void rotateRight(Node<E> grand) {
		Node<E> parent = grand.left;
		Node<E> child = parent.right;
		grand.left = child;
		parent.right = grand;
		afterRotate(grand, parent, child);
	}
	
	protected void afterRotate(Node<E> grand, Node<E> parent, Node<E> child) {
		// 让parent称为子树的根节点
		parent.parent = grand.parent;
		if (grand.isLeftChild()) {
			grand.parent.left = parent;
		} else if (grand.isRightChild()) {
			grand.parent.right = parent;
		} else { // grand是root节点
			root = parent;
		}
		
		// 更新child的parent
		if (child != null) {
			child.parent = grand;
		}
		
		// 更新grand的parent
		grand.parent = parent;
	}
	
	protected void rotate(
			Node<E> r, // 子树的根节点
			Node<E> b, Node<E> c,
			Node<E> d,
			Node<E> e, Node<E> f) {
		// 让d成为这棵子树的根节点
		d.parent = r.parent;
		if (r.isLeftChild()) {
			r.parent.left = d;
		} else if (r.isRightChild()) {
			r.parent.right = d;
		} else {
			root = d;
		}
		
		//b-c
		b.right = c;
		if (c != null) {
			c.parent = b;
		}
		
		// e-f
		f.left = e;
		if (e != null) {
			e.parent = f;
		}
		
		// b-d-f
		d.left = b;
		d.right = f;
		b.parent = d;
		f.parent = d;
	}
}
```

```java
package com.mj.tree;

import java.util.Comparator;

public class RBTree<E> extends BBST<E> {
	private static final boolean RED = false;
	private static final boolean BLACK = true;
	
	public RBTree() {
		this(null);
	}
	
	public RBTree(Comparator<E> comparator) {
		super(comparator);
	}
	
	@Override
	protected void afterAdd(Node<E> node) {
		Node<E> parent = node.parent;
		
		// 添加的是根节点 或者 上溢到达了根节点
		if (parent == null) {
			black(node);
			return;
		}
		
		// 如果父节点是黑色，直接返回
		if (isBlack(parent)) return;
		
		// 叔父节点
		Node<E> uncle = parent.sibling();
		// 祖父节点
		Node<E> grand = red(parent.parent);
		if (isRed(uncle)) { // 叔父节点是红色【B树节点上溢】
			black(parent);
			black(uncle);
			// 把祖父节点当做是新添加的节点
			afterAdd(grand);
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
	
	@Override
	protected void afterRemove(Node<E> node) {
		// 如果删除的节点是红色
		// 或者 用以取代删除节点的子节点是红色
		if (isRed(node)) {
			black(node);
			return;
		}
		
		Node<E> parent = node.parent;
		// 删除的是根节点
		if (parent == null) return;
		
		// 删除的是黑色叶子节点【下溢】
		// 判断被删除的node是左还是右
		boolean left = parent.left == null || node.isLeftChild();
		Node<E> sibling = left ? parent.right : parent.left;
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
//	protected void afterRemove(Node<E> node, Node<E> replacement) {
//		// 如果删除的节点是红色
//		if (isRed(node)) return;
//		
//		// 用以取代node的子节点是红色
//		if (isRed(replacement)) {
//			black(replacement);
//			return;
//		}
//		
//		Node<E> parent = node.parent;
//		// 删除的是根节点
//		if (parent == null) return;
//		
//		// 删除的是黑色叶子节点【下溢】
//		// 判断被删除的node是左还是右
//		boolean left = parent.left == null || node.isLeftChild();
//		Node<E> sibling = left ? parent.right : parent.left;
//		if (left) { // 被删除的节点在左边，兄弟节点在右边
//			if (isRed(sibling)) { // 兄弟节点是红色
//				black(sibling);
//				red(parent);
//				rotateLeft(parent);
//				// 更换兄弟
//				sibling = parent.right;
//			}
//			
//			// 兄弟节点必然是黑色
//			if (isBlack(sibling.left) && isBlack(sibling.right)) {
//				// 兄弟节点没有1个红色子节点，父节点要向下跟兄弟节点合并
//				boolean parentBlack = isBlack(parent);
//				black(parent);
//				red(sibling);
//				if (parentBlack) {
//					afterRemove(parent, null);
//				}
//			} else { // 兄弟节点至少有1个红色子节点，向兄弟节点借元素
//				// 兄弟节点的左边是黑色，兄弟要先旋转
//				if (isBlack(sibling.right)) {
//					rotateRight(sibling);
//					sibling = parent.right;
//				}
//				
//				color(sibling, colorOf(parent));
//				black(sibling.right);
//				black(parent);
//				rotateLeft(parent);
//			}
//		} else { // 被删除的节点在右边，兄弟节点在左边
//			if (isRed(sibling)) { // 兄弟节点是红色
//				black(sibling);
//				red(parent);
//				rotateRight(parent);
//				// 更换兄弟
//				sibling = parent.left;
//			}
//			
//			// 兄弟节点必然是黑色
//			if (isBlack(sibling.left) && isBlack(sibling.right)) {
//				// 兄弟节点没有1个红色子节点，父节点要向下跟兄弟节点合并
//				boolean parentBlack = isBlack(parent);
//				black(parent);
//				red(sibling);
//				if (parentBlack) {
//					afterRemove(parent, null);
//				}
//			} else { // 兄弟节点至少有1个红色子节点，向兄弟节点借元素
//				// 兄弟节点的左边是黑色，兄弟要先旋转
//				if (isBlack(sibling.left)) {
//					rotateLeft(sibling);
//					sibling = parent.left;
//				}
//				
//				color(sibling, colorOf(parent));
//				black(sibling.left);
//				black(parent);
//				rotateRight(parent);
//			}
//		}
//	}

	private Node<E> color(Node<E> node, boolean color) {
		if (node == null) return node;
		((RBNode<E>)node).color = color;
		return node;
	}
	
	private Node<E> red(Node<E> node) {
		return color(node, RED);
	}
	
	private Node<E> black(Node<E> node) {
		return color(node, BLACK);
	}
	
	private boolean colorOf(Node<E> node) {
		return node == null ? BLACK : ((RBNode<E>)node).color;
	}
	
	private boolean isBlack(Node<E> node) {
		return colorOf(node) == BLACK;
	}
	
	private boolean isRed(Node<E> node) {
		return colorOf(node) == RED;
	}
	
	@Override
	protected Node<E> createNode(E element, Node<E> parent) {
		return new RBNode<>(element, parent);
	}

	private static class RBNode<E> extends Node<E> {
		boolean color = RED;
		public RBNode(E element, Node<E> parent) {
			super(element, parent);
		}
		
		@Override
		public String toString() {
			String str = "";
			if (color == RED) {
				str = "R_";
			}
			return str + element.toString();
		}
	}
}
```

自定义红黑树实现集合 TreeSet

```java
package com.mj.set;

import java.util.Comparator;

import com.mj.tree.BinaryTree;
import com.mj.tree.RBTree;

public class TreeSet<E> implements Set<E> {
	private RBTree<E> tree;
	
	public TreeSet() {
		this(null);
	}
	
	public TreeSet(Comparator<E> comparator) {
		tree = new RBTree<>(comparator);
	}
	
	@Override
	public int size() {
		return tree.size();
	}

	@Override
	public boolean isEmpty() {
		return tree.isEmpty();
	}

	@Override
	public void clear() {
		tree.clear();
	}

	@Override
	public boolean contains(E element) {
		return tree.contains(element);
	}

	@Override
	public void add(E element) {
		tree.add(element);
	}

	@Override
	public void remove(E element) {
		tree.remove(element);
	}

	@Override
	public void traversal(Visitor<E> visitor) {
		tree.inorder(new BinaryTree.Visitor<E>() {
			@Override
			public boolean visit(E element) {
				return visitor.visit(element);
			}
		});
	}
}
```



```java

```
```java

```


