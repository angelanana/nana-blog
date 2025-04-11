---
title: Vue3组合式API最新实践
date: 2025-04-01 15:38:29
tags:
  - Vue3
---

# 前言
Vue3组合式API是Vue3引入的一种新的API风格，它通过将组件的逻辑拆分为多个可重用的函数，使得组件的代码更加清晰、可维护和可测试。本文将介绍Vue3组合式API的最新实践，包括最佳实践、性能优化和未来发展趋势。

# 一、为什么组合式API是Vue3的革命性升级
## 1.1 选项式API的痛点
- <strong>代码碎片化：</strong>数据在 `data`，方法在 `methods`，计算属性在`computed`，生命周期钩子在 `mounted` 等选项中分散。
- <strong>逻辑耦合：</strong>1000行组件中去找关联逻辑如同大海捞针。
- <strong>可维护性差：</strong>Mixins存在命名冲突和来源不清晰问题。

```js
// 传统Options API（用户管理组件）
export default {
  data () {
    return {
      users: [],
      filters: {},
      pagination: {}
    }
  },
  methods: {
    fetchUsers () {
      // 实现用户数据获取逻辑
    },
    deleteUsers () {
      // 实现用户过滤逻辑
    },
    handlePagination () {
      // 实现分页逻辑
    }
  },
  computed: {
    filteredUsers () {
      // 实现用户过滤计算
    }
  },
  watch: {
    filters: {
      handler () {
        // 实现用户过滤监听
      },
      deep: true
    }
  }
}
```

## 1.2 组合式API的三大优势
- <strong>逻辑聚合：</strong>按功能而非选项组织代码
- <strong>完美复用：</strong>函数式封装实现"即插即用"
- <strong>类型支持：</strong>天然适配TypeScript
```js
// 使用组合式API重构
import { useUserFetch } from './composables/userFetch'
import { useTableFilter } from './composables/tableFilter'
 
export default {
  setup() {
    const { users, fetchUsers } = useUserFetch()
    const { filteredData, filters } = useTableFilter(users)
    
    return { users, filteredData, filters, fetchUsers }
  }
}
```
<!-- 引入对比图 -->
![图片](/images/Vue2vsVue3.png)

# 二、组合式API核心机制深度剖析（附完整代码）
## 2.1 setup函数：新世界的入口
```vue
<template>
  <button @click="increment">{{ count }}</button>
</template>

<script setup>
// 编译器语法糖
import { ref, onMounted } from 'vue'
const count = ref(0)
function increment() {
  count.value++
}
</script>
```

**关键细节：**
- **执行时机**：在`beforeCreate`之前
- **参数解析**：`props`是响应式的，不用解构
- **Context对象**：包含`attrs` / `slots` / `emit`等

## 2.2 ref() vs reactive() 选择指南

| 场景           | 推荐方案                     | 原因                     |
|----------------|------------------------------|--------------------------|
| 基础类型数据   | `ref()`                      | 自动解包，模板使用更方便 |
| 复杂对象/数组  | `reactive()`                 | 深层响应式，性能更优     |
| 第三方类实例   | `reactive()`                 | 保持原型链方法           |
| 跨组件状态共享 | `ref()` + `provide/inject`  | 响应式追踪更可控         |

**ref的底层原理**
```js
// 深响应式
export function ref(value?: unknown) {
  return createRef(value, false)
}

// 浅响应式
export function shallowRef(value?: unknown) {
  return createRef(value, true)
}

function createRef(rawValue: unknown, shallow: boolean) {
  // 如果传入的值已经是一个 ref，则直接返回它
  if (isRef(rawValue)) {
    return rawValue
  }
  // 否则，创建一个新的 RefImpl 实例
  return new RefImpl(rawValue, shallow)
}

class RefImpl<T> {
  // 存储响应式的值。我们追踪和更新的就是_value。（这个是重点）
  private _value: T
  // 用于存储原始值，即未经任何响应式处理的值。（用于对比的，这块的内容可以不看）
  private _rawValue: T 

  // 用于依赖跟踪的 Dep 类实例
  public dep?: Dep = undefined
  // 一个标记，表示这是一个 ref 实例
  public readonly __v_isRef = true

  constructor(
    value: T,
    public readonly __v_isShallow: boolean,
  ) {
    // 如果是浅响应式，直接使用原始值，否则转换为非响应式原始值
    this._rawValue = __v_isShallow ? value : toRaw(value)
    // 如果是浅响应式，直接使用原始值，否则转换为响应式值
    this._value = __v_isShallow ? value : toReactive(value)
    
    // toRaw 用于将响应式引用转换回原始值
    // toReactive 函数用于将传入的值转换为响应式对象。对于基本数据类型，toReactive 直接返回原始值。
    // 对于对象和数组，toReactive 内部会调用 reactive 来创建一个响应式代理。
    // 因此，对于 ref 来说，基本数据类型的值会被 RefImpl 直接包装，而对象和数组
    // 会被 reactive 转换为响应式代理，最后也会被 RefImpl 包装。
    // 这样，无论是哪种类型的数据，ref 都可以提供响应式的 value 属性，
    // 使得数据变化可以被 Vue 正确追踪和更新。
    // export const toReactive = (value) => isObject(value) ? reactive(value) : value
  }

  get value() {
    // 追踪依赖，这样当 ref 的值发生变化时，依赖这个 ref 的组件或副作用函数可以重新运行。
    trackRefValue(this)
    // 返回存储的响应式值
    return this._value
  }

  set value(newVal) {
    // 判断是否应该使用新值的直接形式（浅响应式或只读）
    const useDirectValue =
      this.__v_isShallow || isShallow(newVal) || isReadonly(newVal)
    // 如果需要，将新值转换为非响应式原始值
    newVal = useDirectValue ? newVal : toRaw(newVal)
    // 如果新值与旧值不同，更新 _rawValue 和 _value
    if (hasChanged(newVal, this._rawValue)) {
      this._rawValue = newVal
      this._value = useDirectValue ? newVal : toReactive(newVal)
      // 触发依赖更新
      triggerRefValue(this, DirtyLevels.Dirty, newVal)
    }
  }
}

```
ref是一个函数，它接收一个内部值并且返回一个响应式且可变的引用对象。这个引用对象有一个`.value`属性，这个属性指向内部值。

**reactive的底层原理**
`reactive`是一个函数，它接受一个对象并返回该对象的响应式代理，也就是`proxy`。
```js
function reactive(target) {
  if (target && target.__v_isReactive) {
    return target
  }

  return createReactiveObject(
    target,
    false,
    mutableHandlers,
    mutableCollectionHandlers,
    reactiveMap
  )
}

function createReactiveObject(
  target,
  isReadonly,
  baseHandlers,
  collectionHandlers,
  proxyMap
) {
  if (!isObject(target)) {
    return target
  }
  
  const existingProxy = proxyMap.get(target)
  if (existingProxy) {
    return existingProxy
  }

  const proxy = new Proxy(target, baseHandlers)
  proxyMap.set(target, proxy)
  return proxy
}
```
`reactive`的源码简单很多，`reactive`通过`new Proxy(target, baseHandlers)` 创建了一个代理。这个代理会拦截对目标对象的操作，从而实现响应式。

具体的使用：
```js
import { reactive } from 'vue'

let state = reactive({ count: 0 })
state.count++
```

可以看出ref和reactive的区别：
- 底层原理不一样
- `ref`采用`RefImpl`实现，`reactive`则采用`Proxy`实现

### ref更深入的理解
当你使用 `new RefImpl(value)` 创建一个 `RefImpl` 实例时，实际上发生了以下几个步骤：
1. **内部值**：实例存储了传递给构造函数的初始值。
2. **依赖收集**：实例需要跟踪所有依赖于他的效果（effect），例如计算属性或者副作用函数。通常通过一个依赖列表或函数来实现。
3. **触发更新**：当实例的值发生变化时，它需要通知所有依赖它的效果进行更新，以便他们可以重新计算或执行。

RefImpl 类似于发布-订阅模式的设计，以下是一个简化的`RefImpl`类的伪代码：
```js
class Dep {
  constructor() {
    this.subscribers = new Set();
  }
  depend() {
    if (activeEffect) {
      this.subscribers.add(activeEffect);
    }
  }
  notify() {
    this.subscribers.forEach(effect => effect());
  }
}

let activeEffect = null;

function watchEffect(effect) {
  activeEffect = effect;
  effect();
  activeEffect = null;
}

class RefImpl {
  constructor(value) {
    this._value = value;
    this.dep = new Dep();
  }
  get value() {
    // 当获取值时，进行依赖收集
    this.dep.depend();
    return this._value;
  }
  set value(newValue) {
    if (newValue !== this._value) {
      this._value = newValue;
      // 值改变时，触发更新
      this.dep.notify();
    }
  }
}

// 使用实例
let count = new RefImpl(0);

watchEffect(() => {
  console.log('Effect 1:', count.value); // 订阅变化
})
count.value = 1; // 触发更新
```

Dep类负责管理一个依赖列表，并提供依赖收集和通知更新的功能；RefImpl类包含一个内部值_value和一个Dep实例。当value被访问时，通过get方法进行依赖收集；当value被赋予新值时，通过set方法触发更新。

### reactive的局限性
Vue3中，reactive API 通过 proxy实现了一种响应式数据的方式，尽管这种方法在性能上比vue2有所提升，但proxy的局限性也导致了reactive的局限性，这些局限性会影响开发者的使用体验。
1.仅对引用数据类型有效
reactive 主要适用于对象，包括数组和一些集合类型（如：Map和Set）。对于基础数据类型（如：string、number、boolean），reactive 无法直接响应。
2.使用不当会失去响应
  - 直接赋值对象，失去响应性
  - 直接替换响应对象，失去响应性
  - 直接解构对象，会失去响应性。解决：用toRefs函数来将响应式对象转换为ref对象
  - 将响应式对象的属性赋值给变量，会失去响应式

# 高级实战技巧
## 3.1 通用数据请求封装
```js
// useFetch.js
export const useFetch = (url) => {
  const data = ref(null)
  const error = ref(null)
  const loading = ref(false)

  const fetchData = async () => {
    try {
      loading.value = true
      const response = await axios.get(url)
      data.value = response.data
    } catch (err) {
      error.value = err
    } finally {
      loading.value = false
    }
  }
  onMounted(fetchData)
  return { data, error, loading, retry: fetchData }
}

// 组件中使用
const { data: posts } = useFetch('/api/posts')
```
## 3.2 防抖探索实战
```js
// useDebounceSearch.js
export function useDebounceSearch(callback, delay=500) {
  const searchQuery = ref('')
  let timeoutId = null

  watch(searchQuery, (newVal) => {
    clearTimeout(timeoutId)
    timeoutId = setTimeout(() => {
      callback(newVal)
    }, delay)
  })

  return { searchQuery }
}
```

# 性能优化最佳实践
## 4.1 计算属性缓存策略
```js
const filteredList = computed(() => {
  // 通过闭包缓存中间结果
  const cached = {}
  return (filterKey) => {
    if(cache[filterKey]) return cache[filterKey]
    return cache[filterKey] = heavyCompute()
  }
})
```

## 4.2 watchEffect()的高级用法
```js
// 立即执行+自动追踪依赖
watchEffect(() => {
  const data = fetchData(params.value)
  console.log('依赖自动追踪：',data)
}, {
  flush: 'post', // DOM更新后执行
  onTrack(e) {} // 调试追踪
})
```

## 4.3 内催泄露防范
```js
// 定时器示例
onMounted(() => {
  const timer = setInterval(() => {...}, 1000)
  onUnmounted(() => clearInterval(timer))
})
```

# TypeScript终级适配方案
```js
interface User {
  name: string
  age: number,
  id: number
}

// 带类型的Ref
const user = ref<User>({id: 1, name: 'Alice', age: 25})

// 组合函数类型定义
export function useCounter(): {
  count: Ref<number>
  increment: () => void
} {
  // 实现逻辑...
}
```

# 总结
如果你对这篇文章有任何疑问、建议或者独特的见解，欢迎在评论区留言。无论是探讨技术细节，还是分享项目经验，都能让我们共同进步。