# 认识 Flow

[Flow](https://flow.org/en/docs/getting-started/) 是 facebook 出品的 JavaScript 静态类型检查⼯具。在vue2.0的项目中加入flow类型检查。当前项目是用js写的，当项目越来越大，由于js弱类型的特性，相比ts(typescript)这种强类型的语言而言，后期维护会越来越困难。为了解决这个问题，决定使用flow加入类型检查。

## Flow 的⼯作⽅式
类型推断：通过变量的使⽤上下⽂来推断出变量类型，然后根据这些推断来检查类型。
因为Flow非常了解JavaScript，所以不需要其中许多类型。您只需要做很少的工作就可以向Flow描述您的代码，它将推断其余的代码。在很多时候，Flow完全不需要任何类型就可以理解您的代码。

```js
/*@flow*/

// @flow
function square(n) {
  return n * n; // Error!
}

square("2");
```

运行上面一段代码，控制台将输出：

```
Cannot perform arithmetic operation because string [1] is not a number.

   index.js:3:14
   3|   return n * n; // Error!
                   ^

References:
   index.js:6:8
   6| square("2");
             ^^^ [1]
```

类型注释：Flow 会基于事先写好注释来判断类型。
Flow通过静态类型注释检查代码是否存在错误。这些[类型](https://flow.org/en/docs/types/)使您可以告诉Flow您希望代码如何工作，并且Flow将确保它以这种方式工作。

```js
// @flow

function foo(x: ?number): string {
  if (x) {
    return x;
  }
  return "default string";
}
```

运行上面一段代码，控制台将输出：

```
Cannot return `x` because number [1] is incompatible with string [2].

   index.js:6:12
   6|     return x;
                 ^

References:
   index.js:4:18
   4| function foo(x: ?number): string {
                       ^^^^^^ [1]
   index.js:4:27
   4| function foo(x: ?number): string {
                                ^^^^^^ [2]
```

## Flow 在第三方库中的应用
Flow不认识第三方库的类型定义，使用如vue中的vnode、component等类型在检查的时候会报错。为了解决这类问题，Flow提出了⼀个libdef的概念，可以⽤来识别这些第三⽅库或者是⾃定义类型。
flow init命令创建的.flowconfig告诉Flow 在类型检查代码时包括指定的库定义，可以指定多个库。默认情况下，flow-typed项目根目录中的文件夹作为库目录包含在内。
vue源码中[libs]配置在flow⽂件夹内。

```
flow
├── compiler.js        # 编译相关
├── component.js       # 组件数据结构
├── global-api.js      # Global API 结构
├── modules.js         # 第三方库定义
├── options.js         # 选项相关
├── ssr.js             # 服务端渲染相关
├── vnode.js           # 虚拟 node 相关
```

---

参考：https://www.jianshu.com/p/d4fbe53a1e39
     https://github.com/ustbhuangyi/vue-analysis
     https://flow.org/en/docs/getting-started/
