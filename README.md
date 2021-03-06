# Daily-vueJs-sourceCode-learn

Vue是一套用于构建用户界面的渐进式框架。与其它大型框架不同的是，Vue 被设计为可以自底向上逐层应用。Vue 的核心库只关注视图层，不仅易于上手，还便于与第三方库或既有项目整合。另一方面，当与现代化的工具链以及各种支持类库结合使用时，Vue 也完全能够为复杂的单页应用提供驱动。

## 介绍

本仓库创建于2020年2月，笔者能用Vue(+Vuex+router)+webpack/Vue-cli开发中小型web移动端应用之时，抱着好奇学习和get源码阅读技能目的开始。
本次选择的Vue源码版本号为"2.5.17-beta.0"，尽管开发过程中为了性能我们会⽤ Runtime only版本开发⽐较多，但考虑到编译相关源码阅读的价值，本学习文档分析的源码是 Runtime+Compiler 的 Vue.js。
PS：Vue3.0将于今年公布真正可用版本，届时将会有大量使用变动和更快更小的性能改变，期待3.0的源码解读。尤大🐮🍺！


## 目录

### Vue

[初识flow](./docs/flow.md)

[Vue.js构建及结构](./docs/build.md)

[Vue.js初始化过程](./docs/initialization.md)

### Vue-router

...

### Vuex

...


最后献上尤大在知乎回复时让我印象深刻的一段话：
“Vue 从一开始的定位就是尽可能的降低前端开发的门槛，让更多的人能够更快地上手开发。我以前也说过，开发vue的初衷不是为了搞个大新闻，只是做了个我自己用得舒服的框架。我虽然也在Google这样的大公司呆过，但骨子里是一个喜欢自由的人，也一直觉得独立开发者很酷（这也是为什么最终自己也成了一个独立开发者）。很多时候我更希望自己做的东西能帮到那些中小型企业和个人开发者。举个例子来说，美国传统行业里有很多smallbusiness它们不像大公司那样有专门的 IT团队来信息化整个流程，很多只能雇一个普通的 contractor程序员，有些甚至是老板自己兼职研究代码。我收到过好几封这样的感谢信，说因为Vue让它们多快好省地做了个内部应用，解决了实际问题，这样的故事是让我觉得特别爽的。”

参考优秀源码解读地址:
https://github.com/ustbhuangyi/vue-analysis
https://github.com/answershuto/learnVue
