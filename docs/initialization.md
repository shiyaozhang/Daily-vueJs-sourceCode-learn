# 数据驱动
数据驱动，是指视图是由数据驱动⽣成的，我们对视图的修改，不会直接操作DOM，⽽是通过修改数据。它相⽐使⽤ jQuery 等库直接修改DOM，⼤⼤简化了代码量。特别是当交互复杂的时候，只关⼼数据的修改会让代码的逻辑变的⾮常清晰，因为DOM变成了数据的映射，我们所有的逻辑都是对数据的修改，⽽不⽤碰触 DOM，这样的代码⾮常利于维护。
Vue.js 是一个提供了 MVVM 风格的双向数据绑定的 Javascript 库，专注于View 层。它让开发者省去了操作DOM的过程，只需要改变数据。Vue会通过Dircetives指令，对DOM做一层封装，当数据发生改变会通知指令去修改对应的DOM，数据驱动DOM变化，DOM是数据的一种自然映射。
Vue还会对操作进行监听，当视图发生改变时，vue监听到这些变化，从而改变数据，这样就形成了数据的双向绑定。

## 实例化
  通过function实现Vue类定义，在src/core/instance/index.js中：
```js
//src/core/instance/index.js

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
`this._init`方法通过initMixin(Vue)混入，在src/core/instance/init.js中定义：
```js
//src/core/instance/init.js

export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    // a uid
    vm._uid = uid++

    let startTag, endTag
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      startTag = `vue-perf-start:${vm._uid}`
      endTag = `vue-perf-end:${vm._uid}`
      mark(startTag)
    }

    // a flag to avoid this being observed
    vm._isVue = true
    // merge options
    if (options && options._isComponent) {
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      initProxy(vm)
    } else {
      vm._renderProxy = vm
    }
    // expose real self
    vm._self = vm
    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)
    callHook(vm, 'beforeCreate')
    initInjections(vm) // resolve injections before data/props
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')

    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      vm._name = formatComponentName(vm, false)
      mark(endTag)
      measure(`vue ${vm._name} init`, startTag, endTag)
    }

    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}
```
初始化函数继续沿用模块化的编程思想，把不同功能分成单独的函数实现，完成后混合入主线，让主线逻辑思路清晰明了，是十分值得借鉴的思维。
`_init`方法主要做的事就是挂载uid，合并option配置到$option上，初始化⽣命周期，初始化事件，初始化渲染，初始化 data、props、computed、watcher，和调用creat相关的生命周期函数。初始化过程中步骤较多，先重点关注下data的初始化过程。

```js
//src/core/instance/state.js

export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}

function initData (vm: Component) {
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    )
  }
  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) {
      proxy(vm, `_data`, key)
    }
  }
  // observe data
  observe(data, true /* asRootData */)
}

export function proxy (target: Object, sourceKey: string, key: string) {
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}

```
initState中调用initData方法获取data对象，遍历key检查是否与props、mothod上的key重复，因为最终都会挂载在vm实例上供用户this[key]的方式获取,这一步通过proxy函数在vm上封装一层代理实现。最后调用observe函数实现响应式处理，这点将在下节分析。

回到src/core/instance/init.js的`_init`方法最后，是根据$options.el做组件挂载，挂载vm实例的过程是执行$mount方法，$mount 这个⽅法的实现和平台、构建⽅式都相关，此处来看src/platform/web/entry-runtimewith-compiler.js文件中$mount的实现：
```js
//src/platform/web/entry-runtimewith-compiler.js

const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && query(el)

  /* istanbul ignore if */
  if (el === document.body || el === document.documentElement) {
    process.env.NODE_ENV !== 'production' && warn(
      `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
    )
    return this
  }

  const options = this.$options
  // resolve template/el and convert to render function
  if (!options.render) {
    let template = options.template
    if (template) {
      if (typeof template === 'string') {
        if (template.charAt(0) === '#') {
          template = idToTemplate(template)
          /* istanbul ignore if */
          if (process.env.NODE_ENV !== 'production' && !template) {
            warn(
              `Template element not found or is empty: ${options.template}`,
              this
            )
          }
        }
      } else if (template.nodeType) {
        template = template.innerHTML
      } else {
        if (process.env.NODE_ENV !== 'production') {
          warn('invalid template option:' + template, this)
        }
        return this
      }
    } else if (el) {
      template = getOuterHTML(el)
    }
    if (template) {
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile')
      }

      const { render, staticRenderFns } = compileToFunctions(template, {
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this)
      options.render = render
      options.staticRenderFns = staticRenderFns

      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile end')
        measure(`vue ${this._name} compile`, 'compile', 'compile end')
      }
    }
  }
  return mount.call(this, el, hydrating)
}
```
值得注意的是，这里先将原型上的$mount方法缓存，再重新定义中最后调用了这个缓存的mount,这样设计是为了更好地按需复用runtime的代码。在新的$mount方法中限制挂载DOM不能是根节点，并将在检测到options.render不存在时，调用compileToFunctions函数在线编译render方法。编译相关实现将在以后做分析，这里向上查看src/platform/web/runtime/index.js中定义的原型$mount方法。
```js
//src/platform/web/runtime/index.js

Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}

```
$mount在内部获取挂载DOM之后调用mountComponent方法，并传入DOM元素和hydrating参数(与服务端渲染相关，此处不需要)。向上查看src/core/instance/lifecycle.js中mountComponent的定义：
```js
//src/core/instance/lifecycle.js

export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el
  if (!vm.$options.render) {
    vm.$options.render = createEmptyVNode
    if (process.env.NODE_ENV !== 'production') {
      /* istanbul ignore if */
      if ((vm.$options.template && vm.$options.template.charAt(0) !== '#') ||
        vm.$options.el || el) {
        warn(
          'You are using the runtime-only build of Vue where the template ' +
          'compiler is not available. Either pre-compile the templates into ' +
          'render functions, or use the compiler-included build.',
          vm
        )
      } else {
        warn(
          'Failed to mount component: template or render function not defined.',
          vm
        )
      }
    }
  }
  callHook(vm, 'beforeMount')

  let updateComponent
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    //mark是开发环境下的性能埋点的输出，这里不作篇幅讲述
    updateComponent = () => {
      const name = vm._name
      const id = vm._uid
      const startTag = `vue-perf-start:${id}`
      const endTag = `vue-perf-end:${id}`

      mark(startTag)
      const vnode = vm._render()
      mark(endTag)
      measure(`vue ${name} render`, startTag, endTag)

      mark(startTag)
      vm._update(vnode, hydrating)
      mark(endTag)
      measure(`vue ${name} patch`, startTag, endTag)
    }
  } else {
    updateComponent = () => {
      vm._update(vm._render(), hydrating)
    }
  }

  // we set this to vm._watcher inside the watcher's constructor
  // since the watcher's initial patch may call $forceUpdate (e.g. inside child
  // component's mounted hook), which relies on vm._watcher being already defined
  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  hydrating = false

  // manually mounted instance, call mounted on self
  // mounted is called for render-created child components in its inserted hook
  if (vm.$vnode == null) {
    //vm.$vnode表⽰Vue实例的⽗虚拟Node，为Null则表⽰当前是根Vue的实例
    vm._isMounted = true
    callHook(vm, 'mounted')
  }
  return vm
}
```
mountComponent函数中首先判断render方法的有无，根据template报出不同错误情况的警告。这里重点是定义出updateComponent函数，内部调用`vm._update`方法，传入`vm._render`⽅法⽣成的虚拟Node来更新试图。接着用updateComponent作为参数实例化一个渲染watcher，用来在初始化和订阅到数据更新时执行这个回调函数，watcher相关内容将在之后的篇幅分析。其余部分就是标记`vm._isMounted`已挂载，和调用生命周期钩子函数。
至此可以看出，`vm._render`和`vm._update`是更新视图的关键。

## render函数
`_render`方法是在在src/core/instance/index.js中调用renderMixin(Vue)时挂载在Vue原型上。renderMixin定义在src/core/instance/render.js中：
```js
//src/core/instance/render.js

export function renderMixin (Vue: Class<Component>) {
  // install runtime convenience helpers
  installRenderHelpers(Vue.prototype)

  Vue.prototype.$nextTick = function (fn: Function) {
    return nextTick(fn, this)
  }

  Vue.prototype._render = function (): VNode {
    const vm: Component = this
    const { render, _parentVnode } = vm.$options

    // reset _rendered flag on slots for duplicate slot check
    if (process.env.NODE_ENV !== 'production') {
      for (const key in vm.$slots) {
        // $flow-disable-line
        vm.$slots[key]._rendered = false
      }
    }

    if (_parentVnode) {
      vm.$scopedSlots = _parentVnode.data.scopedSlots || emptyObject
    }

    // set parent vnode. this allows render functions to have access
    // to the data on the placeholder node.
    vm.$vnode = _parentVnode
    // render self
    let vnode
    try {
      vnode = render.call(vm._renderProxy, vm.$createElement)
    } catch (e) {
      handleError(e, vm, `render`)
      // return error render result,
      // or previous vnode to prevent render error causing blank component
      /* istanbul ignore else */
      if (process.env.NODE_ENV !== 'production') {
        if (vm.$options.renderError) {
          try {
            vnode = vm.$options.renderError.call(vm._renderProxy, vm.$createElement, e)
          } catch (e) {
            handleError(e, vm, `renderError`)
            vnode = vm._vnode
          }
        } else {
          vnode = vm._vnode
        }
      } else {
        vnode = vm._vnode
      }
    }
    // return empty vnode in case the render function errored out
    if (!(vnode instanceof VNode)) {
      if (process.env.NODE_ENV !== 'production' && Array.isArray(vnode)) {
        warn(
          'Multiple root nodes returned from render function. Render function ' +
          'should return a single root node.',
          vm
        )
      }
      vnode = createEmptyVNode()
    }
    // set parent
    vnode.parent = _parentVnode
    return vnode
  }
}
```
这里首先获取到vm.$options上的render方法(手写或编译生成的)，重点是`vnode = render.call(vm._renderProxy, vm.$createElement)`调用render函数生成虚拟DOM,最后对捕捉错误并进行降级处理。其中`vm._renderProxy`是render函数上下文，定义在src/core/instance/init.js中：
```js
//src/core/instance/init.js

    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      initProxy(vm)
    } else {
      vm._renderProxy = vm
    }
```
生产环境下`vm._renderProxy`就是vm本身，开发环境下则是对vm做了一层添加了边界情况的警告的代理。

接下来自然可以看出，render.call调用的第二个参数vm.$createElement就是手写render函数中的createElement(AKA h)方法，其定义在src/core/instance/render.js的initRender中：
```js
//src/core/instance/render.js

export function initRender (vm: Component) {
  vm._vnode = null // the root of the child tree
  vm._staticTrees = null // v-once cached trees
  const options = vm.$options

  ...

  // bind the createElement fn to this instance
  // so that we get proper render context inside it.
  // args order: tag, data, children, normalizationType, alwaysNormalize
  // internal version is used by render functions compiled from templates
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
  // normalization is always applied for the public version, used in
  // user-written render functions.
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)

  ...

}
```
`vm.$createElement`与`vm._c`内部实现都是调用createElement方法，只有传入最后一个参数不同用来区别调用时是用户手写还是编译生成的。在进一步分析createElement方法前，需要先对Virtual DOM有一些认知。

## Virtual Dom
Virtual DOM是相对与DOM(文档对象模型)来说的，MDN上关于DOM的定义：“DOM模型用一个逻辑树来表示一个文档，树的每个分支的终点都是一个节点(node)，每个节点都包含着对象(objects)。DOM的方法(methods)让你可以用特定方式操作这个树，用这些方法你可以改变文档的结构、样式或者内容”。真正的DOM元素是非常庞大的，因为浏览器把DOM设计地非常复杂，所以当我们频繁地去更新DOM时，会产生一定的性能问题。

Virtual DOM 其实就是一棵以 JavaScript 对象( VNode 节点)作为基础的树，用对象属性来描述节点，实际上它只是一层对真实DOM的抽象。最终可以通过一系列操作使这棵树映射到真实环境上。简单来说，可以把Virtual DOM理解为一个简单的JS对象，并且最少包含标签名(tag)、属性(attrs)和子元素对象(children)三个属性。相对于频繁地去操作DOM引起的性能问题，Vritual DOM很好地将DOM做了一层映射关系，将原来需要在DOM上的一系列操作，映射到来操作Virtual DOM。

在 Vue.js 中，Virtual DOM是一个名为Vnode的class，⽤定义在
src/core/vdom/vnode.js中：
```js
//src/core/vdom/vnode.js

export default class VNode {
  tag: string | void;
  data: VNodeData | void;
  children: ?Array<VNode>;
  text: string | void;
  elm: Node | void;
  ns: string | void;
  context: Component | void; // rendered in this component's scope
  key: string | number | void;
  componentOptions: VNodeComponentOptions | void;
  componentInstance: Component | void; // component instance
  parent: VNode | void; // component placeholder node

  // strictly internal
  raw: boolean; // contains raw HTML? (server only)
  isStatic: boolean; // hoisted static node
  isRootInsert: boolean; // necessary for enter transition check
  isComment: boolean; // empty comment placeholder?
  isCloned: boolean; // is a cloned node?
  isOnce: boolean; // is a v-once node?
  asyncFactory: Function | void; // async component factory function
  asyncMeta: Object | void;
  isAsyncPlaceholder: boolean;
  ssrContext: Object | void;
  fnContext: Component | void; // real context vm for functional nodes
  fnOptions: ?ComponentOptions; // for SSR caching
  fnScopeId: ?string; // functional scope id support

  constructor (
    tag?: string,
    data?: VNodeData,
    children?: ?Array<VNode>,
    text?: string,
    elm?: Node,
    context?: Component,
    componentOptions?: VNodeComponentOptions,
    asyncFactory?: Function
  ) {
    this.tag = tag
    this.data = data
    this.children = children
    this.text = text
    this.elm = elm
    this.ns = undefined
    this.context = context
    this.fnContext = undefined
    this.fnOptions = undefined
    this.fnScopeId = undefined
    this.key = data && data.key
    this.componentOptions = componentOptions
    this.componentInstance = undefined
    this.parent = undefined
    this.raw = false
    this.isStatic = false
    this.isRootInsert = true
    this.isComment = false
    this.isCloned = false
    this.isOnce = false
    this.asyncFactory = asyncFactory
    this.asyncMeta = undefined
    this.isAsyncPlaceholder = false
  }

  // DEPRECATED: alias for componentInstance for backwards compat.
  /* istanbul ignore next */
  get child (): Component | void {
    return this.componentInstance
  }
}
```
VNode class在定义时为了描述,定义了与DOM相关的关键属性，如标签名、数据、⼦
节点、键值等，其它属性都是都是⽤来扩展 VNode 的灵活性以及实现⼀些特殊 feature 的。在 Vue.js 中，Vnode是在render过程中调用createElement方法生成的。

## 关于createElement

前小结讲到的createElement定义在src/core/vdom/create-elemenet.js中：
```js
//src/core/vdom/create-elemenet.js

export function createElement (
  context: Component,
  tag: any,
  data: any,
  children: any,
  normalizationType: any,
  alwaysNormalize: boolean
): VNode | Array<VNode> {
  if (Array.isArray(data) || isPrimitive(data)) {
    normalizationType = children
    children = data
    data = undefined
  }
  if (isTrue(alwaysNormalize)) {
    normalizationType = ALWAYS_NORMALIZE
  }
  return _createElement(context, tag, data, children, normalizationType)
}


```
createElement对参数进行重载，可以省略data参数从而使调用时传参更加灵活，是对_createElement的一层参数处理的封装。

```js
//src/core/vdom/create-elemenet.js

export function _createElement (
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {
  if (isDef(data) && isDef((data: any).__ob__)) {
    process.env.NODE_ENV !== 'production' && warn(
      `Avoid using observed data object as vnode data: ${JSON.stringify(data)}\n` +
      'Always create fresh vnode data objects in each render!',
      context
    )
    return createEmptyVNode()
  }
  // object syntax in v-bind
  if (isDef(data) && isDef(data.is)) {
    tag = data.is
  }
  if (!tag) {
    // in case of component :is set to falsy value
    return createEmptyVNode()
  }
  // warn against non-primitive key
  if (process.env.NODE_ENV !== 'production' &&
    isDef(data) && isDef(data.key) && !isPrimitive(data.key)
  ) {
    if (!__WEEX__ || !('@binding' in data.key)) {
      warn(
        'Avoid using non-primitive value as key, ' +
        'use string/number value instead.',
        context
      )
    }
  }
  // support single function children as default scoped slot
  if (Array.isArray(children) &&
    typeof children[0] === 'function'
  ) {
    data = data || {}
    data.scopedSlots = { default: children[0] }
    children.length = 0
  }
  if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }
  
  ...

}
```
`_createElement`要求传入的data不能是响应式的，data.is存在时不能为无效值，否则返回空文本结点。在对函数子节点设置default slot处理后，来到了重点——normalizeChildren进行children的规范化，定义在src/core/vdom/helpers/normalzie-children.js中：
```js
//src/core/vdom/helpers/normalzie-children.js 

// 2. When the children contains constructs that always generated nested Arrays,
// e.g. <template>, <slot>, v-for, or when the children is provided by user
// with hand-written render functions / JSX. In such cases a full normalization
// is needed to cater to all possible types of children values.
export function normalizeChildren (children: any): ?Array<VNode> {
  return isPrimitive(children)
    ? [createTextVNode(children)]
    : Array.isArray(children)
      ? normalizeArrayChildren(children)
      : undefined
}
```
normalizeChildren⽅法的调⽤场景有2种，⼀个场景是 render 函数是⽤户⼿写的，另⼀个场景是当编译模板、slot、v-for时候会产⽣嵌套数组的情况。
children只有⼀个节点的时候，Vue.js 从接⼝层⾯允许⽤户把 children写成基础类型⽤来创建单个简单的⽂本节点，这种情况会调⽤ createTextVNode 创建⼀个⽂本节点的 VNode；另⼀个场景是当编译 slot 、 v-for时候会产⽣嵌套数组的情况，会调⽤ normalizeArrayChildren ⽅法，
接下来看⼀下它的实现：
```js
//src/core/vdom/helpers/normalzie-children.js 

function normalizeArrayChildren (children: any, nestedIndex?: string): Array<VNode> {
  const res = []
  let i, c, lastIndex, last
  for (i = 0; i < children.length; i++) {
    c = children[i]
    if (isUndef(c) || typeof c === 'boolean') continue
    lastIndex = res.length - 1
    last = res[lastIndex]
    //  nested
    if (Array.isArray(c)) {
      if (c.length > 0) {
        c = normalizeArrayChildren(c, `${nestedIndex || ''}_${i}`)
        // merge adjacent text nodes
        if (isTextNode(c[0]) && isTextNode(last)) {
          res[lastIndex] = createTextVNode(last.text + (c[0]: any).text)
          c.shift()
        }
        res.push.apply(res, c)
      }
    } else if (isPrimitive(c)) {
      if (isTextNode(last)) {
        // merge adjacent text nodes
        // this is necessary for SSR hydration because text nodes are
        // essentially merged when rendered to HTML strings
        res[lastIndex] = createTextVNode(last.text + c)
      } else if (c !== '') {
        // convert primitive to vnode
        res.push(createTextVNode(c))
      }
    } else {
      if (isTextNode(c) && isTextNode(last)) {
        // merge adjacent text nodes
        res[lastIndex] = createTextVNode(last.text + c.text)
      } else {
        // default key for nested array children (likely generated by v-for)
        if (isTrue(children._isVList) &&
          isDef(c.tag) &&
          isUndef(c.key) &&
          isDef(nestedIndex)) {
          c.key = `__vlist${}_${i}__`
        }
        res.push(c)
      }
    }
  }
  return res
}
```
normalizeArrayChildren 接收 2 个参数， children 表⽰要规范的⼦节点， nestedIndex 表⽰嵌套的索引，用来标识v-for子节点的key。 主要流程为遍历children，对每个单个结点判断类型，数组递归normalizeArrayChildren；基础类型或文本Vnode就创建文本结点push进res数组；如果是非文本的Vnode并且是嵌套在列表中的Vnode，没有定义key则根据nestedIndex更新key值。

这里值得注意的一点优化处理，是利用last和lastIndex记录res最后成员的对象和索引，判断两个连续节点的情况合并成一个textNode。

```js
//src/core/vdom/helpers/normalzie-children.js 

// 1. When the children contains components - because a functional component
// may return an Array instead of a single root. In this case, just a simple
// normalization is needed - if any child is an Array, we flatten the whole
// thing with Array.prototype.concat. It is guaranteed to be only 1-level deep
// because functional components already normalize their own children.
export function simpleNormalizeChildren (children: any) {
  for (let i = 0; i < children.length; i++) {
    if (Array.isArray(children[i])) {
      return Array.prototype.concat.apply([], children)
    }
  }
  return children
}

```
simpleNormalizeChildren是为了在使用编译生成的render函数时对子节点的一层深度遍历，因为编译生成的child一般已经是Vnode类型，但考虑到可能有函数式组件的数组返回值，就利用Array.prototype.concat⽅法把整个
children数组打平。

`_createElement`根据normalizationType规范化后，使children变为Vnode类型数组。接下来对tag做判断，生成Vnode：
```js
////src/core/vdom/create-elemenet.js
export function _createElement (
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {

  ...

 let vnode, ns
  if (typeof tag === 'string') {
    let Ctor
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    if (config.isReservedTag(tag)) {
      // platform built-in elements
      vnode = new VNode(
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      )
    } else if (isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component
      vnode = createComponent(Ctor, data, context, children, tag)
    } else {
      // unknown or unlisted namespaced elements
      // check at runtime because it may get assigned a namespace when its
      // parent normalizes children
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      )
    }
  } else {
    // direct component options / constructor
    vnode = createComponent(tag, data, context, children)
  }
  if (Array.isArray(vnode)) {
    return vnode
  } else if (isDef(vnode)) {
    if (isDef(ns)) applyNS(vnode, ns)
    if (isDef(data)) registerDeepBindings(data)
    return vnode
  } else {
    return createEmptyVNode()
  }
}
```
如果tag是string 类型，且是built-in类型标签名，则直接创建⼀个普通VNode；匹配为已注册的组件名，则通过createComponent创建⼀个组件类型的VNode；如果没有匹配到以上两种字符，创建⼀个未知的标签的 VNode。另一种场景就是tag⼀个Component类型，直接调⽤
createComponent创建⼀个组件类型的VNode节点。其余情况返回空Vnode。
createElement在render函数中被调用，最终返回创建的Vnode Tree，回到前面提到的watch定义时传入updateComponent的更新组件功能函数，`vm._update(vm._render(), hydrating)`这段核心代码完成了Vnode到真实DOM的映射，可以看出_update是此处完成Vnode对象到DOM渲染的关键。

## 关于update
`_update`方法在lifecycleMixin时挂在在Vue原型上，首次渲染和数据更新时被调用，它的定义在src/core/instance/lifecycle.js：
```js
//src/core/instance/lifecycle.js

export function lifecycleMixin (Vue: Class<Component>) {
  Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    const prevActiveInstance = activeInstance
    activeInstance = vm
    vm._vnode = vnode
    // Vue.prototype.__patch__ is injected in entry points
    // based on the rendering backend used.
    if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    activeInstance = prevActiveInstance
    // update __vue__ reference
    if (prevEl) {
      prevEl.__vue__ = null
    }
    if (vm.$el) {
      vm.$el.__vue__ = vm
    }
    // if parent is an HOC, update its $el as well
    if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
      vm.$parent.$el = vm.$el
    }
    // updated hook is called by the scheduler to ensure that children are
    // updated in a parent's updated hook.
  }
```
忽略更新试图时与旧结点比对的变量和逻辑，这里主要执行的是`__patch__`方法并把返回值赋值给$el真实DOM，并挂载当前vue实例到该DOM元素。 `__patch__`定义在src/platforms/web/runtime/index.js中：
```js
//src/platforms/web/runtime/index.js

// install platform patch function
Vue.prototype.__patch__ = inBrowser ? patch : noop

```
向上查看patch，是执行createPatchFunction返回的函数：
```js
//src/platforms/web/runtime/patch.js

export const patch: Function = createPatchFunction({ nodeOps, modules })

```
注意到这里传入的nodeOps、modules分别是对真实DOM的操作函数和属性相关的钩子函数，向上查看createPatchFunction：
```js
//src/core/vdom/patch.js

const hooks = ['create', 'activate', 'update', 'remove', 'destroy']

  ...

export function createPatchFunction (backend) {
  let i, j
  const cbs = {}

  const { modules, nodeOps } = backend

  for (i = 0; i < hooks.length; ++i) {
    cbs[hooks[i]] = []
    for (j = 0; j < modules.length; ++j) {
      if (isDef(modules[j][hooks[i]])) {
        cbs[hooks[i]].push(modules[j][hooks[i]])
      }
    }
  }

  function emptyNodeAt (elm) {
    ...
  }

  function createRmCb (childElm, listeners) {
    ...
  }

  function removeNode (el) {
    ...
  }

  function isUnknownElement (vnode, inVPre) {
    ...
  }

  let creatingElmInVPre = 0

  function createElm (
    vnode,
    insertedVnodeQueue,
    parentElm,
    refElm,
    nested,
    ownerArray,
    index
  ) {
    ...
  }

  function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
    ...
  }

  function initComponent (vnode, insertedVnodeQueue) {
    ...
  }

  function reactivateComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
    ...
  }

  function insert (parent, elm, ref) {
    ...
  }

  function createChildren (vnode, children, insertedVnodeQueue) {
    ...
  }

  function isPatchable (vnode) {
    ...
  }

  function invokeCreateHooks (vnode, insertedVnodeQueue) {
    ...
  }

  // set scope id attribute for scoped CSS.
  // this is implemented as a special case to avoid the overhead
  // of going through the normal attribute patching process.
  function setScope (vnode) {
    ...
  }

  function addVnodes (parentElm, refElm, vnodes, startIdx, endIdx, insertedVnodeQueue) {
    ...
  }

  function invokeDestroyHook (vnode) {
    ...
  }

  function removeVnodes (parentElm, vnodes, startIdx, endIdx) {
    ...
  }

  function removeAndInvokeRemoveHook (vnode, rm) {
    ...
  }

  function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
    ...
  }

  function checkDuplicateKeys (children) {
    ...
  }

  function findIdxInOld (node, oldCh, start, end) {
    ...
  }

  function patchVnode (oldVnode, vnode, insertedVnodeQueue, removeOnly) {
    ...
  }

  function invokeInsertHook (vnode, queue, initial) {
    ...
  }

    ...
  function hydrate (elm, vnode, insertedVnodeQueue, inVPre) {
    ...
  }

  function assertNodeMatch (node, vnode, inVPre) {
    ...
  }

  return function patch (oldVnode, vnode, hydrating, removeOnly) {
    if (isUndef(vnode)) {
      if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
      return
    }

    let isInitialPatch = false
    const insertedVnodeQueue = []

    if (isUndef(oldVnode)) {
      // empty mount (likely as component), create new root element
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)
    } else {
      const isRealElement = isDef(oldVnode.nodeType)
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        // patch existing root node
        patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly)
      } else {
        if (isRealElement) {
          // mounting to a real element
          // check if this is server-rendered content and if we can perform
          // a successful hydration.
          if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
            oldVnode.removeAttribute(SSR_ATTR)
            hydrating = true
          }
          if (isTrue(hydrating)) {
            if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
              invokeInsertHook(vnode, insertedVnodeQueue, true)
              return oldVnode
            } else if (process.env.NODE_ENV !== 'production') {
              warn(
                'The client-side rendered virtual DOM tree is not matching ' +
                'server-rendered content. This is likely caused by incorrect ' +
                'HTML markup, for example nesting block-level elements inside ' +
                '<p>, or missing <tbody>. Bailing hydration and performing ' +
                'full client-side render.'
              )
            }
          }
          // either not server-rendered, or hydration failed.
          // create an empty node and replace it
          oldVnode = emptyNodeAt(oldVnode)
        }

        // replacing existing element
        const oldElm = oldVnode.elm
        const parentElm = nodeOps.parentNode(oldElm)

        // create new node
        createElm(
          vnode,
          insertedVnodeQueue,
          // extremely rare edge case: do not insert if old element is in a
          // leaving transition. Only happens when combining transition +
          // keep-alive + HOCs. (#4590)
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        )

        // update parent placeholder node element, recursively
        if (isDef(vnode.parent)) {
          let ancestor = vnode.parent
          const patchable = isPatchable(vnode)
          while (ancestor) {
            for (let i = 0; i < cbs.destroy.length; ++i) {
              cbs.destroy[i](ancestor)
            }
            ancestor.elm = vnode.elm
            if (patchable) {
              for (let i = 0; i < cbs.create.length; ++i) {
                cbs.create[i](emptyNode, ancestor)
              }
              // #6513
              // invoke insert hooks that may have been merged by create hooks.
              // e.g. for directives that uses the "inserted" hook.
              const insert = ancestor.data.hook.insert
              if (insert.merged) {
                // start at index 1 to avoid re-invoking component mounted hook
                for (let i = 1; i < insert.fns.length; i++) {
                  insert.fns[i]()
                }
              }
            } else {
              registerRef(ancestor)
            }
            ancestor = ancestor.parent
          }
        }

        // destroy old node
        if (isDef(parentElm)) {
          removeVnodes(parentElm, [oldVnode], 0, 0)
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode)
        }
      }
    }

    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
    return vnode.elm
  }
}
```
首先获取到modules里的各个时期的钩子函数并缓存在cbs对象中，接着创建大量辅助函数，最终返回封装好的patch函数。这样的设计是因为patch函数牵涉到DOM的映射和属性更新，与环境平台强相关，这里将与平台相关配置的nodeOps和modules传入createPatchFunction并生成最终可直接在当前环境直接调用的patch函数，这种提前固化差异、在函数中返回配置好的函数的方式就是[函数柯里化](https://segmentfault.com/a/1190000018265172)的技巧。
这篇主要只关心初始化过程的patch函数主线逻辑，可以看到由于没有渲染过，oldVnode是DOM容器，用emptyNodeAt(oldVnode)替换oldVnode使其转化为Vnode对象，再调用createElm创建出真实DOM并插⼊到它的⽗节点中，
```js
//src/core/vdom/patch.js

function createElm (
    vnode,
    insertedVnodeQueue,
    parentElm,
    refElm,
    nested,
    ownerArray,
    index
  ) {
    if (isDef(vnode.elm) && isDef(ownerArray)) {
      // This vnode was used in a previous render!
      // now it's used as a new node, overwriting its elm would cause
      // potential patch errors down the road when it's used as an insertion
      // reference node. Instead, we clone the node on-demand before creating
      // associated DOM element for it.
      vnode = ownerArray[index] = cloneVNode(vnode)
    }

    vnode.isRootInsert = !nested // for transition enter check
    if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
      return
    }

    const data = vnode.data
    const children = vnode.children
    const tag = vnode.tag
    if (isDef(tag)) {
      if (process.env.NODE_ENV !== 'production') {
        if (data && data.pre) {
          creatingElmInVPre++
        }
        if (isUnknownElement(vnode, creatingElmInVPre)) {
          warn(
            'Unknown custom element: <' + tag + '> - did you ' +
            'register the component correctly? For recursive components, ' +
            'make sure to provide the "name" option.',
            vnode.context
          )
        }
      }

      vnode.elm = vnode.ns
        ? nodeOps.createElementNS(vnode.ns, tag)
        : nodeOps.createElement(tag, vnode)
      setScope(vnode)

      /* istanbul ignore if */
      if (__WEEX__) {
        // in Weex, the default insertion order is parent-first.
        // List items can be optimized to use children-first insertion
        // with append="tree".
        const appendAsTree = isDef(data) && isTrue(data.appendAsTree)
        if (!appendAsTree) {
          if (isDef(data)) {
            invokeCreateHooks(vnode, insertedVnodeQueue)
          }
          insert(parentElm, vnode.elm, refElm)
        }
        createChildren(vnode, children, insertedVnodeQueue)
        if (appendAsTree) {
          if (isDef(data)) {
            invokeCreateHooks(vnode, insertedVnodeQueue)
          }
          insert(parentElm, vnode.elm, refElm)
        }
      } else {
        createChildren(vnode, children, insertedVnodeQueue)
        if (isDef(data)) {
          invokeCreateHooks(vnode, insertedVnodeQueue)
        }
        insert(parentElm, vnode.elm, refElm)
      }

      if (process.env.NODE_ENV !== 'production' && data && data.pre) {
        creatingElmInVPre--
      }
    } else if (isTrue(vnode.isComment)) {
      vnode.elm = nodeOps.createComment(vnode.text)
      insert(parentElm, vnode.elm, refElm)
    } else {
      vnode.elm = nodeOps.createTextNode(vnode.text)
      insert(parentElm, vnode.elm, refElm)
    }
  }

```
经过尝试创建⼦组件和对tag的检测，如果当前Vnode是文本或者注释结点则直接insert进父节点，否则来到重点逻辑，即调用配置过的当前环境平台的DOM操作来创建占位符元素。接着用createChildren创建子元素：
```js
//src/core/vdom/patch.js

 function createChildren (vnode, children, insertedVnodeQueue) {
    if (Array.isArray(children)) {
      if (process.env.NODE_ENV !== 'production') {
        checkDuplicateKeys(children)
      }
      for (let i = 0; i < children.length; ++i) {
        createElm(children[i], insertedVnodeQueue, vnode.elm, null, true, children, i)
      }
    } else if (isPrimitive(vnode.text)) {
      nodeOps.appendChild(vnode.elm, nodeOps.createTextNode(String(vnode.text)))
    }
  }
```
这里逻辑比较清晰，遍历children数组递归调用createElm，做子节点的DOM映射，注意到这里第三个参数vnode.elm父容器的DOM作为占位符；或是直接添加文本结点到vnode。
接着invokeCreateHooks⽅法执⾏所有的 create 的钩⼦并把vnode给push到
insertedVnodeQueue中，invokeCreateHooks定义如下：
```js
//src/core/vdom/patch.js

function invokeCreateHooks (vnode, insertedVnodeQueue) {
for (let i = 0; i < cbs.create.length; ++i) {
cbs.create[i](emptyNode, vnode)
}
i = vnode.data.hook // Reuse variable
if (isDef(i)) {
if (isDef(i.create)) i.create(emptyNode, vnode)
if (isDef(i.insert)) insertedVnodeQueue.push(vnode)
}

```
最后调用insert方法把DOM插⼊到⽗节点中，因为这里是先遍历子节点再插入的递归调用，所以是深度优先的先之后父的插入顺序。
至此，初始化过程从Vnode到DOM的映射过程基本完成，跟生命周期钩子函数相关内容在稍后作篇幅讲述。

# 总结
这篇讲述了从模板到渲染DOM的初始化主线流程，简化为最基础的步骤。可以看到初始化从模板->[编译]->render函数->Vnode->update函数->patch函数->createEle函数->DOM整个的相对直观的过程，真实使用场景远比此复杂，这也是为什么每个过程都有边界判断处理和逻辑支线。下一篇讲述Vue在项目中核心思想———组件化过程。

参考来源：https://juejin.im/post/5b3a067c51882551c7026d5f
         https://juejin.im/post/5b86f6cc5188256fd44c0ce9
         https://juejin.im/post/5d12c931f265da1bb2773fcc
