# Jsp使用var出现警告


> 参考：  
> https://www.zhihu.com/question/34294629  
> https://www.sunzhongwei.com/var-let-const-difference-in-es6


## 介绍

idea 编辑 jsp 文件时，在 jsp 内使用 var 定义变量时出现警告，信息如下

```
'var' used instead of 'let' or 'const'
```

idea 推荐使用 let 或 const 替代 var 来定义变量。

因为 es6 推荐不要使用 var 定义变量，使用 let 或 const 定义变量

那么 var 与 let 、 const 有什么区别呢？

## const

测试下 const

```js
> const framework = "weex";
> framework = "react native";
TypeError: Assignment to constant variable.
> const weex = {"weexpack": "good", "weex-toolkit": "sucks"};
> weex["weex-toolkit"] = "good";
'good'
> weex = "react native";
TypeError: Assignment to constant variable.
```

- const 定义的变量只能被赋值一次
- const 定义的变量的属性可以被多次赋值

const 的使用场景：定义一个常量

示例：

```js
const store = new Vuex.Store({
  actions,
  mutations,

  state: {
    activeType: null,
    items: {},
    users: {},
    counts: {
      top: 20,
      new: 20,
    },
    lists: {
      top: [],
      new: [],
    }
  }

})

export default store
```

将 store 定义为 const 的好处：明确声明 store 在其他地方不能被再次赋值，但是其属性可以被修改。

## let

The let statement declares a block scope local variable, optionally initializing it to a value. let allows you to declare variables that are limited in scope to the block, statement, or expression on which it is used. This is unlike the var keyword, which defines a variable globally, or locally to an entire function regardless of block scope.

let 与 var 最大的不同是： let 是用来定义块级别的变量，而 var 定义的变量在全局内可以使用。

let 的优点是避免因疏忽重复定义变量或无意间修改全局变量

```js
> let year = 2017;
> year = 2018;
2018
> let year = 2025;
TypeError: Identifier 'year' has already been declared
```

var 与 let 的区别示例

```js
function varTest() {
  var x = 1;
  if (true) {
    var x = 2;  // same variable!
    console.log(x);  // 2
  }
  console.log(x);  // 2
}

function letTest() {
  let x = 1;
  if (true) {
    let x = 2;  // different variable
    console.log(x);  // 2
  }
  console.log(x);  // 1
}
```

## var

var 的问题是过于灵活

```js
> var day = 1;
> var day = 2;
> day
2
```

变量被重复定义了

在 es6 中，我们应该尽量避免使用 var
