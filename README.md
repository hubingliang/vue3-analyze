# Vue3 Reactivity APIs analyze{.text-landing.text-shadow}

Reactivity APIs analyze {.text-intro}

## [Reactivity APIs](https://composition-api.vuejs.org/api.html#reactivity-apis)

- reactive
- ref
- computed
- readonly
- watchEffect
- watch

## reactive

响应式转换是“深层的”：会影响对象内部所有嵌套的属性。基于 ES2015 的 Proxy 实现，返回的代理对象不等于原始对象。建议仅使用代理对象而避免依赖原始对象。

---

Example {.text-subtitle}

```html
<p>Count: {{ data.count }}</p>
```

```js
import { reactive } from "vue";
export default {
  setup() {
    // beforeCreate & Created
    const data = reactive({ count: 0 });
    return {
      data,
    };
  },
};
```

## ref

接受一个参数值并返回一个响应式且可改变的 ref 对象。ref 对象拥有一个指向内部值的单一属性 .value。

如果传入 ref 的是一个对象，将调用 reactive 方法进行深层响应转换。

```html
<p>Count: {{ count }}</p>
```

```js
import { ref } from "vue";
export default {
  setup() {
    const count = ref(0);
    return {
      count,
    };
  },
};
```

## computed

传入一个 getter 函数，返回一个默认不可手动修改的 ref 对象。

如果传入 ref 的是一个对象，将调用 reactive 方法进行深层响应转换。

```html
<p>Count: {{ count }}</p>
<p>plusOne: {{ plusOne }}</p>
```

```js
import { ref, computed } from "vue";
export default {
  setup() {
    const count = ref(0);
    const plusOne = computed(() => count.value + 1);
    return {
      count,
      plusOne,
    };
  },
};
```

## readonly

传入一个对象（响应式或普通）或 ref，返回一个原始对象的只读代理。一个只读的代理是“深层的”，对象内部任何嵌套的属性也都是只读的。

```js
import { reactive, readonly } from "vue";
export default {
  setup() {
    const original = reactive({ count: 0 });
    const copy = readonly(original);
    // original 上的修改会触发 copy 上的侦听
    original.count++;
    // 无法修改 copy 并会被警告
    copy.count++; // warning!
    return {
      original,
      copy,
    };
  },
};
```

## watchEffect

立即执行传入的一个函数，并响应式追踪其依赖，并在其依赖变更时重新运行该函数。

```html
<p>Count: {{ count }}</p>
<button @click="addCount">add</button>
```

```js
import { ref, watchEffect } from "vue";
export default {
  setup() {
    const count = ref(0);
    const addCount = () => count.value++;
    watchEffect(() => console.log("count: ", count.value));
    return {
      count,
      addCount,
    };
  },
};
```

## watch

watch 需要侦听特定的数据源，并在回调函数中执行副作用。默认情况是懒执行的，也就是说仅在侦听的源变更时才执行回调。

对比 watchEffect，watch 允许我们：

- 懒执行副作用；
- 更明确哪些状态的改变会触发侦听器重新运行副作用；
- 访问侦听状态变化前后的值。

```html
<p>Count: {{ count }}</p>
<button @click="addCount">add</button>
```

```js
import { ref, watchEffect } from "vue";
export default {
  setup() {
    const count = ref(0);
    const addCount = () => count.value++;
    watch(count, (count, prevCount) => {
      console.log("count: ", count);
    });
    return {
      count,
      addCount,
    };
  },
};
```

## raw vs reactive

raw

```js
const rawObj = {
  __v_isReactive: true,
  __v_raw: undefined,
  __v_reactive: reactiveObj,
};
```

reactive

```js
const reactiveObj = {
  __v_isReactive: true,
  __v_raw: rawObj,
  __v_reactive: reactiveObj,
};
```

## effect function

```ts
const effectStack: effectFunction[] = []
const trackStack: shouldTrack[] = []
const targetMap: WeakMap<Target, depsMap: Map<Key, dep: Set<effectFunction>>> = new WeakMap();
const effectFunction = {
  id: 0, // effectFunction id
  _isEffect: true,
  active: true,
  raw: Function,
  deps: [dep: Set]
  options: {}
};
```
