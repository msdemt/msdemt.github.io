# 恋上数据结构与算法-1-09-AVL树


## AVL树

AVL树是最早发明的自平衡二叉搜索树之一

AVL取名与两位发明者的名字
- G.M.Adelson-Velsky 和 E.M.Landis （来自苏联的科学家）

**平衡因子**（Balance Factor）：某结点的左右子树的高度差

AVL树的特点
- 每个节点的平衡因子只可能是 1、0、-1（绝对值 <= 1，如果超过 1，称之为“失衡”）
- 每个节点的左右子树高度差不超过 1
- 搜索、添加、删除的时间复杂度是 O(log n)

![AVL树的平衡](/images/恋上算法与数据结构/1-09-AVL树/20200918-AVL树的平衡.png)  

### 平衡对比

输入数据：35、37、34、56、25、62、57、9、74、32、94、80、75、100、16、82

![平衡对比](/images/恋上算法与数据结构/1-09-AVL树/20200918-平衡对比.png)  

### 简单的继承结构

![简单的继承结构](/images/恋上算法与数据结构/1-09-AVL树/20200918-简单的继承结构.png)  

二叉树实现

```java
package org.msdemt.demo.tree;

import org.msdemt.demo.printer.BinaryTreeInfo;

import java.util.LinkedList;
import java.util.Queue;

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

        int height = 0;
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

        Node<E> p = node.left;
        if (p != null) {
            while (p.right != null) {
                p = p.right;
            }
            return p;
        }

        while (node.parent != null && node == node.parent.left) {
            node = node.parent;
        }

        return node.parent;
    }

    protected Node<E> successor(Node<E> node) {
        if (node == null) return null;

        Node<E> p = node.right;
        if (p != null) {
            while (p.left != null) {
                p = p.left;
            }
            return p;
        }
        while (node.parent != null && node == node.parent.right) {
            node = node.parent;
        }
        return node.parent;
    }


    public static abstract class Visitor<E> {
        boolean stop;

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

二叉搜索树实现

```java
package org.msdemt.demo.tree;

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
     *
     * @param node 新添加的节点
     */
    protected void afterAdd(Node<E> node) {
    }

    /**
     * 删除node之后的调整
     *
     * @param node 被删除的节点
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
            afterRemove(node);
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

AVL树实现

```java
package org.msdemt.demo.tree;

import java.util.Comparator;

public class AVLTree<E> extends BST<E> {
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

    private void rotate(
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
        updateHeight(b);

        // e-f
        f.left = e;
        if (e != null) {
            e.parent = f;
        }
        updateHeight(f);

        // b-d-f
        d.left = b;
        d.right = f;
        b.parent = d;
        f.parent = d;
        updateHeight(d);
    }

    private void rotateLeft(Node<E> grand) {
        Node<E> parent = grand.right;
        Node<E> child = parent.left;
        grand.right = child;
        parent.left = grand;
        afterRotate(grand, parent, child);
    }

    private void rotateRight(Node<E> grand) {
        Node<E> parent = grand.left;
        Node<E> child = parent.right;
        grand.left = child;
        parent.right = grand;
        afterRotate(grand, parent, child);
    }

    private void afterRotate(Node<E> grand, Node<E> parent, Node<E> child) {
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

        // 更新高度
        updateHeight(grand);
        updateHeight(parent);
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

### 添加导致的失衡

示例：往下面这棵树中添加 13

最坏情况：可能会导致所有祖先节点都失衡

父节点、非祖先节点，都不可能失衡

![添加导致的失衡](/images/恋上算法与数据结构/1-09-AVL树/20200918-添加导致的失衡.png)  

#### LL - 右旋转（单旋）

T0 添加一个元素，会导致祖先节点 g 失衡

添加的节点（T1或T2上添加） 位于 失衡节点 g 的 左边、左边 方向，所以称为 LL

这种情况对 g 进行右旋转

![右旋转](/images/恋上算法与数据结构/1-09-AVL树/20200918-右旋转.png)  

解决办法：右旋转

- g.left = p.right
- p.right = g
- 让 p 成为这棵子树的根节点
- 仍然是一棵二叉搜索树：T0 < n < T1 < p < T2 < g < T3
- 整棵树都达到平衡

- 还需要注意维护的内容
  - T2、p、g 的 parent 属性
  - 先后更新 g、p 的高度

有些教程里，将右旋转叫做 zig，旋转之后的状态叫做 zigged

#### RR - 左旋转（单旋）

T3 添加一个元素，会导致祖先节点 g 失衡

添加的节点（T2或T3上添加） 位于 失衡节点 g 的 右边、右边 方向，所以称为 RR

这种情况对 g 进行左旋转

![左旋转](/images/恋上算法与数据结构/1-09-AVL树/20200918-左旋转.png)  

解决办法：左旋转

- g.right = p.left
- p.left = g
- 让 p 成为这棵子树的根节点
- 仍然是一棵二叉搜索树：T0 < g < T1 < p < T2 < n < T3
- 整棵树都达到平衡

- 还需要注意维护的内容
  - T1、p、g 的 parent 属性
  - 先后更新 g、p 的高度

有些教程里将左旋转叫做 zag，旋转之后的状态叫做 zagged

#### LR - RR左旋转，LL右旋转（双旋）

T2 添加元素，导致祖先节点 g 失衡

添加的节点（T1或T2） 位于 失衡节点 g 的 左边、右边 方向，所以称为 LR

这种情况先对 p 进行左旋转，让 n 成为子树的根节点，此时 g 是 LL 的情况，在对 g 进行一次右旋转，最终搞定

![先左旋再右旋](/images/恋上算法与数据结构/1-09-AVL树/20200918-先左旋再右旋.png)  

#### RL - LL右旋转，RR左旋转（双旋）

在失衡节点 g 的右方向、左方向添加节点导致的失衡，这种情况先对 p 进行一次右旋转，选择之后 n 成为子树根节点，这时 g 符合 RR 情况，对 g 进行一次左旋转

![先右旋再左旋](/images/恋上算法与数据结构/1-09-AVL树/20200918-先右旋再左旋.png)  

#### 实现

AVLNode 继承 二叉树的 Node，维护高度的属性

二叉树的 Node

```java
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
```

AVLNode的实现

```java
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
```

AVL中判断节点是否平衡

```java
private boolean isBalanced(Node<E> node) {
    return Math.abs(((AVLNode<E>) node).balanceFactor()) <= 1;
}
```

AVL中添加节点后更新高度

```java
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

private void updateHeight(Node<E> node) {
    ((AVLNode<E>) node).updateHeight();
}
```

恢复平衡

```java
/**
 * 恢复平衡
 *
 * @param grand 高度最低的那个不平衡节点
 */
@SuppressWarnings("unused")
private void rebalance(Node<E> grand) {
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

private void rotateLeft(Node<E> grand) {
    Node<E> parent = grand.right;
    Node<E> child = parent.left;
    grand.right = child;
    parent.left = grand;
    afterRotate(grand, parent, child);
}

private void rotateRight(Node<E> grand) {
    Node<E> parent = grand.left;
    Node<E> child = parent.right;
    grand.left = child;
    parent.right = grand;
    afterRotate(grand, parent, child);
}
```

恢复平衡的优化，统一旋转的操作

![统一旋转操作](/images/恋上算法与数据结构/1-09-AVL树/20200918-统一旋转操作.png)  

根据二叉搜索树的性质，a < b < c < d < e < f < g

最终恢复平衡的样子都是一样的，所以搞清楚对应的 a、b、c、d、e、f、g 是谁就可以了

```java
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

private void rotate(
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
    updateHeight(b);

    // e-f
    f.left = e;
    if (e != null) {
        e.parent = f;
    }
    updateHeight(f);

    // b-d-f
    d.left = b;
    d.right = f;
    b.parent = d;
    f.parent = d;
    updateHeight(d);
}
```

### 删除导致的失衡

示例：删除子树中的 16

可能会导致**父节点**或**祖先节点**失衡（只有 1 个节点会失衡），其他节点，都不可能失衡

![删除导致的失衡](/images/恋上算法与数据结构/1-09-AVL树/20200918-删除导致的失衡.png)  

#### LL - 右旋转（单旋）

假设删除 T3 的红色节点，此时节点 g 会失去平衡，因为此时属于 LL 情况，所以需要对 g 进行右旋转

![删除导致失衡-右旋转](/images/恋上算法与数据结构/1-09-AVL树/20200918-删除导致失衡-右旋转.png)  

如果绿色节点不存在，更高层的祖先节点可能也会失衡，需要再次恢复平衡，然后有可能导致更高层的祖先节点失衡...

极端情况下，所有祖先节点都需要进行恢复平衡的操作，共 O(log n) 次调整

#### RR - 左旋转（单旋）

![删除导致失衡-左旋转](/images/恋上算法与数据结构/1-09-AVL树/20200918-删除导致失衡-左旋转.png)  

#### LR - RR左旋转，LL右旋转（双旋）

![删除导致失衡-左旋右旋](/images/恋上算法与数据结构/1-09-AVL树/20200918-删除导致失衡-左旋右旋.png)  

#### RL - LL右旋转，RR左旋转（双旋）

![删除导致失衡-右旋左旋](/images/恋上算法与数据结构/1-09-AVL树/20200918-删除导致失衡-右旋左旋.png)  

#### 实现

```java
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
```



### 总结

添加
- 可能导致所有祖先节点都失衡
- 只要让高度最低的失衡节点恢复平衡，整棵树就恢复平衡（仅需 O(1) 次调整）

删除
- 可能会导致父节点或祖先节点失衡（只有 1 个节点会失衡）
- 恢复平衡后，可能会导致更高层的祖先节点失衡（最多需要 O(log n) 次调整）

平均复杂度
- 搜索：O(log n)
- 添加：O(log n)，仅需 O(1) 次的旋转操作
- 删除：O(log n)，最多需要 O(log n) 次的旋转操作

