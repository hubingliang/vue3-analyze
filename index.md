title: Vue3-alpha 介绍
speaker: hubingliang
url: https://github.com/hubingliang/-vue3-introduce

<slide class="bg-black-blue aligncenter">

# Vue3-alpha 介绍 {.text-landing.text-shadow}

By hubingliang {.text-intro}

<slide class="bg-black-blue aligncenter">

## setup

setup => beforeCreate & Created

<slide class="bg-black-blue aligncenter">

## Lifecycle Hooks

- ~~beforeCreate~~ -> use setup()
- ~~created~~ -> use setup()
- beforeMount -> onBeforeMount
- mounted -> onMounted
- beforeUpdate -> onBeforeUpdate
- updated -> onUpdated
- beforeDestroy -> onBeforeUnmount
- destroyed -> onUnmounted
- errorCaptured -> onErrorCaptured

<slide class="bg-black-blue aligncenter">

![carbon.png](https://i.loli.net/2020/03/01/gf34JuyxEqpHvLS.png)

<slide class="bg-black-blue aligncenter">

## New hooks

- onRenderTracked
- onRenderTriggered

![carbon (1).png](https://i.loli.net/2020/03/01/Qs7l4pqkr5fJU9I.png)

<slide class="bg-black-blue aligncenter">

## 响应式实现

- ref
- reactive
- baseHandlers
- collectionHandlers (Set, Map, WeakMap, WeakSet)
- effect

<slide class="bg-black-blue aligncenter">

## ref

- 除了对象外其他类型的响应式处理
- 对于对象解构赋值时响应丢失的处理
- 类似于 reactive 的超集

<slide class="bg-black-blue aligncenter">

![carbon (3).png](https://i.loli.net/2020/01/18/RxqlH71sNXATe8a.png)

<slide class="bg-black-blue aligncenter">

## customRef

customRef 用于自定义一个 ref，可以显式地控制依赖追踪和触发响应，接受一个工厂函数，两个参数分别是用于追踪的 track 与用于触发响应的 trigger，并返回一个一个带有 get 和 set 属性的对象。

<slide class="bg-black-blue aligncenter">

## shallowRef

创建一个 ref ，将会追踪它的 .value 更改操作，但是并不会对变更后的 .value 做响应式代理转换（即变更不会调用 reactive）
![carbon (1).png](https://i.loli.net/2020/05/25/NAvHsuphzYa2oPL.png)

<slide class="bg-black-blue aligncenter">

## reactive

- 只针对对象做响应式处理 (Proxy)
- 建立响应式与原始数据的双向映射 (WeakMap)
- reactive / template 会把 ref 解套

<slide class="bg-black-blue aligncenter">

Reflect.get(target, key, receiver) vs target[key]

![carbon (4).png](https://i.loli.net/2020/01/18/j1hRvmzaypcOPTX.png)

<slide class="bg-black-blue aligncenter">

## baseHandlers

- 对于原始对象数据，会通过 Proxy 劫持，返回新的响应式数据(代理数据)
- 对于代理数据的任何读写操作，都会通过 Refelct 反射到原始对象上
- 在这个过程中，对于读操作，会执行收集依赖的逻辑。对于写操作，会触发监听函数的逻辑
- collectionHandlers (Set, Map, WeakMap, WeakSet)

<slide class="bg-black-blue aligncenter">

## effect

- 注册监听函数
