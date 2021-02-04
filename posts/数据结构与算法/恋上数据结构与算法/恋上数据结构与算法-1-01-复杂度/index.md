# 恋上数据结构与算法-1-01-复杂度


## 什么是算法

算法是用于解决特定问题的一系列的执行步骤

```java
/**
 * 计算 a 与 b 的和
 *
 * @param a
 * @param b
 * @return
 */
public static int plus(int a, int b) {
    return a + b;
}

/**
 * 计算 1+2+3+...+n 的和
 *
 * @param n
 * @return
 */
public static int sum(int n) {
    int result = 0;
    for (int i = 1; i <= n; i++) {
        result += i;
    }
    return result;
}
```

使用不同的算法，解决同一个问题，效率可能相差非常大。

比如：求第 n 个斐波那契数（fibonacci number）

## 如何评判一个算法的优劣？

```java
/**
 * 计算 1+2+3+...+n 的和
 *
 * @param n
 * @return
 */
public static int sum(int n) {
    int result = 0;
    for (int i = 1; i <= n; i++) {
        result += i;
    }
    return result;
}

/**
 * 计算 1+2+3+...+n 的和
 * @param n
 * @return
 */
public static int sum2(int n) {
    return (1 + n) * n / 2;
}
```

如果单从执行效率上进行评估，可能会想到一种方案，即比较不同算法对同一组输入的执行处理时间，这种方案也叫**事后统计法**。

**事后统计法**有比较明显的缺点，缺点如下
- 执行时间严重依赖硬件以及运行时各种不确定的环境因素
- 必须编写相应的测算代码
- 测试数据的选择比较难保证公正性


一般我们从以下维度来评估算法的优劣
- **正确性**、**可读性**、**健壮性**（对不合理输入的反应能力和处理能力）
- **时间复杂度**（time complexity）：估算程序指令的执行次数（执行时间）
- **空间复杂度**（space complexity）：估算所需占用的存储空间

## 大 O 表示法（Big O）

一般用**大 O 表示法**来描述复杂度，它表示的是数据规模 n 对应的复杂度。

使用**大 O 表示法**时，忽略常数、系数、低阶，如下
- 9 >> O(1)
- 2n+3 >> O(n)
- n<sup>2</sup>+2n+6 >> O(n<sup>2</sup>)
- 4n<sup>3</sup>+3n<sup>2</sup>+22n+100 >> O(n<sup>3</sup>)

注意：**大 O 表示法**仅仅是一种粗略的分析模型，是一种估算，能帮助我们短时间内了解一个算法的执行效率。

### 对数阶的细节

对数阶一般省略底数

log<sub>2</sub>n = log<sub>2</sub>9 * log<sub>9</sub>n

所以 log<sub>2</sub>n 、 log<sub>9</sub>n 统称为log n

### 常见的复杂度

执行次数 | 复杂度 | 非正式术语
-- | -- | -- 
12 | O(1) | 常数阶
2n+3 | O(n) | 线性阶
4n<sup>2</sup>+2n+6 | O(n<sup>2</sup>) | 平方阶
4log<sub>2</sub>n+25 | O(log n) | 对数阶
3n+2nlog<sub>3</sub>n+15 | O(nlog n) | nlog n 阶
4n<sup>3</sup>+3n<sup>2</sup>+22n+100 | O(n<sup>3</sup>) | 立方阶
2<sup>n</sup> | O(2<sup>n</sup>) | 指数阶

复杂度大小比较

O(1) < O(log n) < O(n) < O(nlog n) < O(n<sup>2</sup>) < O(n<sup>3</sup>) < O(2<sup>n</sup>) < O(n!) < O(n<sup>n</sup>)

可以借助函数生成工具对比复杂度的大小

https://zh.numberempire.com/graphingcalculator.php

数据规模较小时，复杂度对比如下

![20200909-数据规模较小时复杂度对比图](/images/恋上算法与数据结构/1-01-复杂度/20200909-数据规模较小时复杂度对比图.png)  

数据规模较大时，复杂度对比如下

![20200909-数据规模较大时复杂度对比图](/images/恋上算法与数据结构/1-01-复杂度/20200909-数据规模较大时复杂度对比图.png)  

### 多个数据规模的情况

```java
public static void test(int n, int k) {
    for (int i = 0; i < n; i++) {
        System.out.println("test");
    }
    for (int i = 0; i < k; i++) {
        System.out.println("test");
    }
}
```

test()函数的时间复杂度为 O(n+k)

## 斐波那契函数

斐波那契数：第一项为 0，第二项为 1，从第三项开始，每一项都等于前两项之和。如下所示

0, 1, 1, 2, 3, 5, 8, 13, ... ...

### 使用递归方式实现求第 n 个斐波那契数

```java
/**
 * 递归方式实现求第N个斐波那契数
 * <p>
 * 若n为64，则结果迟迟出不来，说明该算法效率很低
 * 时间复杂度为O(2^n)
 * @param n
 * @return
 */
public static int fib1(int n) {
    if (n <= 1) return n;
    return fib1(n - 1) + fib1(n - 2);
}
```

该方式的时间复杂度为 O(2<sup>n</sup>)

图解：

![使用递归计算给定整数的斐波那契数](/images/恋上算法与数据结构/1-01-复杂度/20200909-使用递归计算给定整数的斐波那契数.jpg)  

上图表示了 fib(5) 计算过程的递归树

算法：
- 检查整数 n ，如果 n 小于等于 1 ， 则返回 n 。
- 否则，通过递归关系： f(n) = f(n-1) + f(n-2) 调用自身。
- 直到所有计算返回结果得到答案。

复杂度分析
- 时间复杂度：O(2<sup>n</sup>)。这是计算斐波那契数最慢的方法。因为它需要指数的时间。
- 空间复杂度：O(n)，在堆栈中我们需要与 n 成正比的空间大小。该堆栈跟踪 fib(n) 的函数调用，随着堆栈的不断增长如果没有足够的内存则会导致 StackOverflowError。

### 使用循环方式实现求第 n 个斐波那契数


该方式的时间复杂度为 O(n)

```java
/**
 * for循环实现求第N个斐波那契数
 * 时间复杂度为O(n)
 * @param n
 * @return
 */
public static int fib2(int n) {
    if (n <= 1) return n;

    int first = 0;
    int second = 1;
    for (int i = 0; i < n - 1; i++) {
        int sum = first + second;
        first = second;
        second = sum;
    }
    return second;
}
```

```java
/**
 * 使用while循环实现求第N个斐波那契数
 * 时间复杂度为O(n)
 * 优化：循环体中节省一个局部变量（sum），使用while循环条件中节省一个局部变量
 * @param n
 * @return
 */
public static int fib3(int n) {
    if (n <= 1) return n;
    int first = 0;
    int second = 1;
    while (n-- > 1) {
        second += first;
        first = second - first;
    }
    return second;
}
```

### 使用线性代数方式实现求第 n 个斐波那契数

```java
/**
 * 使用线性代数实现求第N个斐波那契数
 * 时间复杂度为O(1)
 * @param n
 * @return
 */
public static int fib4(int n){
    double c = Math.sqrt(5);
    return (int) ((Math.pow((1+c)/2, n) - Math.pow((1-c)/2, n))/c);
}
```

复杂度视为 O(1) 。

### leetcode

leetcode 中斐波那契数练习题：

https://leetcode-cn.com/problems/fibonacci-number/

更多斐波那契数列解法参考：https://leetcode-cn.com/problems/fibonacci-number/solution/fei-bo-na-qi-shu-by-leetcode/

## 算法的优化方向

- 用尽量少的存储空间
- 用尽量少的执行步骤（执行时间）

根据情况，可以
- 空间换时间
- 时间换空间
