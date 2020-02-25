# Vue.js 目录

vue根据功能将代码模块化，各功能和复用代码独立成每个目录。vue2.0中src代码目录如下：

```
src
├── compiler        // 编译相关，ast解析优化，render函数代码生成
├── core            // 核心代码，内置组件，API，Vue实例化，observer，vnode 
├── platforms       // 不同平台的支持，web，weex
├── server          // 服务端渲染，nodejs端
├── sfc             // .vue 文件解析为js对象
├── shared          // 共享代码，工具方法
```
# Vue.js 构建
Vue.js 源码是基于[rollup](https://github.com/rollup/rollup) 构建的。Rollup 是一个 [JavaScript 模块打包器](https://www.rollupjs.com/) ，可以将小块代码编译成大块复杂的代码。Rollup偏向应用于js库，webpack偏向应用于前端工程，UI库；如果你的应用场景中只是js代码，希望做ES转换，模块解析，可以使用Rollup。如果你的场景中涉及到css、html，涉及到复杂的代码拆分合并，建议使用webpack。

```js
//package.json

{
  "name": "vue",
  "version": "2.5.17-beta.0",
  "description": "Reactive, component-oriented view layer for modern web interfaces.",
  "main": "dist/vue.runtime.common.js",

  ...

  "scripts": {

    ...

    "build": "node scripts/build.js",
    "build:ssr": "npm run build -- web-runtime-cjs,web-server-renderer",
    "build:weex": "npm run build -- weex"
    }
}
```

script中有三条build命令，后两条是在第一条基础上添加环境参数，可以看出构建过程就是通过运行scripts/build.js执行的：

```js
//scripts/build.js

  ...

let builds = require('./config').getAllBuilds()

// filter builds via command line arg
if (process.argv[2]) {
  const filters = process.argv[2].split(',')
  builds = builds.filter(b => {
    return filters.some(f => b.output.file.indexOf(f) > -1 || b._name.indexOf(f) > -1)
  })
} else {
  // filter out weex builds by default
  builds = builds.filter(b => {
    return b.output.file.indexOf('weex') === -1
  })
}

build(builds)

```

build.js中先取出配置根据命令行环境参数过滤，构建出不同需求的vue.js。配置读取在scripts/config.js中：

```js
//scripts/config.js

const builds = {
  // Runtime only (CommonJS). Used by bundlers e.g. Webpack & Browserify
  'web-runtime-cjs': {
    entry: resolve('web/entry-runtime.js'),
    dest: resolve('dist/vue.runtime.common.js'),
    format: 'cjs',
    banner
  },

  ...

  // Runtime only (ES Modules). Used by bundlers that support ES Modules,
  // e.g. Rollup & Webpack 2
  'web-runtime-esm': {
    entry: resolve('web/entry-runtime.js'),
    dest: resolve('dist/vue.runtime.esm.js'),
    format: 'es',
    banner
  },

  ...

  // Web compiler (CommonJS).
  'web-compiler': {
    entry: resolve('web/entry-compiler.js'),
    dest: resolve('packages/vue-template-compiler/build.js'),
    format: 'cjs',
    external: Object.keys(require('../packages/vue-template-compiler/package.json').dependencies)
  },

  ...

  // Web server renderer (CommonJS).
  'web-server-renderer': {
    entry: resolve('web/entry-server-renderer.js'),
    dest: resolve('packages/vue-server-renderer/build.js'),
    format: 'cjs',
    external: Object.keys(require('../packages/vue-server-renderer/package.json').dependencies)
  },

  ...

  // Weex compiler (CommonJS). Used by Weex's Webpack loader.
  'weex-compiler': {
    weex: true,
    entry: resolve('weex/entry-compiler.js'),
    dest: resolve('packages/weex-template-compiler/build.js'),
    format: 'cjs',
    external: Object.keys(require('../packages/weex-template-compiler/package.json').dependencies)
  }
}
```

初始化vue.js项目时常常需要选择版本。其中，Runtime Only只包含运⾏时的 Vue.js 代码所以轻量，通常需要借助如 webpack 的 vue-loader ⼯具把 .vue ⽂件编译成 JavaScript。Runtime+Compiler在运行时编译模板生成render function，性能相对较差。

根据配置需求不同，有runtime与compiler，ssr，和不同js包规范（umd，cjs，esm）配置选择，有许多组合选项，不一一列举。但每个配置都遵循rollup规则，entry为resolve后的在src文件夹下不同平台构建入口文件的路径，dest为dist文件夹下输出文件路径，format为构建包格式。

```js
//scripts/config.js

function genConfig (name) {
  const opts = builds[name]
  //config Object for rollup
  const config = {
    input: opts.entry,
    external: opts.external,
    plugins: [
      replace({
        __WEEX__: !!opts.weex,
        __WEEX_VERSION__: weexVersion,
        __VERSION__: version
      }),
      flow(),
      buble(),
      alias(Object.assign({}, aliases, opts.alias))
    ].concat(opts.plugins || []),
    output: {
      file: opts.dest,
      format: opts.format,
      banner: opts.banner,
      name: opts.moduleName || 'Vue'
    }

  }

  if (opts.env) {
    config.plugins.push(replace({
      'process.env.NODE_ENV': JSON.stringify(opts.env)
    }))
  }

  Object.defineProperty(config, '_name', {
    enumerable: false,
    value: name
  })

  return config
}

if (process.env.TARGET) {
  module.exports = genConfig(process.env.TARGET)
} else {
  exports.getBuild = genConfig
  exports.getAllBuilds = () => Object.keys(builds).map(genConfig)
}
```

config.js导出的getAllBuilds方法中遍历builds对象执行genConfig，返回重新生成的rollup打包时所用的config对象的映射数组。build.js中拿到这个数组过滤后进行build：

```js
//scripts/build.js

function build (builds) {
  let built = 0
  const total = builds.length
  const next = () => {
    buildEntry(builds[built]).then(() => {
      built++
      if (built < total) {
        next()
      }
    }).catch(logError)
  }

  next()
}

function buildEntry (config) {
  const output = config.output
  const { file, banner } = output
  const isProd = /min\.js$/.test(file)
  return rollup.rollup(config)
    .then(bundle => bundle.generate(output))
    .then(({ code }) => {
      if (isProd) {
        var minified = (banner ? banner + '\n' : '') + uglify.minify(code, {
          output: {
            ascii_only: true
          },
          compress: {
            pure_funcs: ['makeMap']
          }
        }).code
        return write(file, minified, true)
      } else {
        return write(file, code)
      }
    })
}
```

build过程中遍历rollup打包所用的config数组执行buildEntry异步操作，返回的promise不断next直到遍历完成。buildEntry中调用rollup处理后将生成code写入output路径文件中。注意到promise.then细节处，在判断到输出文件以.min.js结尾时默认为生产环境输出，对code调用了uglify.minify进行压缩。

# Vue.js 入口
本次分析的源码是Runtime + Compiler版本，⼊⼝是src/platforms/web/entry-runtime-with-compiler.js。

```js
// src/platforms/web/entry-runtime-with-compiler.js

    ...

import Vue from './runtime/index'
import { compileToFunctions } from './compiler/index'

    ...

const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {

    ...

}

    ...

Vue.compile = compileToFunctions
export default Vue
```

入口文件最后一行export的Vue在'./runtime/index'导出的Vue对象上改写了原型上的$mount方法，并挂载了'./compiler/index'中导出的compileToFunctions方法。这种在Vue原型上根据需求，动态重写覆盖方法体现出模块化的好处，高度复用性。向上查看'./runtime/index.js'中定义的Vue：
```js
//src/platforms/web/runtime/index.js

import Vue from 'core/index'

    ...

// install platform specific utils
Vue.config.mustUseProp = mustUseProp
Vue.config.isReservedTag = isReservedTag
Vue.config.isReservedAttr = isReservedAttr
Vue.config.getTagNamespace = getTagNamespace
Vue.config.isUnknownElement = isUnknownElement

// install platform runtime directives & components
extend(Vue.options.directives, platformDirectives)
extend(Vue.options.components, platformComponents)

// install platform patch function
Vue.prototype.__patch__ = inBrowser ? patch : noop

// public mount method
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}

```
runtime/index.js中对'core/index'中获取到的Vue挂载了静态全局配置，__patch__、$mount等方法。向上查看'core/index.js'中定义的Vue：

```js
//src/core/index.js

import Vue from './instance/index'
import { initGlobalAPI } from './global-api/index'

    ...

initGlobalAPI(Vue)

    ...

Vue.version = '__VERSION__'

export default Vue
```
core/index.js中对'./instance/index'中获取到的Vue进行全局API初始化，挂载SSR相关属性和version属性等。向上查看'./instance/index.js'中定义的Vue，和'./global-api/index'
中的全局API定义：

```js
//src/instance/index.js

import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'

function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)

export default Vue

```
这里终于到了定义Vue的函数，注意到这里限制除new方法的实例化调用以外会报警告。下面的Mixin方法通过在Vue.prototype上挂载不同的方法，这样拆分混入利于代码的模块化组织。

```js
//src/global-api/index.js

import config from '../config'
import { initUse } from './use'
import { initMixin } from './mixin'
import { initExtend } from './extend'
import { initAssetRegisters } from './assets'
import { set, del } from '../observer/index'
import { ASSET_TYPES } from 'shared/constants'
import builtInComponents from '../components/index'

import {
  warn,
  extend,
  nextTick,
  mergeOptions,
  defineReactive
} from '../util/index'

export function initGlobalAPI (Vue: GlobalAPI) {
  // config
  const configDef = {}
  configDef.get = () => config
  if (process.env.NODE_ENV !== 'production') {
    configDef.set = () => {
      warn(
        'Do not replace the Vue.config object, set individual fields instead.'
      )
    }
  }
  Object.defineProperty(Vue, 'config', configDef)

  // exposed util methods.
  // NOTE: these are not considered part of the public API - avoid relying on
  // them unless you are aware of the risk.
  Vue.util = {
    warn,
    extend,
    mergeOptions,
    defineReactive
  }

  Vue.set = set
  Vue.delete = del
  Vue.nextTick = nextTick

  Vue.options = Object.create(null)
  ASSET_TYPES.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
  })

  // this is used to identify the "base" constructor to extend all plain-object
  // components with in Weex's multi-instance scenarios.
  Vue.options._base = Vue

  extend(Vue.options.components, builtInComponents)

  initUse(Vue)
  initMixin(Vue)
  initExtend(Vue)
  initAssetRegisters(Vue)
}

```
这里主要是在Vue上挂载了全局的静态方法，区别于原型 prototype 上扩展⽅法。此外扩展了'../components/index'的内置组件KeepAlive到Vue.options.components。
# 总结
本篇主要讲述Vue构建过程和Runtime+Compiler版本Vue初始化大致过程：不同作⽤和功能的需求使Vue有多个构建版本对应的⼊⼝，利用rollup工具最终编译打包⽣成的JS⽂件；Vue的初始化过程也就是用function实现Vue class，并在原型和本身上挂载大量方法和属性，至此在调用new Vue()实例化对象后能够拿到这些全局/原型方法和属性，方便下一步的使用和扩展。


参考来源：https://www.jianshu.com/p/60070a6d7631