---
title: Vuejs Composition API
recorddate: 2020-03-25
updatedate: [2020-08-13, 2020-08-15, 2020-08-16, 2020-08-25]
---

# Vuejs RFCs Composition API

[Vuejs-RFC-0013-composition-api][rfc-0013]

[rfc-0013]: https://github.com/vuejs/rfcs/blob/master/active-rfcs/0013-composition-api.md

适用版本： 2.x / 3.x

> 由于此 RFC 很长，所以另在独立页面部署：
>
> - Vue Composition API [RFC](https://vue-composition-api-rfc.netlify.com/)
> - [API Reference](https://vue-composition-api-rfc.netlify.com/api/)

## 🧭 摘要

介绍 Composition API ：包含一组可添加的、基于函数的 API，允许灵活地组合组件逻辑。

## 🌱 基本示例

```vue
<template>
  <button @click="increment">
    Count is: {{ state.count }}, double is: {{ state.double }}
  </button>
</template>

<script>
import { reactive, computed } from 'vue'

export default {
  setup() {
    const state = reactive({
      count: 0,
      double: computed(() => state.count * 2)
    })

    function increment() {
      state.count++
    }

    return {
      state,
      increment
    }
  }
}
</script>
```

## 🍃 动机

### 🔸 逻辑复用和代码组织

现在将代码组织为每个函数都执行特定的功能，而不必总是通过选项来组织代码。

新的 API 还使得在组件之间提取和重用逻辑变得更加简单。

### 🔸 更好的类型推断

新 API 大多使用的是普通的变量和函数，它们自然是类型友好的。

使用新 API 编写的代码可以享受完整的类型推断，几乎不需要手动类型提示。

使用新 API 编写的 TypeScript 代码与通过 JavaScript 编写的看起来几乎相同。

因此，即使是使用 JavaScript 的用户也可以获得更好的 IDE 支持。

## 📜 详细设计

> - API 简介
> - 代码组织
> - 逻辑提取和重用
> - 与现有 API 一起使用
> - 插件开发

### 1️⃣ API 简介

这里提出的 API 并没有引入新的概念，而是更多地将 Vue 的核心功能公开为独立功能。

🔹 **Reactive State and Side Effects** 响应状态和副作用

```js
import { reactive } from 'vue'

const state = reactive({ count: 0 })
```

`reactive()` 与 2.x 版本中的 `Vue.observable()` 方法等效，
重命名是为了避免与 RxJS 的可观察对象 ( Observables ) 混淆。

上例返回的 `state` 为 Vue 用户应该熟悉的可反应对象。

Vue 中可反应状态的基本用例是我们可以在渲染期间使用它。
由于依赖项跟踪，视图在响应状态更改时自动更新。

在 DOM 中渲染一些东西被认为是副作用 ( Side Effect ) 的：
我们的程序正在修改程序本身 ( DOM ) 外部的状态。

要应用并根据反应状态自动重新应用副作用，我们可以使用 `watchEffect()` API ：

```js
import { reactive, watchEffect } from 'vue'

const state = reactive({ count: 0 })

watchEffect(() => {
  document.body.innerHTML = `count is ${state.count}`
})
```

`watchEffect()` 方法的参数期望是一个具有可实现所需副作用的函数，在上例中设置了 `innerHTML` 值。

`watchEffect()` 会立即执行该函数，并跟踪其在执行期间作为依赖项使用的所有可反应状态属性。

上例中的 `state.count` 在初始执行后，将作为此监视程序的依赖项进行跟踪。
当 `state.count` 在将来的某个时候发生变化时，内部函数将再次执行。

这正是 Vue 可反应体系的精髓。当你从组件中的 `data()` 返回一个对象时，它在内部被 `reactive()` 处理为可响应的。

Vue 模板被编译为使用了这些可反应属性的渲染函数，可以认为它是一个更高效的 `innerHTML` 。

> `watchEffect()` 与 2.x 版本中的 `watch` 选项类似，但它不需要将所观察的数据源和副作用回调分离。
>
> Composition API 还提供了 `watch()` 函数，实现与 2.x 中 `watch` 选项相同功能。

继续上面例子，让我们来处理用户的输入操作：

```js
function increment() {
  state.count++
}

document.body.addEventListener('click', increment)
```

下面让我们用一个假设的 `renderTemplate()` 方法来简化示例，这样我们就可以专注于响应性方面：

```js
import { reactive, watchEffect } from 'vue'

const state = reactive({ count: 0 })

function increment() {
  state.count++
}

const renderContext = {
  state,
  increment
}

watchEffect(() => {
  // 假设的内部代码，不是实际的 API
  renderTemplate(
    `<button @click="increment">{{ state.count }}</button>`,
    renderContext
  )
})
```

🔹 **Computed State and Refs** 计算状态和引用

有时我们需要依赖于其他状态的状态 - 在 Vue 中是通过计算属性 ( Computed Properties ) 来处理的。

要创建一个计算值，我们可以直接使用 `computed()` API ：

```js
import { reactive, computed } from 'vue'

const state = reactive({ count: 0 })

const double = computed(() => state.count * 2)
```

`computed()` 函数返回的是什么？若猜测 `computed()` 的内部实现，我们可能会想到以下内容：

```js
// 简化的伪代码
function computed(getter) {
  let value
  watchEffect(() => {
    value = getter()
  })
  return value
}
```

但我们知道这是行不通的，如果 `value` 是类似于 `number` 的基本类型，
一旦返回，则它与内部 `computed()` 的更新逻辑的连接将丢失。
这是因为 JS 基本类型是通过值而不是通过引用传递的。

将值作为属性分配给对象时，也会出现相同的问题。
如果一个响应性值作为属性分配或从函数返回时不能保持其响应性。
为了确保我们始终可以读取计算的最新值，我们需要将实际值包装在一个对象中，然后返回该对象：

```js
// 简化的伪代码
function computed(getter) {
  const ref = {
    value: null
  }
  watchEffect(() => {
    ref.value = getter()
  })
  return ref
}
```

此外，我们还需要拦截对对象的 `.value` 属性的读/写操作，以执行依赖项跟踪和更改通知。
现在我们可以通过引用传递计算值，而不必担心失去响应性。
代价是，我们现在需要通过 `.value` 来获取最新的值：

```js
const double = computed(() => state.count * 2)

watchEffect(() => {
  console.log(double.value)
}) // -> 0

state.count++ // -> 2
```

上例的 `double` 是一个我们称为 `ref` 的对象，因为它是对它所持有的内部值的响应式引用。

- ref 对象有一个指向内部值的单一属性 `.value`

使用 Composition API 时，响应式引用和模板引用的概念是统一的。

> [Template Refs - Vue Composition API](https://composition-api.vuejs.org/api.html#template-refs)

为了获得对模板中元素或组件实例的引用，我们可以像往常一样声明 `ref` 并从 `setup()` 中返回：

```vue
<template>
  <div ref="root"></div>
</template>

<script>
import { ref, onMounted } from 'vue'

export default {
  setup() {
    const root = ref(null)

    onMounted(() => {
      // 初始渲染后 DOM 元素将被分配给 ref
      console.log(root.value) // <div/>
    })

    return { root }
  }
}
</script>
```

除了计算引用之外，我们还可以使用 `ref()` API 直接创建普通的可变引用：

```js
const count = ref(0)
console.log(count.value) // 0

count.value++
console.log(count.value) // 1
```

🔹 **Ref Unwrapping** 引用（Ref）展开

我们可以将 ref 公开为渲染上下文的属性。

在内部，Vue 将对 refs 进行特殊处理，这样当在渲染上下文中遇到 ref 时，该上下文直接暴露其内部值。

这意味着在模板中，我们可以直接写 `{{count}}` 而不是 `{{count.value}}` 。

下面是同一个的计数器示例的另一个版本，其使用了 `ref` 来替换 `reactive` ：

```js
import { ref, watchEffect } from 'vue'

const count = ref(0)

function increment() {
  count.value++
}

const renderContext = {
  count,
  increment
}

watchEffect(() => {
  renderTemplate(
    `<button @click="increment">{{ count }}</button>`,
    renderContext
  )
})
```

此外，当 ref 作为一个属性嵌套在一个反应性对象下时，它也会在访问时自动展开：

```js
const state = reactive({
  count: 0,
  double: computed(() => state.count * 2)
})

// 不必使用 `state.double.value`
console.log(state.double)
```

🔹 **Usage in Components** 在组件中使用

到目前为止，我们的代码已经提供了可以根据用户输入进行更新的工作 UI，但是该代码仅运行一次且不可重用。

如果我们想重用逻辑，那么合理的下一步似乎是将其重构为一个函数：

```js
import { reactive, computed, watchEffect } from 'vue'

function setup() {
  const state = reactive({
    count: 0,
    double: computed(() => state.count * 2)
  })

  function increment() {
    state.count++
  }

  return {
    state,
    increment
  }
}

const renderContext = setup()

watchEffect(() => {
  renderTemplate(
    `<button @click="increment">
      Count is: {{ state.count }}, double is: {{ state.double }}
    </button>`,
    renderContext
  )
})
```

> 注意，上面的代码并不依赖于组件实例的存在。
>
> 实际上，到目前为止介绍的 API 都可以在组件上下文之外使用，
> 这使我们能够在更广泛的场景中利用 Vue 的反应性系统 ( reactivity system ) 。

现在，如果我们把【调用 `setup()` 、创建监视器、以及模板渲染】的任务交给框架，
我们就可以仅使用 `setup()` 函数和模板来定义一个组件：

```vue
<template>
  <button @click="increment">
    Count is: {{ state.count }}, double is: {{ state.double }}
  </button>
</template>

<script>
import { reactive, computed } from 'vue'

export default {
  setup() {
    const state = reactive({
      count: 0,
      double: computed(() => state.count * 2)
    })

    function increment() {
      state.count++
    }

    return {
      state,
      increment
    }
  }
}
</script>
```

🔹 **Lifecycle Hooks** 生命周期钩子

我们知道可以使用 `watchEffect()` 和 `watch` APIs 来应用基于状态变化的副作用。

至于在不同的生命周期钩子中执行副作用，我们可以使用专用的 `onXXX` APIs （对应现有的生命周期选项）：

```js
import { onMounted } from 'vue'

export default {
  setup() {
    onMounted(() => {
      console.log('component is mounted!')
    })
  }
}
```

这些生命周期注册方法只能在 `setup()` 钩子调用当中使用。

因为它们依赖于内部全局状态来定位当前活动实例。

在没有当前活动实例的情况下调用它们将导致错误。

它会自动推算出，使用内部全局状态调用 `setup()` 钩子的当前实例。

这样设计是为了减少将逻辑提取到外部函数时的摩擦。

### 2️⃣ 代码组织

至此，我们已经用导入的函数复制了组件 API，但是这是为了什么呢?
用选项定义组件似乎比在一个大函数中混合所有功能更有条理!

有这样的第一印象是可以理解的。
但是正如在动机一节中提到的，我们相信复合 API 实际上会带来更好的代码组织，特别是在复杂的组件中。
下面，我们将尝试解释其中的原因。

🔹 什么是有组织的代码？

让我们后退一步，考虑一下我们所说的【有组织的代码】 ( Organized Code ) 到底是什么意思。

保持代码井井有条 ( Organized ) 的最终目标应该是使代码更容易阅读和理解。

在理解一个组件时，我们更关心的是【组件正在尝试实现什么功能】而不是【组件用到了哪些选项】。

使用基于选项的 API 编写的代码，自然的回答了后者，但是在表达前者方面做的相当糟糕。

🔹 逻辑关注点 VS 选项类型

对于复杂的组件用例，可读性问题尤为突出。

阅读基于选项的组件代码时，很难立即梳理出各个功能逻辑关注点，
因为对于特定功能逻辑相关的代码通常会分散在各处。

例如一个功能用到了两个数据属性、一个计算属性和一个方法，但数据和方法间隔了几十行甚至上百行。

正是这种碎片化 ( Fragmentation ) 使得很难去理解和维护一个复杂的组件。

如果我们能够将相同逻辑问题的相关代码集中放在一起，那就会好很多。
这正是 Composition API 使我们能够做到的。

我们可以将某一功能相关的所有逻辑都合并封装在一个函数中。
由于该函数有一个描述性的名称，该函数在某种程度上是自描述的，
这就是我们所说的组合函数 ( Composition Function ) 。

建议约定使用 `use` 作为函数的开头，来表示它是一个实现了某一功能的组合函数。

现在，每个逻辑关注点的代码都被组合进了一个个函数中，这使得组件逻辑更清晰并更易浏览。

```js
function useA() {
  //...
  return A
}
function useB() {
  //...
  return { B, C }
}
function useD(A, C) {
  //...
  return D
}

export default {
  setup() {
    const A = useA()
    const { B, C } = useB()
    const D = useD(A, C)
    return { B, D }
  }
}
```

`setup()` 方法现在的主要作用就是作为所有组合函数的调用入口。

`setup()` 方法读起来就像是对组件要执行的操作的口头描述，这在基于选项的版本中是完全缺失的信息。

我们还可以通过传递的参数，清楚的看到组合函数之间的依赖关系流 ( Dependency Flow ) 。

`setup()` 方法最后的 `return` 语句是查看暴露给模板什么内容的唯一位置。

基于选项的 API 迫使我们根据选项类型 ( Option Types ) 组织代码，
而 Composition API 使我们能够基于逻辑关注点 ( Logical Concerns ) 来组织代码。

### 3️⃣ 逻辑提取和复用

当涉及到跨组件提取和复用逻辑时【组合式 API 】 ( Composition API ) 非常灵活。

组合函数并不依赖于微妙的 `this` 上下文，而是只依赖于它的参数和全局导入的 Vue API 。

你可以非常方便的复用组件内的逻辑，只需将其封装为一个函数来引用。

你甚至可以通过导出一个组件的整个 `setup()` 函数，来达到和 `extends` 一样的效果。

类似的逻辑复用也可以通过使用现有的模式来实现，比如 `mixins` 、
高阶组件 ( Higher-order Components ) 或（通过作用域插槽实现的）无渲染组件。
更高层次的想法是，与组合函数相比，这些模式都有各自的缺点:

- 在渲染上下文中暴露的属性来源不不清晰。
  例如在阅读一个使用了多个混入的组件时，很难分辨出某个特定属性是从哪个混入注入的。

- 命名空间冲突。混入可能会在属性和方法名称上产生潜在的冲突，
  而高阶组件 ( HOC ) 可能在预期的 prop 名称上发生冲突。

- 性能方面，高阶组件 ( HOC ) 和无渲染组件都需要额外的有状态的组件实例，因此造成性能的损耗。

相比之下，使用 Composition API 有以下优势：

- 暴露给模板的属性有明确的来源，因为它们都是从组合函数返回的值。

- 合成函数的返回值可以任意命名，因此不会有命名空间冲突。

- 不需要为逻辑重用而创建不必要的组件实例。

### 4️⃣ 与现有的 API 一起使用

Composition API 可以与现有的基于选项的 API 一起使用。

- Composition API 会在 2.x 版本的选项 ( `data`, `computed` & `methods` ) 之前解析，
  并且无法访问由这些选项定义的属性。

- 从 `setup()` 函数返回的属性将被暴露给 `this` 对象，并可在 2.x 版本的选项中访问。

### 5️⃣ 插件开发

如今许多 Vue 插件都会将属性注入 `this` 当中。

例如 Vue Router 注入了 `this.$route` 和 `this.$router` ， Vuex 注入了 `this.$store` 。

这使得类型推断变得棘手 ( Tricky ) ，因为每个插件都需要用户为注入的属性增加 Vue 类型定义。

当使用 Composition API 时，由于不使用 `this` ，插件可以在内部使用【依赖注入】
( `provide` & `inject` ) 并公开为一个组合函数。

定义一个插件：

```js
// store-plugin.js
import { provide, inject } from 'vue'

const StoreSymbol = Symbol()

export function provideStore(store) {
  provide(StoreSymbol, store)
}

export function useStore() {
  const store = inject(StoreSymbol)
  if (!store) {
    throw new Error('Store plugin error.')
  }
  return store
}
```

使用插件：

```js
// 根组件中提供数据
import { provideStore } from 'path/to/store-plugin'
import store from 'path/to/store'

const App = {
  setup() {
    provideStore(store)
  }
}

// 子组件中使用数据
import { useStore } from 'path/to/store-plugin'

const Child = {
  setup() {
    const store = useStore()
  }
}
```

注意，也可以通过 [全局 API 更改提案](rfc-0009) 中建议的应用程序 App 级别的 `provide` 来提供 `store` 数据。

[rfc-0000]: https://github.com/vuejs/rfcs/blob/master/active-rfcs/0009-global-api-change.md#provide--inject

定义插件：

```js
// store-plugin2.js
import { inject } from 'vue'

const StoreSymbol = Symbol()

export function useStore() {
  const store = inject(StoreSymbol)
  //...
}

export default {
  install(app, store) {
    app.provide(StoreSymbol, store)
  }
}
```

使用插件：

```js
import { createApp } from 'vue'
import App from './App.vue'
import storePlugin from 'path/to/store-plugin2'
import store from 'path/to/store'

const app = createApp(App)

app.use(storePlugin, store)

app.mount('#app')
```

在子组件中使用注入还是跟 `useStore()` 风格是一样的。

## 💔 缺点

> - 引入 Ref 的开销
> - Ref vs. Reactive
> - 沉长的返回语句
> - 更多的灵活性需要更多的自我约束

### 🔸 引入 Refs 的开销

从技术上讲 Ref 是此提案中唯一引入的“新”概念。

引入 Ref 是为了将响应式的值作为变量传递，而无需依赖对 `this` 对象的访问。

1. 当使用 Composition API 时，我们需要不断地将【响应式引用】 ( Refs ) 与普通值和对象区分开来，
   因而增加了使用此 API 的精神负担。
   通过使用命名约定或使用类型系统，可以极大地减轻这种心理负担。
   例如为所有 Ref 变量添加 `xxxRef` 后缀。
   换句话说，由于提高了代码组织的灵活性，因此组件逻辑可以被分割为一些小的函数，
   些函数的局部上下文都很简单，引用的开销也比较容易管理。

2. 由于需要通过 `.value` 访问，读取和修改 Ref 要比使用普通值更冗长。

我们已经讨论过是否可以完全避免使用 Ref 概念，而只是用响应性对象，但是：

- 计算值的 getters 可以返回基础类型，因此类似 Ref 的容器是不可避免的。

- 一些只接收或返回基本类型的组合函数需要将其值包裹在对象中才能达到响应性目的。
  若框架不提供标准实现，用户很可能最终会发明他们自己的类 Ref 模式，并导致生态系统碎片化。

### 🔸 Ref vs. Reactvie

可以理解的是，用户可能会对 `ref` 和 `reactive` 之间该使用哪一个感到困惑。

首先要明白，这两个概念你必须都要理解，才能有效地使用 Composition API 。
只是用一个很有可能会导致工作的复杂化或重复造轮子。

使用 `ref` 和 `reactive` 之间的区别可以通过编写标准 JavaScript 逻辑的方式进行比较：

```js
// 1. 分离的变量
let x = 0
let y = 0
function updatePosition(e) {
  x = e.pageX
  y = e.pageY
}

// 2. 单一对象
const pos = { x: 0, y: 0 }
function updatePosition(e) {
  pos.x = e.pageX
  pos.y = e.pageY
}
```

- 若使用 `ref` 则在很大程度上是对上面第一种方式更详细的实现，以使基础类型值具有响应性。

- 使用 `reactive` 基本和第二种方式相同，我们只需要使用 `reactvie()` 创建此对象，仅此而已。

### 🔸 沉长的返回语句

### 🔸 更多的灵活性需要更多的自我约束
