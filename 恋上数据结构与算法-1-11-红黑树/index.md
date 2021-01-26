# 恋上数据结构与算法-1-11-红黑树


## 红黑树（Red Black Tree）

![红黑树](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-红黑树.png)  

**红黑树**也是一种自平衡的二叉搜索树
- 以前也叫做**平衡二叉 B 树**（Symmetric Binary B-tree）

红黑树必须满足以下 5 条性质
1. 节点是 RED 或者 BLACK
2. 根节点是 BLACK
3. 叶子节点（外部节点、空节点）都是 BLACK
4. RED 节点的子节点都是 BLACK
   - RED 节点的 parent 都是 BLACK
   - 从根节点到叶子节点的所有路径上不能有 2 个连续的 RED 节点
5. 从任一节点到叶子节点的所有路径都包含相同数目的 BLACK 节点

为何在这些规则下，就能保证平衡？

请问下面这棵是红黑树吗？

![错误的红黑树](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-错误的红黑树.png)  

不是红黑树，不符合第 5 条规律，38 节点右侧有个空的叶子节点，该路径只有 3 个黑色节点（包含空的叶子节点），其他路径都是 4 个黑色节点

## 红黑树的等价变换

![红黑树的等价变换](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-红黑树的等价变换.png)  

红黑树 和 4阶B树（2-3-4树）具有等价性

BLACK 节点与它的 RED 子节点融合在一起，形成 1 个 B 树节点

红黑树的 BLACK 节点个数 与 4阶B树 的节点总个数相等

网上有些教程：用2-3树 与 红黑树 进行类比，这是极其不严谨的，2-3树 并不能完美匹配红黑树的各种情况

注意：后面展示的红黑树都会省略 NULL 节点


## 红黑树 vs 2-3-4树

![红黑树vs2-3-4树](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-红黑树vs2-3-4树.png)  

思考：如果上图最底层的 BLACK 节点是不存在的，在 B 树中是什么样的情形？
- 整棵 B 树只有 1 个节点，而且是超级节点

## 红黑树中单词概念

![红黑树中单词概念](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-红黑树中单词概念.png)  

parent: 父节点

sibling: 兄弟节点

uncle: 叔父节点（ parent 的兄弟节点）

grand: 祖父节点（ parent 的父节点）

## 添加

![红黑树中添加元素](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-红黑树中添加元素.png)  

已知
- B 树中，新元素必定是添加到叶子节点中
- 4阶B树所有节点的元素个数 x 都符合 `1 <= x <= 3`

建议新添加的节点默认为 RED，这样能够让红黑树的性质尽快满足（性质 1、2、3、5 都满足，性质 4 不一定）

如果添加的是根节点，染成 BLACK 即可

红黑树添加节点的所有情况

![红黑树添加节点的所有情况](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-红黑树添加节点的所有情况.png)  

有 4 种情况满足红黑树的性质 4：parent 为 BLACK
- 同样也满足 4阶B树 的性质
- 因此不用做任何额外处理

![红黑树添加节点满足性质4](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-红黑树添加节点满足性质4.png)  

有 8 种情况不满足红黑树的性质 4：parent 为 RED （Double RED）
- 其中前 4 种属于 B 树节点上溢的情况

![红黑树添加节点不满足性质4](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-红黑树添加节点不满足性质4.png)  

注意：红黑树的平衡不是靠平衡因子保证的，是靠红黑树5条性质保证的，红黑树和AVL树没关系，红黑树节点没有高度的属性。

### 添加 - 修复性质4 - LL\RR

![红黑树添加节点修复性质4-1](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-红黑树添加节点修复性质4-1.png)  

判定条件：uncle 不是 RED
1. parent 染成 BLACK，grand 染成 RED
2. grand 进行单旋操作（对46进行左旋转，对76进行右旋转）
   - LL: 右旋转
   - RR: 左旋转

![红黑树添加节点修复性质4-2](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-红黑树添加节点修复性质4-2.png)  

### 添加 - 修复性质4 - LR\RL

![红黑树添加节点修复性质4-3](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-红黑树添加节点修复性质4-3.png)  

判定条件：uncle 不是 RED
1. 自己染成 BLACK，grand 染成 RED
2. 进行双旋操作
   - LR: parent 左旋转，grand 右旋转（72左旋转，76右旋转，74成为子树根节点）
   - RL: parent 右旋转，grand 左旋转（50右旋转，46左旋转，48成为子树根节点）

### 添加 - 修复性质4 - 上溢 - LL

![红黑树添加节点修复性质4-4](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-红黑树添加节点修复性质4-4.png)  

判定条件：uncle 是 RED
1. parent、uncle 染成 BLACK
2. grand 向上合并
   - 染成 RED，当作是新添加的节点进行处理

grand 向上合并时，可能继续发生上溢

若上溢持续到根节点，只需将根节点染成 BLACK

### 添加 - 修复性质4 - 上溢 - RR

![红黑树添加节点修复性质4-5](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-红黑树添加节点修复性质4-5.png)  

判定条件：uncle 是 RED
1. parent、uncle 染成 BLACK
2. grand 向上合并
   - 染成 RED，当做是新添加的节点进行处理

### 添加 - 修复性质4 - 上溢 - LR

![红黑树添加节点修复性质4-6](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-红黑树添加节点修复性质4-6.png)  

判定条件：uncle 是 RED
1. parent、uncle 染成 BLACK
2. grand 向上合并
   - 染成 RED，当作是新添加的节点进行处理

### 添加 - 修复性质4 - 上溢 - RL

![红黑树添加节点修复性质4-7](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-红黑树添加节点修复性质4-7.png)  

判定条件：uncle 是 RED
1. parent、uncle 染成 BLACK
2. grand 向上合并
   - 染成 RED，当作是新添加的节点进行处理


### 添加的实现

```java
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
```

继承结构的调整

二叉树 —> BST(二叉搜索树) -> BBST(平衡二叉搜索树) -> AVL树或红黑树

二叉树整体代码如下

```java
package org.msdemt.demo.tree;

import org.msdemt.demo.printer.BinaryTreeInfo;

import java.util.LinkedList;
import java.util.Queue;

/**
 * 二叉树
 *
 * @param <E>
 */
@SuppressWarnings("unchecked")
public class BinaryTree<E> implements BinaryTreeInfo {
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
        abstract boolean visit(E element);
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

    @Override
    public Object root() {
        return root;
    }

    @Override
    public Object left(Object node) {
        return ((Node<E>) node).left;
    }

    @Override
    public Object right(Object node) {
        return ((Node<E>) node).right;
    }

    @Override
    public Object string(Object node) {
        return node;
    }
}
```

二叉搜索树整体代码如下

```java
package org.msdemt.demo.tree;

import java.util.Comparator;

/**
 * 二叉搜索树
 *
 * @param <E>
 */
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
     *
     * @param node 新添加的节点
     */
    protected void afterAdd(Node<E> node) {
    }

    /**
     * 删除node之后的调整
     *
     * @param node 被删除的节点 或者 用以取代被删除节点的子节点（当被删除节点的度为1）
     */
    protected void afterRemove(Node<E> node) {
    }

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
        return ((Comparable<E>) e1).compareTo(e2);
    }

    private void elementNotNullCheck(E element) {
        if (element == null) {
            throw new IllegalArgumentException("element must not be null");
        }
    }
}
```

平衡二叉搜索树整体代码如下

```java
package org.msdemt.demo.tree;

import java.util.Comparator;

/**
 * 平衡二叉搜索树
 *
 * @param <E>
 */
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

AVL 树整体代码如下

```java
package org.msdemt.demo.tree;

import java.util.Comparator;

/**
 * AVL树
 *
 * @param <E>
 */
public class AVLTree<E> extends BBST<E> {
    public AVLTree() {
        this(null);
    }

    public AVLTree(Comparator<E> comparator) {
        super(comparator);
    }

    @Override
    protected void afterAdd(Node<E> node) {
        while ((node = node.parent) != null) {
            if (isBalanced(node)) {
                // 更新高度
                updateHeight(node);
            } else {
                // 恢复平衡
                rebalance(node);
                // 整棵树恢复平衡
                break;
            }
        }
    }

    @Override
    protected void afterRemove(Node<E> node) {
        while ((node = node.parent) != null) {
            if (isBalanced(node)) {
                // 更新高度
                updateHeight(node);
            } else {
                // 恢复平衡
                rebalance(node);
            }
        }
    }

    @Override
    protected Node<E> createNode(E element, Node<E> parent) {
        return new AVLNode<>(element, parent);
    }

    /**
     * 恢复平衡
     *
     * @param grand 高度最低的那个不平衡节点
     */
    @SuppressWarnings("unused")
    private void rebalance2(Node<E> grand) {
        Node<E> parent = ((AVLNode<E>) grand).tallerChild();
        Node<E> node = ((AVLNode<E>) parent).tallerChild();
        if (parent.isLeftChild()) { // L
            if (node.isLeftChild()) { // LL
                rotateRight(grand);
            } else { // LR
                rotateLeft(parent);
                rotateRight(grand);
            }
        } else { // R
            if (node.isLeftChild()) { // RL
                rotateRight(parent);
                rotateLeft(grand);
            } else { // RR
                rotateLeft(grand);
            }
        }
    }

    @Override
    protected void afterRotate(Node<E> grand, Node<E> parent, Node<E> child) {
        super.afterRotate(grand, parent, child);

        // 更新高度
        updateHeight(grand);
        updateHeight(parent);
    }

    @Override
    protected void rotate(Node<E> r, Node<E> b, Node<E> c, Node<E> d, Node<E> e, Node<E> f) {
        super.rotate(r, b, c, d, e, f);

        // 更新高度
        updateHeight(b);
        updateHeight(f);
        updateHeight(d);
    }

    /**
     * 恢复平衡
     *
     * @param grand 高度最低的那个不平衡节点
     */
    private void rebalance(Node<E> grand) {
        Node<E> parent = ((AVLNode<E>) grand).tallerChild();
        Node<E> node = ((AVLNode<E>) parent).tallerChild();
        if (parent.isLeftChild()) { // L
            if (node.isLeftChild()) { // LL
                rotate(grand, node, node.right, parent, parent.right, grand);
            } else { // LR
                rotate(grand, parent, node.left, node, node.right, grand);
            }
        } else { // R
            if (node.isLeftChild()) { // RL
                rotate(grand, grand, node.left, node, node.right, parent);
            } else { // RR
                rotate(grand, grand, parent.left, parent, node.left, node);
            }
        }
    }

    private boolean isBalanced(Node<E> node) {
        return Math.abs(((AVLNode<E>) node).balanceFactor()) <= 1;
    }

    private void updateHeight(Node<E> node) {
        ((AVLNode<E>) node).updateHeight();
    }

    private static class AVLNode<E> extends Node<E> {
        int height = 1;

        public AVLNode(E element, Node<E> parent) {
            super(element, parent);
        }

        public int balanceFactor() {
            int leftHeight = left == null ? 0 : ((AVLNode<E>) left).height;
            int rightHeight = right == null ? 0 : ((AVLNode<E>) right).height;
            return leftHeight - rightHeight;
        }

        public void updateHeight() {
            int leftHeight = left == null ? 0 : ((AVLNode<E>) left).height;
            int rightHeight = right == null ? 0 : ((AVLNode<E>) right).height;
            height = 1 + Math.max(leftHeight, rightHeight);
        }

        public Node<E> tallerChild() {
            int leftHeight = left == null ? 0 : ((AVLNode<E>) left).height;
            int rightHeight = right == null ? 0 : ((AVLNode<E>) right).height;
            if (leftHeight > rightHeight) return left;
            if (leftHeight < rightHeight) return right;
            return isLeftChild() ? left : right;
        }

        @Override
        public String toString() {
            String parentString = "null";
            if (parent != null) {
                parentString = parent.element.toString();
            }
            return element + "_p(" + parentString + ")_h(" + height + ")";
        }
    }
}
```

红黑树整体代码如下

```java
package org.msdemt.demo.tree;

import java.util.Comparator;

/**
 * 红黑树
 *
 * @param <E>
 */
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
        ((RBNode<E>) node).color = color;
        return node;
    }

    private Node<E> red(Node<E> node) {
        return color(node, RED);
    }

    private Node<E> black(Node<E> node) {
        return color(node, BLACK);
    }

    private boolean colorOf(Node<E> node) {
        return node == null ? BLACK : ((RBNode<E>) node).color;
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


## 删除

B 树中，最后真正被删除的元素都在叶子节点中

![B树的元素删除](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-B树的元素删除.png)  

### 删除 - RED 节点

![删除红色节点](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-删除红色节点.png)  

直接删除，不用作任何调整

### 删除 - BLACK 节点

![删除黑色节点](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-删除黑色节点.png)  

有 3 中情况
- 拥有 2 个 RED 子节点的 BLACK 节点
  - 不可能被直接删除，因为会找它的前驱或后继（本例为子节点）替代删除
  - 因此不用考虑这种情况
- 拥有 1 个 RED 子节点的 BLACK 节点
- BLACK 叶子节点

#### 删除 - 拥有 1 个 RED 子节点的 BLACK 节点

![删除拥有1个红色子节点的黑色节点](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-删除拥有1个红色子节点的黑色节点.png)  

判定条件：用以替代的子节点是 RED

将替代的子节点染成 BLACK 即可保持红黑树性质

#### 删除 - BLACK叶子节点 - sibling为BLACK

##### 兄弟节点能借一个节点的情况

BLACK 叶子节点被删除后，会导致B树节点下溢（比如删除88）

如果 sibling 至少有 1 个 RED 子节点
- 进行旋转操作
- 旋转之后的中心节点继承 parent 的颜色
- 旋转之后的左右节点染成 BLACK

![删除黑色叶子节点，兄弟节点为黑色1](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-删除黑色叶子节点，兄弟节点为黑色1.png)  

第三幅图有两种情况，LL 或 LR

![删除黑色叶子节点，兄弟节点为黑色2](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-删除黑色叶子节点，兄弟节点为黑色2.png)  

##### 兄弟节点不能借一个节点的情况

![兄弟节点不能借一个节点](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-兄弟节点不能借一个节点.png)  

判定条件：sibling 没有 1 个 RED 子节点

将 sibling 染成 RED，parent 染成 BLACK 即可修复红黑树性质

如果 parent 是 BLACK
- 会导致 parent 也下溢
- 这时只需要把 parent 当作被删除的节点处理即可

#### 删除 - BLACK叶子节点 - sibling为RED

如果 sibling 是 RED
- sibling 染成 BLACK，parent 染成 RED，进行旋转
- 于是又回到了 sibling 是 BLACK 的情况

![删除黑色叶子节点，兄弟节点为红色](../../../../images/恋上算法与数据结构/1-11-红黑树/20200921-删除黑色叶子节点，兄弟节点为红色.png)  

## 红黑树的平衡

最初遗留的困惑：为何那 5 条性质，就能保证红黑树是平衡的？
- 那 5 条性质，可以保证 红黑树 等价于 4阶B树

![红黑树的平衡](../../../../images/恋上算法与数据结构/1-11-红黑树/20200922-红黑树的平衡.png)  

相比 AVL树，红黑树的平衡标准比较宽松：没有一条路径会大于其他路径的 2 倍

红黑树的平衡是一种弱平衡，黑高度平衡

红黑树的最大高度是 2 * log<sub>2</sub>(n+1)，依然是 O(log n) 级别

## 红黑树的平均时间复杂度

搜索：O(log n)

添加：O(log n)，O(1)次的旋转操作

删除：O(log n)，O(1)次的旋转操作

## AVL树 vs 红黑树

AVL树
- 平衡标准比较严格：每个左右子树的高度差不能超过 1
- 最大高度是 1.44 * log<sub>2</sub>(n+2) - 1.328 （100W 个节点，AVL树最大树高28）
- 搜索、添加、删除都是 O(log n) 复杂度，其中添加仅需 O(1) 次旋转调整，删除最多需要 O(log n) 次旋转调整

红黑树
- 平衡标准比较宽松：没有一条路径会大于其他路径的 2 倍
- 最大高度是 2 * log<sub>2</sub>(n+1) （100W 个节点，红黑树最大树高40）
- 搜索、添加、删除都是 O(log n) 复杂度，其中添加、删除都仅需 O(1) 次旋转调整

如果搜索的次数远远大于插入和删除，选择AVL树；搜索、插入、删除次数几乎差不多，选择红黑树

相对于AVL树来说，红黑树牺牲了部分平衡性以换取插入/删除操作时少量的选择操作，整体来说性能要优于AVL树

红黑树的平均统计性能优于AVL树，实际应用中更多选择红黑树

## BST vs AVL Tree vs Red Black Tree

![树的对比](../../../../images/恋上算法与数据结构/1-11-红黑树/20200922-树的对比.png)  




