# 恋上数据结构与算法-1-06-二叉树


## 树（Tree）的基本概念

![树](/images/恋上算法与数据结构/1-06-二叉树/20200916-树.png)  

- 节点、根节点、父节点、子节点、兄弟节点
- 一棵树可以没有任何节点，称为**空树**
- 一棵树可以只有1个节点，也就是只有根节点
- 子树、左子树、右子树

- 节点的**度**（degree）：子树的个数
- 树的**度**：所有节点度中的最大值
- **叶子**节点（leaf）：度为0的节点
- **非叶子**节点：度不为0的节点

- **层数**（level）：根节点在第1层，根节点的子节点在第2层，依此类推（有些教程也从第0层开始计算）
- 节点的**深度**（depth）：从根节点到当前节点的唯一路径上的节点总数
- 节点的**高度**（height）：从当前节点到最远叶子节点的路径上的节点总数
- 树的**深度**：所有节点深度中的最大值
- 树的**高度**：所有节点高度中的最大值
- 树的**深度**等于树的**高度**


- 有序树
  - 树中任意节点的子节点之间有顺序关系
- 无序树
  - 树中任意节点的子节点之间没有顺序关系
  - 无序树也成为“自由树”
- 森林
  - 由 m (m >= 0) 颗互不相交的树组成的集合

## 二叉树（Binary Tree）

二叉树的特点
- 每个节点的度最大为2（最多拥有2颗子树）
- 左子树和右子树是由顺序的
- 即使某节点只有一颗子树，也要区分左右子树

![二叉树1](/images/恋上算法与数据结构/1-06-二叉树/20200916-二叉树1.png)  

![二叉树2](/images/恋上算法与数据结构/1-06-二叉树/20200916-二叉树2.png)  

二叉树是有序树还是无序树？
- 有序树

### 二叉树的性质

![二叉树的性质](/images/恋上算法与数据结构/1-06-二叉树/20200916-二叉树的性质.png)  

1. 非空二叉树的第 i 层，最多有 2<sup>i-1</sup> 个节点（i >= 1）

2. 在高度为 h 的二叉树上最多有 2<sup>h</sup> - 1 个节点（h >= 1）

3. 对于任何一颗非空二叉树，如果叶子节点个数为 n0 ，度为 2 的节点个数为 n2 ，则有：n0 = n2 + 1
   - 假设度为 1 的节点个数为 n1 ，那么二叉树的节点总数 n = n0 + n1 + n2
   - 二叉树的边数 T = n1 + 2*n2 = n - 1 = n0 + n1 + n2 - 1 

### 真二叉树

**真二叉树**：所有节点的度都要么为 0 ，要么为 2

![真二叉树](/images/恋上算法与数据结构/1-06-二叉树/20200916-真二叉树.png)  

### 满二叉树（Full Binary Tree）

满二叉树：所有节点的度都要么为 0 ， 要么为 2，且所有的叶子节点都在最后一层

![满二叉树](/images/恋上算法与数据结构/1-06-二叉树/20200916-满二叉树.png)  

在同样高度的二叉树中，满二叉树的叶子节点数量最多，总节点数量最多

满二叉树一定是真二叉树，真二叉树不一定是满二叉树

假设满二叉树的高度为 h （h >= 1），那么
- 第 i 层的节点数量：2<sup>i-1</sup>
- 叶子节点数量：2<sup>h-1</sup>
- 总节点数量：n
  - n = 2<sup>h</sup> - 1 = 2<sup>0</sup> + 2<sup>1</sup> + 2<sup>2</sup> + ... + 2<sup>h-1</sup>
  - h = log<sub>2</sub>(n+1)

### 完全二叉树（Complete Binary Tree）

**完全二叉树**：叶子节点只会出现最后 2 层，且最后 1 层的叶子节点都靠左对齐

![完全二叉树](/images/恋上算法与数据结构/1-06-二叉树/20200916-完全二叉树.png)  

完全二叉树从根节点至倒数第 2 层是一颗满二叉树

满二叉树一定是完全二叉树，完全二叉树不一定是满二叉树

#### 完全二叉树的性质

1. 度为 1 的节点只有左子树

2. 度为 1 的节点要么是 1 个，要么是 0 个

3. 同样节点数量的二叉树，完全二叉树的高度最小

4. 假设完全二叉树的高度为 h (h >= 1)，那么
   - 至少有 2<sup>h-1</sup> 个节点 （2<sup>0</sup> + 2<sup>1</sup> + 2<sup>2</sup> + ... + 2<sup>h-2</sup> + 1）
   - 最多有 2<sup>h</sup> -1 个节点 （2<sup>0</sup> + 2<sup>1</sup> + 2<sup>2</sup> + ... + 2<sup>h-1</sup>，满二叉树）
   - 总节点数量为 n
     - 2<sup>h-1</sup> <= n < 2<sup>h</sup>
     - h - 1 <= log<sub>2</sub>n < h
     - h = floor(log<sub>2</sub>n) + 1

> floor（向下取整）：只取前面的整数，比如 floor(4.6) 为 4
> 
> ceiling（向上取整）：如果小数不为 0，取前面的整数加 1，否则只取前面的整数。比如 ceiling(4.6) 为 5，ceiling(4.0) 为 4
> 
> foor 和 ceiling 都是不管四舍五入规则的。

5. 一颗有 n 个节点的完全二叉树（n > 0），从上到下、从左到右对节点从 0 开始编号，对任意第 i 个节点
   - 如果 i = 0，它是根节点
   - 如果 i > 0，它的父节点编号为 floor((i-1)/2)
   - 如果 2i + 1 <= n - 1，它的左子节点编号为 2i + 1
   - 如果 2i + 1 > n - 1，它无左子节点
   - 如果 2i + 2 <= n - 1，它的右子节点编号为 2i + 2
   - 如果 2i + 2 > n - 1，它无右子节点

![完全二叉树性质](/images/恋上算法与数据结构/1-06-二叉树/20200916-完全二叉树性质.png)  

#### 面试题

如果一颗完全二叉树有 768 个节点，求叶子节点的个数

- 假设叶子节点个数为 n0，度为 1 的节点个数为 n1，度为 2 的节点个数为 n2
- 总节点个数 n = n0 + n1 + n2，而且 n0 = n2 + 1
  - n = 2n0 + n1 - 1
- 完全二叉树的 n1 要么为 0，要么为 1
  - n1 为 1 时，n = 2n0，n 必然是偶数
    - 叶子节点个数 n0 = n / 2，非叶子节点个数 n1 + n2 = n/2
  - n1 为 0 时，n = 2n0 - 1，n必然是奇数
    - 叶子节点个数 n0 = (n+1)/2，非叶子节点个数 n1 + n2 = (n-1)/2
- 叶子节点个数 n0 = floor((n+1)/2) = ceiling(n/2)
- 非叶子节点个数 n1 + n2 = floor(n/2) = ceiling((n-1)/2)
- 因此叶子节点个数为384

推导过程

- 总节点数量 n
- n 如果是偶数，叶子节点数量 n0 = n / 2
- n 如果是奇数，叶子节点数量 n0 = (n+1) / 2
- n0 = floor((n+1)/2) 或 n0 = ceiling(n/2)
- java语言默认是向下取整的，所以
- n0 = (n+1)/2
- n0 = (n+1) >> 1


### 二叉树的遍历

遍历时数据结构中的常见操作
- 把所有元素都访问一遍

线性数据结构的遍历比较简单
- 正序遍历
- 逆序遍历

根据节点访问顺序的不同，二叉树的常见遍历方式有4种
- 前序遍历（Preorder Traversal）
- 中序遍历（Inorder Traversal）
- 后序遍历（Postorder Traversal）
- 层序遍历（Level Order Traversal）

#### 前序遍历（Preorder Traversal）

![前序遍历](/images/恋上算法与数据结构/1-06-二叉树/20200918-前序遍历.png)  

访问顺序
- 根节点、前序遍历左子树、前序遍历右子树
- 7、4、2、1、3、5、9、8、11、10、12

在二叉搜索树中使用递归实现前序遍历

```java
public void preorder() {
    preorder(root);
}

private void preorder(Node<E> node) {
    if(node == null) return;

    System.out.println(node.element);
    preorder(node.left);
    preorder(node.right);
}
```

#### 中序遍历（Inorder Traversal）

![中序遍历](/images/恋上算法与数据结构/1-06-二叉树/20200918-中序遍历.png)  

访问顺序
- 中序遍历左子树、根节点、中序遍历右子树
- 1、2、3、4、5、6、7、8、9、10、11、12

如果访问顺序是下面这样呢？
- 中序遍历右子树、根节点、中序遍历左子树
- 12、11、10、9、8、7、6、5、4、3、2、1

二叉搜索树的中序遍历结果是升序或降序的

在二叉搜索树中使用递归实现中序遍历

中序遍历升序

```java
public void inorder() {
    inorder(root);
}

private void inorder(Node<E> node) {
    if(node == null) return;

    inorder(node.left);
    System.out.println(node.element);
    inorder(node.right);
}
```

中序遍历降序

```java
public void inorder() {
    inorder(root);
}

private void inorder(Node<E> node) {
    if(node == null) return;

    inorder(node.right);
    System.out.println(node.element);
    inorder(node.left);
}
```

#### 后序遍历（Postorder Traversal）

![后序遍历](/images/恋上算法与数据结构/1-06-二叉树/20200918-后序遍历.png)  

访问顺序
- 后续遍历左子树、后序遍历右子树、根节点
- 1、3、2、5、4、8、10、12、11、9、7

在二叉搜索树中使用递归实现后序遍历

```java
public void postorder() {
    postorder(root);
}

private void postorder(Node<E> node) {
    if(node == null) return;
    
    postorder(node.left);
    postorder(node.right);
    System.out.println(node.element);
}
```

#### 层序遍历（Level Order Traversal）

![层序遍历](/images/恋上算法与数据结构/1-06-二叉树/20200918-层序遍历.png)  

访问顺序
- 从上到下，从左到右依次访问每一个节点
- 7、4、9、2、5、8、11、1、3、10、12

实现思路：使用队列
1. 将根节点入队
2. 循环执行以下操作，直到队列为空
   - 将队头节点 A 出队，进行访问
   - 将 A 的左子节点入队
   - 将 A 的右子节点入队

使用队列实现中序遍历

```java
public void levelOrder() {
    if (root == null) return;
       
    Queue<Node<E>> queue = new LinkedList<>();
    queue.offer(root);

    while (!queue.isEmpty()) {
        Node<E> node = queue.poll();
        
        System.out.println(node.element);

        if (node.left != null) {
            queue.offer(node.left);
        }
        if (node.right != null) {
            queue.offer(node.right);
        }
    }
}
```


#### 设计遍历接口

思考：如果允许外界遍历二叉树元素（自定义遍历操作）？该如何设计接口？

通过visitor接口，用户可以自定义遍历二叉树元素时的操作

```java
public static interface Visitor<E> {
    void visit(E element);
}

public void inorder(Visitor<E> visitor) {
    if (visitor == null) return;

    inorder(root, visitor);
}

private void inorder(Node<E> node, Visitor<E> visitor) {
    if (node == null) return;

    inorder(node.left, visitor);
    visitor.visit(node.element);
    inorder(node.right, visitor);
}
```

用户遍历时传入visitor接口实现

```java
bst.inorder(new Visitor<Integer>() {
	public void visit(Integer element) {
		System.out.print("_" + element + "_ ");
	}
});
```

#### 增强遍历接口

遍历时，支持只遍历前几个元素，然后停止遍历，这种该怎么实现呢？

```java
public static abstract class Visitor<E> {
    boolean stop;

    /**
     * 如果返回true，表示停止遍历
     *
     * @param element
     * @return
     */
    public abstract boolean visit(E element);
}

public void inorder(Visitor<E> visitor) {
    if (visitor == null) {
        return;
    }
    inorder(root, visitor);
}

private void inorder(Node<E> node, Visitor<E> visitor) {
    if (node == null || visitor.stop) {
        return;
    }

    inorder(node.left, visitor);
    if (visitor.stop) {
        return;
    }
    visitor.stop = visitor.visit(node.element);
    inorder(node.right, visitor);
}
```

测试代码

```java
bst.inorder(new Visitor<Integer>() {
    @Override
    public boolean visit(Integer element) {
        System.out.print(element + " ");
        return element == 4 ? true : false;
    }
});
```

#### 遍历的应用

前序遍历
- 树状结构展示（注意左右子树的顺序）

中序遍历
- 二叉搜索树的中序遍历按升序或降序处理节点

后序遍历
- 适用于一些先子后父的操作

层序遍历
- 计算二叉树的高度
- 判断一棵树是否为完全二叉树


##### 计算二叉树的高度

递归方式实现获取二叉树的高度

```java
public int height2() {
    return height(root);
}

private int height(Node<E> node) {
    if (node == null) {
        return 0;
    }
    return 1 + Math.max(height(node.left), height(node.right));
}
```

非递归（迭代）方式实现获取二叉树的高度

```java
public int height() {
    if (root == null) {
        return 0;
    }

    //树的高度
    int height = 0;
    //存储每一层的元素数量
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

        if (levelSize == 0) {
            //意味着即将要访问下一层
            levelSize = queue.size();
            height++;
        }
    }
    return height;
}
```

##### 判断一棵树是否为完全二叉树

![完全二叉树](/images/恋上算法与数据结构/1-06-二叉树/20200918-完全二叉树.png)  

如果树为空，返回 false

如果树不为空，开始层序遍历二叉树（用队列）
- 如果 node.left != null && node.right != null，将 node.left、node.right按顺序入队
- 如果 node.left == null && node.right != null，返回 false
- 如果 node.left != null && node.right == null 或者 node.left == null && node.right == null ，则后面的节点都必须为叶子节点，否则返回 false

```java
public boolean isComplete() {
    if (root == null) {
        return false;
    }

    Queue<Node<E>> queue = new LinkedList<>();
    queue.offer(root);

    boolean leaf = false;

    while (!queue.isEmpty()) {
        Node<E> node = queue.poll();
        if (leaf && !node.isLeaf()) {
            return false;
        }

        if (node.left != null && node.right != null) {
            queue.offer(node.left);
            queue.offer(node.right);
        } else if (node.left == null && node.right != null) {
            return false;
        } else {
            //后面遍历的节点都必须是叶子节点
            leaf = true;
            if (node.left != null) {
                queue.offer(node.left);
            }
        }
    }
    return true;
}
```

优化

```java
/**
 * 判断是否为完全二叉树，较于上面的isComplete()方法可以减少重复判断
 * <p>
 * 如果树为空，返回false
 * 如果树不为空，开始层序遍历二叉树（用队列）
 * - 如果 node.left != null ，将 node.left 入队
 * - 如果 node.left == null && node.right != null，返回false
 * - 如果 node.right != null ，将 node.right 入队
 * - 如果 node.right == null，那么后面遍历的节点应该都为叶子节点，才是完全二叉树，否则返回false
 * - 遍历结束，返回true
 *
 * @return
 */
public boolean isComplete2() {
    if (root == null) {
        return false;
    }

    Queue<Node<E>> queue = new LinkedList<>();
    queue.offer(root);

    boolean leaf = false;
    while (!queue.isEmpty()) {
        Node<E> node = queue.poll();
        if (leaf && !node.isLeaf()) {
            return false;
        }

        if (node.left != null) {
            queue.offer(node.left);
        } else if (node.right != null) {
            //此时的情况为 node.left == null && node.right != null
            return false;
        }

        if (node.right != null) {
            queue.offer(node.right);
        } else {
            //此时的情况为 node.right == null
            leaf = true;
        }
    }
    return true;
}
```

##### 翻转二叉树

https://leetcode-cn.com/problems/invert-binary-tree/

```java
public class TreeNode {
	int val;
	TreeNode left;
	TreeNode right;
	TreeNode(int x) { 
		val = x; 
	}
}
```

前序遍历实现翻转二叉树

```java
public TreeNode invertTree(TreeNode root) {
   if (root == null) return root;
   
   TreeNode tmp = root.left;
   root.left = root.right;
   root.right = tmp;
   
    invertTree(root.left);
    invertTree(root.right);
    
    return root;
}
```

后序遍历实现翻转二叉树

```java
public TreeNode invertTree(TreeNode root) {
   if (root == null) return root;
   
    invertTree(root.left);
    invertTree(root.right);
   
   TreeNode tmp = root.left;
   root.left = root.right;
   root.right = tmp;
    
    return root;
 }
```

中序遍历实现翻转二叉树

```java
public TreeNode invertTree(TreeNode root) {
    if (root == null) return root;
   
    invertTree(root.left);

    TreeNode tmp = root.left;
    root.left = root.right;
    root.right = tmp;

    invertTree(root.left);
    
    return root;
 }
```

层序遍历实现翻转二叉树

```java
public TreeNode invertTree(TreeNode root) {
	if (root == null) return root;
	
	Queue<TreeNode> queue = new LinkedList<>();
	queue.offer(root);
	
	while (!queue.isEmpty()) {
		TreeNode node = queue.poll();
	    TreeNode tmp = node.left;
	    node.left = node.right;
	    node.right = tmp;
		
		if (node.left != null) {
			queue.offer(node.left);
		}
		
		if (node.right != null) {
			queue.offer(node.right);
		}
	}
	return root;
}
```

#### 根据遍历结果重构二叉树

以下结果可以保证重构出唯一的一颗二叉树
- 前序遍历 + 中序遍历
- 后序遍历 + 中序遍历

![根据遍历结果重构二叉树1](/images/恋上算法与数据结构/1-06-二叉树/20200918-根据遍历结果重构二叉树1.png)  

前后遍历 + 后序遍历
- 如果它是一颗真二叉树（Proper Binary Tree），结果是唯一的
- 不然结果不唯一

![根据遍历结果重构二叉树2](/images/恋上算法与数据结构/1-06-二叉树/20200918-根据遍历结果重构二叉树2.png)  

#### 前驱节点（predecessor）

前驱节点：中序遍历时的前一个节点
- 如果是二叉搜索树，前驱节点就是前一个比它小的节点

![前驱节点](/images/恋上算法与数据结构/1-06-二叉树/20200918-前驱节点.png)  

node.left != null
- 举例：6、13、8
- predecessor = node.left.right.right.right...
- 终止条件：right 为 null

node.left == null && node.parent != null
- 举例：7、11、9、1
- predecessor = node.parent.parent.parent...
- 终止条件：node 在 parent 的右子树中

node.left == null && node.parent == null
- 那就没有前驱节点
- 举例：没有左子树的根节点

代码实现

```java
/**
 * 查找node的前驱节点
 * <p>
 * 前驱节点：中序遍历时的前一个节点
 * 如果是二叉搜索树，前驱节点就是前一个比它小的节点
 * 判断条件：
 * 1. 如果node的左节点不为空，则node的前驱节点为node的左子树的右节点的右节点的右节点。。。直到右节点为空
 * predecessor = node.left.right.right.right...  终止条件 right为null
 * 2. 如果node的左节点为空，父节点不为空，则node的前驱节点为node的父节点的父节点的父节点。。。直到node在父节点的右子树中
 * predecessor = node.parent.parent.parent...  终止条件 node在parent的右子树中
 * 3. 如果node的左子树为空且父节点为空，则该node没有前驱节点，例如没有左子树的根节点
 *
 * @param node
 * @return
 */
private Node<E> predecessor(Node<E> node) {
    if (node == null) {
        return null;
    }
    //前驱节点在左子树中（left.right.right.right...）
    Node<E> p = node.left;
    if (p != null) {
        while (p.right != null) {
            p = p.right;
        }
        return p;
    }

    //node的左子树为空
    //从父节点、祖父节点中寻找前驱节点
    while (node.parent != null && node == node.parent.left) {
        node = node.parent;
    }
    //上面循环结束时的情况如下
    //node.parent == null 或者 node == node.parent.right
    //此时node.parent为前驱节点
    return node.parent;
}
```

#### 后继节点（successor）

后继节点：中序遍历时的后一个节点
- 如果是二叉搜索树，后继节点就是后一个比它大的节点

![后继节点](/images/恋上算法与数据结构/1-06-二叉树/20200918-后继节点.png) 

node.right != null
- 举例：1、8、4
- successor = node.right.left.left.left...
- 终止条件：left 为 null

node.right == null && node.parent != null
- 举例：7、6、3、11
- successor = node.parent.parent.parent...
- 终止条件：node 在 parent 的左子树中

node.right == null && node.parent == null
- 那就没有后继节点
- 举例：没有右子树的根节点

代码实现

```java
/**
 * 查找node的后继节点
 * <p>
 * 后继节点：中序遍历时的后一个节点
 * 如果是二叉搜索树，后继节点就是后一个比它大的节点
 * 判断条件
 * 1. 如果node的右节点不为空，则node的后继节点为node的右子树的左节点的左节点的左节点。。。直到左节点为空
 * successor = node.right.left.left.left... 终止条件 left为null
 * 2. 如果node的右节点为空，父节点不为空，则node的后继节点为node的父节点的父节点的父节点。。。直到node在父节点的左子树中
 * successor = node.parent.parent.parent... 终止条件 node在parent的左子树中
 * 3. 如果node的右子树为空且父节点为空，则该node没有前驱节点，例如没有右子树的根节点
 *
 * @param node
 * @return
 */
private Node<E> successor(Node<E> node) {
    if (node == null) {
        return null;
    }

    //后继节点在右子树中（right.left.left.left...）
    Node<E> p = node.right;
    if (p != null) {
        while (p.left != null) {
            p = p.left;
        }
        return p;
    }

    //从父节点、祖父节点中寻找前驱节点
    while (node.parent != null && node == node.parent.right) {
        node = node.parent;
    }

    return node.parent;
}
```



