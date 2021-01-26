# 恋上数据结构与算法-1-07-二叉搜索树


## 思考

在 n 个动态的整数中搜索某个整数？（查看其是否存在）

假设使用动态数组存放元素，从第 0 个位置开始遍历搜索，平均时间复杂度：O(n)

![思考1](../../../../images/恋上算法与数据结构/1-07-二叉搜索树/20200916-思考1.png)  

如果维护一个有序的动态数组，使用二分搜索，最坏时间复杂度：O(logn)
但是添加、删除的平均时间复杂度是O(n)
![思考2](../../../../images/恋上算法与数据结构/1-07-二叉搜索树/20200916-思考2.png)  

针对这个需求，有没有更好的方案？
- 使用二叉搜索树，添加、删除、搜索的最坏时间复杂度均可优化至：O(logn)

## 二叉搜索树（Binary Search Tree）

![二叉搜索树](../../../../images/恋上算法与数据结构/1-07-二叉搜索树/20200916-二叉搜索树.png)  

二叉搜索树是二叉树的一种，是应用非常广泛的一种二叉树，英文简称为 BST
- 又被称为：二叉查找树、二叉排序树
- 任意一个节点的值都大于其左子树所有节点的值
- 任意一个节点的值都小于其右子树所有节点的值
- 它的左右子树也是一颗二叉搜索树

二叉搜索树可以大大提高搜索数据的效率

二叉搜索树存储的元素必须具备可比较性
- 比如 int、 double 等
- 如果是自定义类型，需要指定比较方式
- 不允许为 null

### 二叉搜索树的接口设计

```java
int size();  //元素的数量
boolean isEmpty();  //是否为空
void clear();  //清空所有元素
void add(E element);  //添加元素
void remove(E element);  //删除元素
boolean contains(E element);  //是否包含某元素
```

需要注意的是
- 对于我们现在使用的二叉树来说，它的元素没有索引的概念


### 二叉搜索树的实现

#### 添加节点
- 找到父节点 parent
- 创建新节点 node
- parent.left = node 或者 parent.right = node 

遇到值相等的元素该如何处理？
- 建议覆盖旧的值

#### 元素的比较方案

1. 允许外界传入一个 Comparator 自定义比较方案
2. 如果没有传入 Comparator，强制认定元素实现了 Comparable 接口

#### 实现

```java
package org.msdemt.demo;

import org.msdemt.demo.printer.BinaryTreeInfo;

import java.util.Comparator;
import java.util.LinkedList;
import java.util.Queue;

/**
 * 二叉搜索树的实现
 * 
 * 继承BinaryTreeInfo是为了实现打印的功能
 * @param <E>
 */
public class BinarySearchTree<E> implements BinaryTreeInfo {
    private int size;
    private Node<E> root;
    private Comparator<E> comparator;

    private static class Node<E> {
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
    }

    public BinarySearchTree() {
        this(null);
    }

    public BinarySearchTree(Comparator<E> comparator) {
        this.comparator = comparator;
    }

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

    public void add(E element) {
        elementNotNullCheck(element);

        //添加第一个节点
        if (root == null) {
            root = new Node<>(element, null);
            size++;
            return;
        }

        //添加的不是第一个节点
        //找到父节点
        Node<E> parent = root;
        Node<E> node = root;
        int cmp = 0;
        do {
            cmp = compare(element, node.element);
            //保存父节点
            parent = node;
            if (cmp > 0) {
                node = node.right;
            } else if (cmp < 0) {
                node = node.left;
            } else {
                node.element = element;
                return;
            }
        } while (node != null);

        //看看插入到父节点的哪个位置
        Node<E> newNode = new Node<>(element, parent);
        if (cmp > 0) {
            parent.right = newNode;
        } else {
            parent.left = newNode;
        }
        size++;
    }

    public boolean contains(E element) {
        return node(element) != null;
    }

    public void remove(E element) {
        remove(node(element));
    }

    private void remove(Node<E> node) {
        if (node == null) {
            return;
        }

        size--;

        //度为2的节点
        if (node.hasTwoChildren()) {
            //找到后继节点
            Node<E> s = successor(node);
            //用后继节点的值覆盖度为2的节点的值，前驱或后继节点的度只能为0或1
            node.element = s.element;
            //删除后继节点，将后继节点s赋给node,在后续的流程中删除该度为0或1的节点
            node = s;
        }

        //删除node节点（此时node的度为1或者0）
        Node<E> replacement = node.left != null ? node.left : node.right;

        if (replacement != null) {
            //node是度为1的节点
            //更改parent
            replacement.parent = node.parent;
            //更改parent的left、right指向
            if (node.parent == null) {
                //node是度为1的节点并且是根节点
                root = replacement;
            } else if (node == node.parent.left) {
                node.parent.left = replacement;
            } else {
                //此时的情况为 node == node.parent.right
                node.parent.right = replacement;
            }
        } else if (node.parent == null) {
            //node是叶子节点并且是根节点
            root = null;
        } else {
            //node是叶子节点，但不是根节点
            if (node == node.parent.left) {
                node.parent.left = null;
            } else {
                //此时的情况为 node == node.parent.right
                node.parent.right = null;
            }
        }

    }

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

    /**
     * 根据元素找到node
     *
     * @param element
     * @return
     */
    private Node<E> node(E element) {
        Node<E> node = root;
        while (node != null) {
            int cmp = compare(element, node.element);
            if (cmp == 0) {
                return node;
            } else if (cmp > 0) {
                node = node.right;
            } else {
                node = node.right;
            }
        }
        return null;
    }

    /**
     * 返回值等于0，表示e1和e2相等；返回值大于0，表示e1大于e2；返回值小于0，表示e1小于e2
     *
     * @param e1
     * @param e2
     * @return
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
    //
    ///**
    // * 使用递归实现前序遍历
    // * 前序遍历访问顺序：根节点、前序遍历左子树、前序遍历右子树
    // */
    //public void preorderTraversal() {
    //    preorderTraversal(root);
    //}
    //
    //private void preorderTraversal(Node<E> node) {
    //    if (node == null) {
    //        return;
    //    }
    //    System.out.println(node.element);
    //    preorderTraversal(node.left);
    //    preorderTraversal(node.right);
    //}
    //
    ///**
    // * 使用递归实现中序遍历
    // * 中序遍历访问顺序：中序遍历左子树、根节点、中序遍历右子树
    // */
    //public void inorderTraversal() {
    //    inorderTraversal(root);
    //}
    //
    //private void inorderTraversal(Node<E> node) {
    //    if (node == null) {
    //        return;
    //    }
    //    inorderTraversal(node.left);
    //    System.out.println(node.element);
    //    inorderTraversal(node.right);
    //}
    //
    ///**
    // * 使用递归实现后续遍历
    // * 后续遍历访问顺序：后续遍历左子树、后续遍历右子树、根节点
    // */
    //public void postorderTraversal() {
    //    postorderTraversal(root);
    //}
    //
    //private void postorderTraversal(Node<E> node) {
    //    if (node == null) {
    //        return;
    //    }
    //    postorderTraversal(node.left);
    //    postorderTraversal(node.right);
    //    System.out.println(node.element);
    //}
    //
    ///**
    // * 使用队列Queue实现层序遍历
    // * 希望能默写出来
    // */
    //public void levelOrderTraversal() {
    //    if (root == null) {
    //        return;
    //    }
    //
    //    Queue<Node<E>> queue = new LinkedList<>();
    //    queue.offer(root);
    //
    //    while (!queue.isEmpty()) {
    //        Node<E> node = queue.poll();
    //        System.out.println(node.element);
    //
    //        if (node.left != null) {
    //            queue.offer(node.left);
    //        }
    //
    //        if (node.right != null) {
    //            queue.offer(node.right);
    //        }
    //    }
    //}

    /**
     * 递归实现前序遍历，支持外界遍历元素时自定义操作 visitor
     * 增强遍历接口，支持遍历到指定节点后停止 visitor.stop
     *
     * @param visitor
     */
    public void preorder(Visitor<E> visitor) {
        if (visitor == null) {
            return;
        }
        preorder(root, visitor);
    }

    private void preorder(Node<E> node, Visitor<E> visitor) {
        if (node == null || visitor.stop) {
            return;
        }
        //if (visitor.stop) {
        //    return;
        //}
        visitor.stop = visitor.visit(node.element);
        preorder(node.left, visitor);
        preorder(node.right, visitor);
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

    public void postorder(Visitor<E> visitor) {
        if (visitor == null) {
            return;
        }
        postorder(root, visitor);
    }

    private void postorder(Node<E> node, Visitor<E> visitor) {
        if (node == null || visitor.stop) {
            return;
        }
        postorder(node.left, visitor);
        postorder(node.right, visitor);
        if (visitor.stop) {
            return;
        }
        visitor.stop = visitor.visit(node.element);
    }

    public void levelOrder(Visitor<E> visitor) {
        if (root == null || visitor == null) {
            return;
        }

        Queue<Node<E>> queue = new LinkedList<>();
        queue.offer(root);

        while (!queue.isEmpty()) {
            Node<E> node = queue.poll();
            if (visitor.visit(node.element)) {
                //遍历到指定数据返回true，停止遍历
                return;
            }
            if (node.left != null) {
                queue.offer(node.left);
            }
            if (node.right != null) {
                queue.offer(node.right);
            }
        }
    }

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

    /**
     * 获取树的高度
     *
     * @return
     */
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

    /**
     * 获取树的高度
     * 方法二，使用递归方式
     *
     * @return
     */
    public int height2() {
        return height(root);
    }

    private int height(Node<E> node) {
        if (node == null) {
            return 0;
        }
        return 1 + Math.max(height(node.left), height(node.right));
    }

    /**
     * 判断是否为完全二叉树
     * <p>
     * 如果树为空，返回false
     * 如果树不为空，开始层序遍历二叉树（用队列）
     * - 如果 node.left != null && node.right != null，将 node.left、node.right按顺序入队
     * - 如果 node.left == null && node.right != null，返回 false
     * - 如果 node.left != null && node.right == null 或者 node.left == null && node.right == null ，则后面的节点都必须为叶子节点
     *
     * @return
     */
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

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder();
        toString(root, sb, "");
        return sb.toString();
    }

    private void toString(Node<E> node, StringBuilder sb, String prefix) {
        if (node == null) {
            return;
        }

        toString(node.left, sb, prefix + "L---");
        sb.append(prefix).append(node.element).append("\n");
        toString(node.right, sb, prefix + "R---");
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
        Node<E> myNode = (Node<E>) node;
        String parentString = "null";
        if (myNode.parent != null) {
            parentString = myNode.parent.element.toString();
        }
        return myNode.element + "_P(" + parentString + ")";
    }
}
```


测试方法

```java
package org.msdemt.demo;


import org.msdemt.demo.file.Files;
import org.msdemt.demo.printer.BinaryTreeInfo;
import org.msdemt.demo.printer.BinaryTrees;

import java.util.Comparator;

@SuppressWarnings("unused")
public class Main {

    private static class PersonComparator implements Comparator<Person> {
        @Override
        public int compare(Person e1, Person e2) {
            return e1.getAge() - e2.getAge();
        }
    }

    private static class PersonComparator2 implements Comparator<Person> {
        @Override
        public int compare(Person e1, Person e2) {
            return e2.getAge() - e1.getAge();
        }
    }

    static void test1() {
        Integer data[] = new Integer[]{
                7, 4, 9, 2, 5, 8, 11, 3, 12, 1
        };

        BinarySearchTree<Integer> bst = new BinarySearchTree<>();
        for (int i = 0; i < data.length; i++) {
            bst.add(data[i]);
        }

        BinaryTrees.println(bst);
    }

    static void test2() {
        Integer data[] = new Integer[]{
                7, 4, 9, 2, 5, 8, 11, 3, 12, 1
        };

        BinarySearchTree<Person> bst1 = new BinarySearchTree<>();
        for (int i = 0; i < data.length; i++) {
            bst1.add(new Person(data[i]));
        }

        BinaryTrees.println(bst1);

        BinarySearchTree<Person> bst2 = new BinarySearchTree<>(new Comparator<Person>() {
            @Override
            public int compare(Person o1, Person o2) {
                return o2.getAge() - o1.getAge();
            }
        });
        for (int i = 0; i < data.length; i++) {
            bst2.add(new Person(data[i]));
        }
        BinaryTrees.println(bst2);
    }

    static void test3() {
        BinarySearchTree<Integer> bst = new BinarySearchTree<>();
        for (int i = 0; i < 40; i++) {
            bst.add((int) (Math.random() * 100));
        }

        String str = BinaryTrees.printString(bst);
        str += "\n";
        Files.writeToFile("F:/1.txt", str, true);

        // BinaryTrees.println(bst);
    }

    static void test4() {
        BinaryTrees.println(new BinaryTreeInfo() {

            @Override
            public Object string(Object node) {
                return node.toString() + "_";
            }

            @Override
            public Object root() {
                return "A";
            }

            @Override
            public Object right(Object node) {
                if (node.equals("A")) {
                    return "C";
                }
                if (node.equals("C")) {
                    return "E";
                }
                return null;
            }

            @Override
            public Object left(Object node) {
                if (node.equals("A")) {
                    return "B";
                }
                if (node.equals("C")) {
                    return "D";
                }
                return null;
            }
        });

        // test3();


        /*
         * Java的匿名类，类似于iOS中的Block、JS中的闭包（function）
         */

//		BinarySearchTree<Person> bst1 = new BinarySearchTree<>(new Comparator<Person>() {
//			@Override
//			public int compare(Person o1, Person o2) {
//				return 0;
//			}
//		});
//		bst1.add(new Person(12));
//		bst1.add(new Person(15));
//		
//		BinarySearchTree<Person> bst2 = new BinarySearchTree<>(new PersonComparator());
//		bst2.add(new Person(12));
//		bst2.add(new Person(15));
//


//		BinarySearchTree<Car> bst4 = new BinarySearchTree<>(new Car);
//		
//		
//		java.util.Comparator<Integer> iComparator;
    }

    static void test5() {
        BinarySearchTree<Person> bst = new BinarySearchTree<>();
        bst.add(new Person(10, "jack"));
        bst.add(new Person(12, "rose"));
        bst.add(new Person(6, "jim"));

        bst.add(new Person(10, "michael"));

        BinaryTrees.println(bst);
    }

    static void test6() {
        Integer data[] = new Integer[]{
                7, 4, 9, 2, 5
        };

        BinarySearchTree<Integer> bst = new BinarySearchTree<>();
        for (int i = 0; i < data.length; i++) {
            bst.add(data[i]);
        }

//		BinarySearchTree<Integer> bst = new BinarySearchTree<>();
//		for (int i = 0; i < 10; i++) {
//			bst.add((int)(Math.random() * 100));
//		}
        BinaryTrees.println(bst);
        System.out.println(bst.isComplete());

        // bst.levelOrderTraversal();
		
		/*
		 *       7
		 *    4    9
		    2   5
		 */

//		bst.levelOrder(new Visitor<Integer>() {
//			public void visit(Integer element) {
//				System.out.print("_" + element + "_ ");
//			}
//		});

//		bst.inorder(new Visitor<Integer>() {
//			public void visit(Integer element) {
//				System.out.print("_" + (element + 3) + "_ ");
//			}
//		});

        // System.out.println(bst.height());
    }

    static void test7() {
        Integer data[] = new Integer[]{
                7, 4, 9, 2, 5, 8, 11, 3, 12, 1
        };

        BinarySearchTree<Integer> bst = new BinarySearchTree<>();
        for (int i = 0; i < data.length; i++) {
            bst.add(data[i]);
        }

        BinaryTrees.println(bst);

        bst.remove(7);

        BinaryTrees.println(bst);
    }

    static void test8() {
        Integer data[] = new Integer[]{
                7, 4, 9, 2, 1
        };

        BinarySearchTree<Integer> bst = new BinarySearchTree<>();
        for (int i = 0; i < data.length; i++) {
            bst.add(data[i]);
        }
        BinaryTrees.println(bst);
        System.out.println(bst.isComplete());
    }

    static void test9() {
        Integer data[] = new Integer[]{
                7, 4, 9, 2, 1
        };

        BinarySearchTree<Integer> bst = new BinarySearchTree<>();
        for (int i = 0; i < data.length; i++) {
            bst.add(data[i]);
        }
        BinaryTrees.println(bst);

        bst.preorder(new BinarySearchTree.Visitor<Integer>() {
            @Override
            public boolean visit(Integer element) {
                System.out.print(element + " ");
                return element == 2 ? true : false;
            }
        });
        System.out.println();

        bst.inorder(new BinarySearchTree.Visitor<Integer>() {
            @Override
            public boolean visit(Integer element) {
                System.out.print(element + " ");
                return element == 4 ? true : false;
            }
        });
        System.out.println();

        bst.postorder(new BinarySearchTree.Visitor<Integer>() {
            @Override
            public boolean visit(Integer element) {
                System.out.print(element + " ");
                return element == 4 ? true : false;
            }
        });
        System.out.println();

        bst.levelOrder(new BinarySearchTree.Visitor<Integer>() {
            @Override
            public boolean visit(Integer element) {
                System.out.print(element + " ");
                return element == 9 ? true : false;
            }
        });
        System.out.println();
    }

    public static void main(String[] args) {
        test9();
    }
}
```

#### 打印二叉搜索树

工具：https://github.com/CoderMJLee/BinaryTrees

使用步骤
- 实现 BinaryTreeInfo 接口
- 调用打印 API
  - BinaryTrees.println(bst);

```java
package org.msdemt.demo.printer;

public interface BinaryTreeInfo {
    /**
     * who is the root node
     */
    Object root();

    /**
     * how to get the left child of the node
     */
    Object left(Object node);

    /**
     * how to get the right child of the node
     */
    Object right(Object node);

    /**
     * how to print the node
     */
    Object string(Object node);
}
```

```java
package org.msdemt.demo.printer;

/**
 * @author MJ Lee
 */
public final class BinaryTrees {

    private BinaryTrees() {
    }

    public static void print(BinaryTreeInfo tree) {
        print(tree, null);
    }

    public static void println(BinaryTreeInfo tree) {
        println(tree, null);
    }

    public static void print(BinaryTreeInfo tree, PrintStyle style) {
        if (tree == null || tree.root() == null) return;
        printer(tree, style).print();
    }

    public static void println(BinaryTreeInfo tree, PrintStyle style) {
        if (tree == null || tree.root() == null) return;
        printer(tree, style).println();
    }

    public static String printString(BinaryTreeInfo tree) {
        return printString(tree, null);
    }

    public static String printString(BinaryTreeInfo tree, PrintStyle style) {
        if (tree == null || tree.root() == null) return null;
        return printer(tree, style).printString();
    }

    private static Printer printer(BinaryTreeInfo tree, PrintStyle style) {
        if (style == PrintStyle.INORDER) return new InorderPrinter(tree);
        return new LevelOrderPrinter(tree);
    }

    public enum PrintStyle {
        LEVEL_ORDER, INORDER
    }
}
```

```java
package org.msdemt.demo.printer;

/**
 * ┌──800
 * ┌──760
 * │   └──600
 * ┌──540
 * │   └──476
 * │       └──445
 * ┌──410
 * │   └──394
 * 381
 * │     ┌──190
 * │     │   └──146
 * │  ┌──40
 * │  │  └──35
 * └──12
 * └──9
 *
 * @author MJ Lee
 */
public class InorderPrinter extends Printer {
    private static String rightAppend;
    private static String leftAppend;
    private static String blankAppend;
    private static String lineAppend;

    static {
        int length = 2;
        rightAppend = "┌" + Strings.repeat("─", length);
        leftAppend = "└" + Strings.repeat("─", length);
        blankAppend = Strings.blank(length + 1);
        lineAppend = "│" + Strings.blank(length);
    }

    public InorderPrinter(BinaryTreeInfo tree) {
        super(tree);
    }

    @Override
    public String printString() {
        StringBuilder string = new StringBuilder(
                printString(tree.root(), "", "", ""));
        string.deleteCharAt(string.length() - 1);
        return string.toString();
    }

    /**
     * 生成node节点的字符串
     *
     * @param nodePrefix  node那一行的前缀字符串
     * @param leftPrefix  node整棵左子树的前缀字符串
     * @param rightPrefix node整棵右子树的前缀字符串
     * @return
     */
    private String printString(
            Object node,
            String nodePrefix,
            String leftPrefix,
            String rightPrefix) {
        Object left = tree.left(node);
        Object right = tree.right(node);
        String string = tree.string(node).toString();

        int length = string.length();
        if (length % 2 == 0) {
            length--;
        }
        length >>= 1;

        String nodeString = "";
        if (right != null) {
            rightPrefix += Strings.blank(length);
            nodeString += printString(right,
                    rightPrefix + rightAppend,
                    rightPrefix + lineAppend,
                    rightPrefix + blankAppend);
        }
        nodeString += nodePrefix + string + "\n";
        if (left != null) {
            leftPrefix += Strings.blank(length);
            nodeString += printString(left,
                    leftPrefix + leftAppend,
                    leftPrefix + blankAppend,
                    leftPrefix + lineAppend);
        }
        return nodeString;
    }
}
```

```java
package org.msdemt.demo.printer;

import java.util.*;

/**
 * ┌───381────┐
 * │          │
 * ┌─12─┐     ┌─410─┐
 * │    │     │     │
 * 9  ┌─40─┐ 394 ┌─540─┐
 * │    │     │     │
 * 35 ┌─190 ┌─476 ┌─760─┐
 * │     │     │     │
 * 146   445   600   800
 *
 * @author MJ Lee
 */
public class LevelOrderPrinter extends Printer {
    /**
     * 节点之间允许的最小间距（最小只能填1）
     */
    private static final int MIN_SPACE = 1;
    private Node root;
    private int minX;
    private int maxWidth;

    public LevelOrderPrinter(BinaryTreeInfo tree) {
        super(tree);

        root = new Node(tree.root(), tree);
        maxWidth = root.width;
    }

    @Override
    public String printString() {
        // nodes用来存放所有的节点
        List<List<Node>> nodes = new ArrayList<>();
        fillNodes(nodes);
        cleanNodes(nodes);
        compressNodes(nodes);
        addLineNodes(nodes);

        int rowCount = nodes.size();

        // 构建字符串
        StringBuilder string = new StringBuilder();
        for (int i = 0; i < rowCount; i++) {
            if (i != 0) {
                string.append("\n");
            }

            List<Node> rowNodes = nodes.get(i);
            StringBuilder rowSb = new StringBuilder();
            for (Node node : rowNodes) {
                int leftSpace = node.x - rowSb.length() - minX;
                rowSb.append(Strings.blank(leftSpace));
                rowSb.append(node.string);
            }

            string.append(rowSb);
        }

        return string.toString();
    }

    /**
     * 添加一个元素节点
     */
    private Node addNode(List<Node> nodes, Object btNode) {
        Node node = null;
        if (btNode != null) {
            node = new Node(btNode, tree);
            maxWidth = Math.max(maxWidth, node.width);
            nodes.add(node);
        } else {
            nodes.add(null);
        }
        return node;
    }

    /**
     * 以满二叉树的形式填充节点
     */
    private void fillNodes(List<List<Node>> nodes) {
        if (nodes == null) return;
        // 第一行
        List<Node> firstRowNodes = new ArrayList<>();
        firstRowNodes.add(root);
        nodes.add(firstRowNodes);

        // 其他行
        while (true) {
            List<Node> preRowNodes = nodes.get(nodes.size() - 1);
            List<Node> rowNodes = new ArrayList<>();

            boolean notNull = false;
            for (Node node : preRowNodes) {
                if (node == null) {
                    rowNodes.add(null);
                    rowNodes.add(null);
                } else {
                    Node left = addNode(rowNodes, tree.left(node.btNode));
                    if (left != null) {
                        node.left = left;
                        left.parent = node;
                        notNull = true;
                    }

                    Node right = addNode(rowNodes, tree.right(node.btNode));
                    if (right != null) {
                        node.right = right;
                        right.parent = node;
                        notNull = true;
                    }
                }
            }

            // 全是null，就退出
            if (!notNull) break;
            nodes.add(rowNodes);
        }
    }

    /**
     * 删除全部null、更新节点的坐标
     */
    private void cleanNodes(List<List<Node>> nodes) {
        if (nodes == null) return;

        int rowCount = nodes.size();
        if (rowCount < 2) return;

        // 最后一行的节点数量
        int lastRowNodeCount = nodes.get(rowCount - 1).size();

        // 每个节点之间的间距
        int nodeSpace = maxWidth + 2;

        // 最后一行的长度
        int lastRowLength = lastRowNodeCount * maxWidth
                + nodeSpace * (lastRowNodeCount - 1);

        // 空集合
        Collection<Object> nullSet = Collections.singleton(null);

        for (int i = 0; i < rowCount; i++) {
            List<Node> rowNodes = nodes.get(i);

            int rowNodeCount = rowNodes.size();
            // 节点左右两边的间距
            int allSpace = lastRowLength - (rowNodeCount - 1) * nodeSpace;
            int cornerSpace = allSpace / rowNodeCount - maxWidth;
            cornerSpace >>= 1;

            int rowLength = 0;
            for (int j = 0; j < rowNodeCount; j++) {
                if (j != 0) {
                    // 每个节点之间的间距
                    rowLength += nodeSpace;
                }
                rowLength += cornerSpace;
                Node node = rowNodes.get(j);
                if (node != null) {
                    // 居中（由于奇偶数的问题，可能有1个符号的误差）
                    int deltaX = (maxWidth - node.width) >> 1;
                    node.x = rowLength + deltaX;
                    node.y = i;
                }
                rowLength += maxWidth;
                rowLength += cornerSpace;
            }
            // 删除所有的null
            rowNodes.removeAll(nullSet);
        }
    }

    /**
     * 压缩空格
     */
    private void compressNodes(List<List<Node>> nodes) {
        if (nodes == null) return;

        int rowCount = nodes.size();
        if (rowCount < 2) return;

        for (int i = rowCount - 2; i >= 0; i--) {
            List<Node> rowNodes = nodes.get(i);
            for (Node node : rowNodes) {
                Node left = node.left;
                Node right = node.right;
                if (left == null && right == null) continue;
                if (left != null && right != null) {
                    // 让左右节点对称
                    node.balance(left, right);

                    // left和right之间可以挪动的最小间距
                    int leftEmpty = node.leftBoundEmptyLength();
                    int rightEmpty = node.rightBoundEmptyLength();
                    int empty = Math.min(leftEmpty, rightEmpty);
                    empty = Math.min(empty, (right.x - left.rightX()) >> 1);

                    // left、right的子节点之间可以挪动的最小间距
                    int space = left.minLevelSpaceToRight(right) - MIN_SPACE;
                    space = Math.min(space >> 1, empty);

                    // left、right往中间挪动
                    if (space > 0) {
                        left.translateX(space);
                        right.translateX(-space);
                    }

                    // 继续挪动
                    space = left.minLevelSpaceToRight(right) - MIN_SPACE;
                    if (space < 1) continue;

                    // 可以继续挪动的间距
                    leftEmpty = node.leftBoundEmptyLength();
                    rightEmpty = node.rightBoundEmptyLength();
                    if (leftEmpty < 1 && rightEmpty < 1) continue;

                    if (leftEmpty > rightEmpty) {
                        left.translateX(Math.min(leftEmpty, space));
                    } else {
                        right.translateX(-Math.min(rightEmpty, space));
                    }
                } else if (left != null) {
                    left.translateX(node.leftBoundEmptyLength());
                } else { // right != null
                    right.translateX(-node.rightBoundEmptyLength());
                }
            }
        }
    }

    private void addXLineNode(List<Node> curRow, Node parent, int x) {
        Node line = new Node("─");
        line.x = x;
        line.y = parent.y;
        curRow.add(line);
    }

    private Node addLineNode(List<Node> curRow, List<Node> nextRow, Node parent, Node child) {
        if (child == null) return null;

        Node top = null;
        int topX = child.topLineX();
        if (child == parent.left) {
            top = new Node("┌");
            curRow.add(top);

            for (int x = topX + 1; x < parent.x; x++) {
                addXLineNode(curRow, parent, x);
            }
        } else {
            for (int x = parent.rightX(); x < topX; x++) {
                addXLineNode(curRow, parent, x);
            }

            top = new Node("┐");
            curRow.add(top);
        }

        // 坐标
        top.x = topX;
        top.y = parent.y;
        child.y = parent.y + 2;
        minX = Math.min(minX, child.x);

        // 竖线
        Node bottom = new Node("│");
        bottom.x = topX;
        bottom.y = parent.y + 1;
        nextRow.add(bottom);

        return top;
    }

    private void addLineNodes(List<List<Node>> nodes) {
        List<List<Node>> newNodes = new ArrayList<>();

        int rowCount = nodes.size();
        if (rowCount < 2) return;

        minX = root.x;

        for (int i = 0; i < rowCount; i++) {
            List<Node> rowNodes = nodes.get(i);
            if (i == rowCount - 1) {
                newNodes.add(rowNodes);
                continue;
            }

            List<Node> newRowNodes = new ArrayList<>();
            newNodes.add(newRowNodes);

            List<Node> lineNodes = new ArrayList<>();
            newNodes.add(lineNodes);
            for (Node node : rowNodes) {
                addLineNode(newRowNodes, lineNodes, node, node.left);
                newRowNodes.add(node);
                addLineNode(newRowNodes, lineNodes, node, node.right);
            }
        }

        nodes.clear();
        nodes.addAll(newNodes);
    }

    private static class Node {
        /**
         * 顶部符号距离父节点的最小距离（最小能填0）
         */
        private static final int TOP_LINE_SPACE = 1;

        Object btNode;
        Node left;
        Node right;
        Node parent;
        /**
         * 首字符的位置
         */
        int x;
        int y;
        int treeHeight;
        String string;
        int width;

        private void init(String string) {
            string = (string == null) ? "null" : string;
            string = string.isEmpty() ? " " : string;

            width = string.length();
            this.string = string;
        }

        public Node(String string) {
            init(string);
        }

        public Node(Object btNode, BinaryTreeInfo opetaion) {
            init(opetaion.string(btNode).toString());

            this.btNode = btNode;
        }

        /**
         * 顶部方向字符的X（极其重要）
         *
         * @return
         */
        private int topLineX() {
            // 宽度的一半
            int delta = width;
            if (delta % 2 == 0) {
                delta--;
            }
            delta >>= 1;

            if (parent != null && this == parent.left) {
                return rightX() - 1 - delta;
            } else {
                return x + delta;
            }
        }

        /**
         * 右边界的位置（rightX 或者 右子节点topLineX的下一个位置）（极其重要）
         */
        private int rightBound() {
            if (right == null) return rightX();
            return right.topLineX() + 1;
        }

        /**
         * 左边界的位置（x 或者 左子节点topLineX）（极其重要）
         */
        private int leftBound() {
            if (left == null) return x;
            return left.topLineX();
        }

        /**
         * x ~ 左边界之间的长度（包括左边界字符）
         *
         * @return
         */
        private int leftBoundLength() {
            return x - leftBound();
        }

        /**
         * rightX ~ 右边界之间的长度（包括右边界字符）
         *
         * @return
         */
        private int rightBoundLength() {
            return rightBound() - rightX();
        }

        /**
         * 左边界可以清空的长度
         *
         * @return
         */
        private int leftBoundEmptyLength() {
            return leftBoundLength() - 1 - TOP_LINE_SPACE;
        }

        /**
         * 右边界可以清空的长度
         *
         * @return
         */
        private int rightBoundEmptyLength() {
            return rightBoundLength() - 1 - TOP_LINE_SPACE;
        }

        /**
         * 让left和right基于this对称
         */
        private void balance(Node left, Node right) {
            if (left == null || right == null)
                return;
            // 【left的尾字符】与【this的首字符】之间的间距
            int deltaLeft = x - left.rightX();
            // 【this的尾字符】与【this的首字符】之间的间距
            int deltaRight = right.x - rightX();

            int delta = Math.max(deltaLeft, deltaRight);
            int newRightX = rightX() + delta;
            right.translateX(newRightX - right.x);

            int newLeftX = x - delta - left.width;
            left.translateX(newLeftX - left.x);
        }

        private int treeHeight(Node node) {
            if (node == null) return 0;
            if (node.treeHeight != 0) return node.treeHeight;
            node.treeHeight = 1 + Math.max(
                    treeHeight(node.left), treeHeight(node.right));
            return node.treeHeight;
        }

        /**
         * 和右节点之间的最小层级距离
         */
        private int minLevelSpaceToRight(Node right) {
            int thisHeight = treeHeight(this);
            int rightHeight = treeHeight(right);
            int minSpace = Integer.MAX_VALUE;
            for (int i = 0; i < thisHeight && i < rightHeight; i++) {
                int space = right.levelInfo(i).leftX
                        - this.levelInfo(i).rightX;
                minSpace = Math.min(minSpace, space);
            }
            return minSpace;
        }

        private LevelInfo levelInfo(int level) {
            if (level < 0) return null;
            int levelY = y + level;
            if (level >= treeHeight(this)) return null;

            List<Node> list = new ArrayList<>();
            Queue<Node> queue = new LinkedList<>();
            queue.offer(this);

            // 层序遍历找出第level行的所有节点
            while (!queue.isEmpty()) {
                Node node = queue.poll();
                if (levelY == node.y) {
                    list.add(node);
                } else if (node.y > levelY) break;

                if (node.left != null) {
                    queue.offer(node.left);
                }
                if (node.right != null) {
                    queue.offer(node.right);
                }
            }

            Node left = list.get(0);
            Node right = list.get(list.size() - 1);
            return new LevelInfo(left, right);
        }

        /**
         * 尾字符的下一个位置
         */
        public int rightX() {
            return x + width;
        }

        public void translateX(int deltaX) {
            if (deltaX == 0) return;
            x += deltaX;

            // 如果是LineNode
            if (btNode == null) return;

            if (left != null) {
                left.translateX(deltaX);
            }
            if (right != null) {
                right.translateX(deltaX);
            }
        }
    }

    private static class LevelInfo {
        int leftX;
        int rightX;

        public LevelInfo(Node left, Node right) {
            this.leftX = left.leftBound();
            this.rightX = right.rightBound();
        }
    }
}
```

```java
package org.msdemt.demo.printer;

public abstract class Printer {
    /**
     * 二叉树的基本信息
     */
    protected BinaryTreeInfo tree;

    public Printer(BinaryTreeInfo tree) {
        this.tree = tree;
    }

    /**
     * 生成打印的字符串
     */
    public abstract String printString();

    /**
     * 打印后换行
     */
    public void println() {
        print();
        System.out.println();
    }

    /**
     * 打印
     */
    public void print() {
        System.out.print(printString());
    }
}
```

```java
package org.msdemt.demo.printer;

public class Strings {
    public static String repeat(String string, int count) {
        if (string == null) return null;

        StringBuilder builder = new StringBuilder();
        while (count-- > 0) {
            builder.append(string);
        }
        return builder.toString();
    }

    public static String blank(int length) {
        if (length < 0) return null;
        if (length == 0) return "";
        return String.format("%" + length + "s", "");
    }
}
```

```java
package org.msdemt.demo.file;

import java.io.BufferedWriter;
import java.io.File;
import java.io.FileWriter;

public class Files {

    public static void writeToFile(String filePath, Object data) {
        writeToFile(filePath, data, false);
    }

    public static void writeToFile(String filePath, Object data, boolean append) {
        if (filePath == null || data == null) return;

        try {
            File file = new File(filePath);
            if (!file.exists()) {
                file.getParentFile().mkdirs();
                file.createNewFile();
            }

            try (FileWriter writer = new FileWriter(file, append);
                 BufferedWriter out = new BufferedWriter(writer)) {
                out.write(data.toString());
                out.flush();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```


#### 网站推荐

- http://520it.com/binarytrees/
- http://btv.melezinek.cz/binary-search-tree.html
- https://www.cs.usfca.edu/~galles/visualization/Algorithms.html
- https://yangez.github.io/btree-js/
- https://www.codelike.in/

#### 根据元素内容获取节点

```java
/**
 * 根据元素找到node
 *
 * @param element
 * @return
 */
private Node<E> node(E element) {
    Node<E> node = root;
    while (node != null) {
        int cmp = compare(element, node.element);
        if (cmp == 0) {
            return node;
        } else if (cmp > 0) {
            node = node.right;
        } else {
            node = node.right;
        }
    }
    return null;
}
```

#### 删除节点

##### 删除叶子节点

![删除叶子节点](../../../../images/恋上算法与数据结构/1-07-二叉搜索树/20200918-删除叶子节点.png)  

直接删除
- node == node.parent.left
  - node.parent.left = null
- node == node.parent.right
  - node.parent.right = null
- node.parent = null
  - root = null

##### 删除度为1的节点

![删除度为1的节点](../../../../images/恋上算法与数据结构/1-07-二叉搜索树/20200918-删除度为1的节点.png)  

用子节点替代原节点的位置
- child 是 node.left 或者 child 是 node.right

- 用 child 替代 node 的位置
  - 如果 node 是左子节点
    - child.parent = node.parent
    - node.parent.left = child
  - 如果 node 是右子节点
    - child.parent = node.parent
    - node.parent.right = child
  - 如果 node 是根节点
    - root = child
    - child.parent = null

##### 删除度为2的节点

![删除度为2的节点](../../../../images/恋上算法与数据结构/1-07-二叉搜索树/20200918-删除度为2的节点.png)  

举例：先删除5，再删除4

先用前驱或者后继节点的值覆盖原节点的值

然后删除相应的前驱或者后继节点

如果一个节点的度为2，那么
- 它的前驱、后继节点的度只可能是0或1

##### 删除节点的实现

```java
public void remove(E element) {
    remove(node(element));
}

private void remove(Node<E> node) {
    if (node == null) {
        return;
    }

    size--;

    //度为2的节点
    if (node.hasTwoChildren()) {
        //找到后继节点
        Node<E> s = successor(node);
        //用后继节点的值覆盖度为2的节点的值，前驱或后继节点的度只能为0或1
        node.element = s.element;
        //删除后继节点，将后继节点s赋给node,在后续的流程中删除该度为0或1的节点
        node = s;
    }

    //删除node节点（此时node的度为1或者0）
    Node<E> replacement = node.left != null ? node.left : node.right;

    if (replacement != null) {
        //node是度为1的节点
        //更改parent
        replacement.parent = node.parent;
        //更改parent的left、right指向
        if (node.parent == null) {
            //node是度为1的节点并且是根节点
            root = replacement;
        } else if (node == node.parent.left) {
            node.parent.left = replacement;
        } else {
            //此时的情况为 node == node.parent.right
            node.parent.right = replacement;
        }
    } else if (node.parent == null) {
        //node是叶子节点并且是根节点
        root = null;
    } else {
        //node是叶子节点，但不是根节点
        if (node == node.parent.left) {
            node.parent.left = null;
        } else {
            //此时的情况为 node == node.parent.right
            node.parent.right = null;
        }
    }
}
```

### 二叉搜索树的重构

![二叉搜索树重构](../../../../images/恋上算法与数据结构/1-07-二叉搜索树/20200918-二叉搜索树重构.png)  

二叉树

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

    /**
     * 前序遍历
     *
     * @param visitor
     */
    public void preorder(Visitor<E> visitor) {
        if (visitor == null) {
            return;
        }
        preorder(root, visitor);
    }

    private void preorder(Node<E> node, Visitor<E> visitor) {
        if (node == null || visitor.stop) {
            return;
        }

        visitor.stop = visitor.visit(node.element);
        preorder(node.left, visitor);
        preorder(node.right, visitor);
    }

    /**
     * 中序遍历
     *
     * @param visitor
     */
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

    /**
     * 后续遍历
     *
     * @param visitor
     */
    public void postorder(Visitor<E> visitor) {
        if (visitor == null) {
            return;
        }
        postorder(root, visitor);
    }

    private void postorder(Node<E> node, Visitor<E> visitor) {
        if (node == null || visitor.stop) {
            return;
        }

        postorder(node.left, visitor);
        postorder(node.right, visitor);
        if (visitor.stop) {
            return;
        }
        visitor.stop = visitor.visit(node.element);
    }

    /**
     * 层序遍历
     *
     * @param visitor
     */
    public void levelOrder(Visitor<E> visitor) {
        if (root == null || visitor == null) {
            return;
        }

        Queue<Node<E>> queue = new LinkedList<>();
        queue.offer(root);

        while (!queue.isEmpty()) {
            Node<E> node = queue.poll();
            if (visitor.visit(node.element)) {
                return;
            }

            if (node.left != null) {
                queue.offer(node.left);
            }

            if (node.right != null) {
                queue.offer(node.right);
            }
        }
    }

    /**
     * 是否为完全二叉树
     *
     * @return
     */
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

    /**
     * 二叉树的高度
     *
     * @return
     */
    public int height() {
        if (root == null) {
            return 0;
        }

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

    /**
     * 二叉树的高度
     *
     * @return
     */
    public int height2() {
        return height(root);
    }

    private int height(Node<E> node) {
        if (node == null) {
            return 0;
        }
        return 1 + Math.max(height(node.left), height(node.right));
    }

    /**
     * 获取node的前驱节点
     *
     * @param node
     * @return
     */
    protected Node<E> predecessor(Node<E> node) {
        if (node == null) {
            return null;
        }

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

    /**
     * 获取node的后继节点
     *
     * @param node
     * @return
     */
    protected Node<E> successor(Node<E> node) {
        if (node == null) {
            return null;
        }

        // 后继节点在右子树当中（right.left.left.left....）
        Node<E> p = node.right;
        if (p != null) {
            while (p.left != null) {
                p = p.left;
            }
            return p;
        }

        // 从父节点、祖父节点中寻找后继节点
        while (node.parent != null && node == node.parent.right) {
            node = node.parent;
        }

        return node.parent;
    }

    /**
     * 自定义节点访问策略
     *
     * @param <E>
     */
    public static abstract class Visitor<E> {
        boolean stop;

        /**
         * @return 如果返回true，就代表停止遍历
         */
        abstract boolean visit(E element);
    }

    /**
     * node节点
     *
     * @param <E>
     */
    protected static class Node<E> {
        E element;
        Node<E> left;
        Node<E> right;
        Node<E> parent;

        public Node(E element, Node<E> parent) {
            this.element = element;
            this.parent = parent;
        }

        /**
         * 该节点是否为叶节点，即无子节点
         *
         * @return
         */
        public boolean isLeaf() {
            return left == null && right == null;
        }

        /**
         * 该节点是否有两个子节点
         *
         * @return
         */
        public boolean hasTwoChildren() {
            return left != null && right != null;
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
        Node<E> myNode = (Node<E>) node;
        String parentString = "null";
        if (myNode.parent != null) {
            parentString = myNode.parent.element.toString();
        }
        return myNode.element + "_p(" + parentString + ")";
    }
}
```

二叉搜索树

```java
package org.msdemt.demo.tree;

import java.util.Comparator;

/**
 * BST: 二叉搜索树
 *
 * @param <E>
 */
@SuppressWarnings("unchecked")
public class BST<E> extends BinaryTree<E> {

    private Comparator<E> comparator;

    /**
     * 支持不使用比较器构造二叉搜索树，此时需要node节点的元素实现Comparable接口，实现compareTo方法，
     * 否则在比较时会因为类型转换错误抛出异常
     */
    public BST() {
        this(null);
    }

    /**
     * 支持使用比较器构造二叉搜索树，即自定义大小比较规则
     *
     * @param comparator
     */
    public BST(Comparator<E> comparator) {
        this.comparator = comparator;
    }

    /**
     * 添加元素
     *
     * @param element
     */
    public void add(E element) {
        elementNotNullCheck(element);

        // 添加第一个节点
        if (root == null) {
            root = new Node<>(element, null);
            size++;
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
        Node<E> newNode = new Node<>(element, parent);
        if (cmp > 0) {
            parent.right = newNode;
        } else {
            parent.left = newNode;
        }
        size++;
    }

    /**
     * 删除元素
     *
     * @param element
     */
    public void remove(E element) {
        remove(node(element));
    }

    /**
     * 二叉搜索树中是否包含某个元素
     *
     * @param element
     * @return
     */
    public boolean contains(E element) {
        return node(element) != null;
    }

    private void remove(Node<E> node) {
        if (node == null) {
            return;
        }

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
        } else if (node.parent == null) { // node是叶子节点并且是根节点
            root = null;
        } else { // node是叶子节点，但不是根节点
            if (node == node.parent.left) {
                node.parent.left = null;
            } else { // node == node.parent.right
                node.parent.right = null;
            }
        }
    }

    /**
     * 根据元素查找对应的节点
     *
     * @param element
     * @return
     */
    private Node<E> node(E element) {
        Node<E> node = root;
        while (node != null) {
            int cmp = compare(element, node.element);
            if (cmp == 0) {
                return node;
            }
            if (cmp > 0) {
                node = node.right;
            } else { // cmp < 0
                node = node.left;
            }
        }
        return null;
    }

    /**
     * 节点元素比较大小
     *
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



