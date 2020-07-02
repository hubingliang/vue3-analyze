# Vue3 Reactivity APIs analyze

Reactivity APIs analyze

## [Reactivity APIs](https://composition-api.vuejs.org/api.html#reactivity-apis)

- reactive
- ref
- computed
- readonly
- watchEffect
- watch

## reactive

响应式转换是“深层的”：会影响对象内部所有嵌套的属性。基于 ES2015 的 Proxy 实现，返回的代理对象不等于原始对象。建议仅使用代理对象而避免依赖原始对象。

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

## 原理部分

响应式原理主要的流程就是在触发响应式数据（ref、reactive）的 getter (Proxy) 时把依赖和数据进行绑定 track（依赖收集），之后在触发对应 setter 时执行所储存的依赖 (trigger)。

> 下面的代码都是源码的简化版 删除了 type 和 复杂的处理边界逻辑

ref 的数据包装逻辑：

```js
export function ref(value) {
  return createRef(value);
}

function createRef(rawValue, shallow = false) {
  // 如果是入参已经是 ref 类型的话直接返回
  if (isRef(rawValue)) {
    return rawValue;
  }
  // 如果入参为对象则交给 reactive 处理
  let value = shallow ? rawValue : convert(rawValue);
  const r = {
    // ref 标识
    __v_isRef: true,
    get value() {
      // 依赖收集 （详情见后文）
      track(r, "get", "value");
      return value;
    },
    set value(newVal) {
      if (hasChanged(toRaw(newVal), rawValue)) {
        // 赋值
        rawValue = newVal;
        value = shallow ? newVal : convert(newVal);
        // 触发依赖（详情见后文）
        trigger(r, "set", "value");
      }
    },
  };
  return r;
}
```

首先是创建响应式对象的函数 createReactiveObject

```js
export function reactive(target) {
  return createReactiveObject(target, proxyHandlers);
}
const createReactiveObject = (target, proxyHandlers) => {
  // 首先判断如果传入的 target 本身就是代理对象则直接返回
  if (target[ReactiveFlags.raw]) {
    return target;
  }
  // 判断如果传入的 target 之前被代理过了 那么返回代理过的对象
  if (hasOwn(target, ReactiveFlags.reactive)) {
    return target[ReactiveFlags.reactive];
  }
  // vue3 内部规定了 allow list，只有符合 "Object,Array,Map,Set,WeakMap,WeakSet" 这些类型才会去代理
  if (!canObserve(target)) {
    return target;
  }
  // 这一步用 Proxy 来包装数据 利用 proxyHandlers 来设置 getter 和 setter
  const observed = new Proxy(target, proxyHandlers);
  // 给原始数据设置 __v_reactive 属性，值为响应式数据
  def(target, "__v_reactive", observed);
  // 返回响应式数据
  return observed;
};
```

vue3 对于原始数据和代理数据的复加属性如下

```js
const rawObj = {
  __v_isReactive: true,
  __v_raw: undefined,
  __v_reactive: reactiveObj,
};
```

```js
const reactiveObj = {
  __v_isReactive: true,
  __v_raw: rawObj,
  __v_reactive: reactiveObj,
};
```

具体 proxyHandlers 处理 getter 和 setter 的逻辑

这里只列举了 get 和 set，源码中还有对 deleteProperty,has,ownKeys 这些操作的处理

那么首先是 get：

```js
export const proxyHandlers = {
  get,
  set,
};
function get(target, key, receiver) {
  // 源码是把 isReadonly 作为一个参数传进来的 因为加入 isReadonly 会使逻辑有点难以看懂 这里是排除掉了 isReadonly 的情况
  const isReadonly = false;
  // 一些获取 vue3 自定义特殊属性的拦截
  if (key === "__v_isReactive") {
    return !isReadonly;
  } else if (key === "__v_readonly") {
    return isReadonly;
  } else if (
    key === "__v_raw" &&
    receiver === (isReadonly ? target.__v_readonly : target.__v_reactive)
  ) {
    return target;
  }
  // 判断 target 是否为数组
  const targetIsArray = isArray(target);
  // 如果 target 为数组且获取的 key 是数组的一些原生方法（'includes', 'indexOf', 'lastIndexOf'）的话，会返回数组的原生方法，并把 this 绑定为原始数据
  if (targetIsArray && hasOwn(arrayInstrumentations, key)) {
    return Reflect.get(arrayInstrumentations, key, receiver);
  }

  // 这里通过 Reflect 拿到 key 的值
  const res = Reflect.get(target, key, receiver);

  // 这里把依赖收集起来（track 是收集依赖的处理函数 分析在后文）
  track(target, "get", key);

  // 判断如果获取的值是 ref 的话这里要解套（官方称为 unwrapping），因为被 ref 处理过的数据真正的值需要 .value 才能获取到
  if (isRef(res)) {
    // ref unwrapping, only for Objects, not for Arrays.
    return targetIsArray ? res : res.value;
  }
  // 如果获取的值为对象 那么在递归用 reactive 处理
  if (isObject(res)) {
    // Convert returned value into a proxy as well. we do the isObject check
    // here to avoid invalid value warning. Also need to lazy access readonly
    // and reactive here to avoid circular dependency.
    return reactive(res);
  }
  // 返回最终 get 目标 key 的值
  return res;
}
```

set:

```js
function set(target, key, value, receiver) {
  // 首先拿到之前的值
  const oldValue = target[key];
  // 这里 toRaw 函数判断 如果 value 为代理数据 则拿到原始数据
  value = toRaw(value);
  // 这里还是对 ref 这种特殊数据类型做了处理
  if (!isArray(target) && isRef(oldValue) && !isRef(value)) {
    oldValue.value = value;
    return true;
  }
  // 判断 key 是否在原数据中存在
  const hadKey = hasOwn(target, key);
  // 获取 set 的返回值
  const result = Reflect.set(target, key, value, receiver);
  // don't trigger if target is something up in the prototype chain of original
  if (target === toRaw(receiver)) {
    if (!hadKey) {
      // 触发 add 操作的依赖
      trigger(target, "add", key, value);
    } else if (hasChanged(value, oldValue)) {
      // 触发 set 操作的依赖
      trigger(target, "set", key, value, oldValue);
    }
  }
  // 返回值
  return result;
}
```

然后是收集依赖和触发依赖的函数

- track:

```js
export function track(target, type, key) {
  // 用 shouldTrack 和 activeEffect 过滤掉不需要收集依赖的情况
  // activeEffect 为当前正在处理的依赖函数 为 undefined 就表示不实在 watch 类api 中触发的 getter
  if (!shouldTrack || activeEffect === undefined) {
    return;
  }
  // 获取数据本以有的依赖 map
  let depsMap = targetMap.get(target);
  // 没有的话初始化依赖 Map
  if (!depsMap) {
    targetMap.set(target, (depsMap = new Map()));
  }
  // 获取这个 key 的依赖
  let dep = depsMap.get(key);
  // 同样的没有就帮他初始化
  if (!dep) {
    depsMap.set(key, (dep = new Set()));
  }
  // 依赖栈增加目标函数
  if (!dep.has(activeEffect)) {
    dep.add(activeEffect);
    activeEffect.deps.push(dep);
  }
}

export function trigger(target, type, key, newValue) {
  // 获取依赖
  const depsMap = targetMap.get(target);
  // 如果没有依赖的话证明这个数据没有被依赖收集
  if (!depsMap) {
    // never been tracked
    return;
  }
  // 区分是计算函数还是单纯执行函数
  const effects = new Set();
  const computedRunners = new Set();
  const add = (effectsToAdd) => {
    if (effectsToAdd) {
      effectsToAdd.forEach((effect) => {
        if (effect !== activeEffect || !shouldTrack) {
          if (effect.options.computed) {
            computedRunners.add(effect);
          } else {
            effects.add(effect);
          }
        }
      });
    }
  };
  // 当 key 存在的时候从依赖树里面拿当前 key 的依赖函数
  // schedule runs for SET | ADD | DELETE
  if (key !== void 0) {
    add(depsMap.get(key));
  }
  // 执行 effect 的配置 scheduler 可以劫持函数执行
  const run = (effect) => {
    if (effect.options.scheduler) {
      effect.options.scheduler(effect);
    } else {
      effect();
    }
  };
  // 全部执行
  // Important: computed effects must be run first so that computed getters
  // can be invalidated before any normal effects that depend on them are run.
  computedRunners.forEach(run);
  effects.forEach(run);
}
```

watchEffect 部分

```js
export function effect(fn, options = {}) {
  // 如果传入的函数已经是处理过的，拿到原函数
  if (isEffect(fn)) {
    fn = fn.raw;
  }
  // 把函数和依赖绑定
  const effect = createReactiveEffect(fn, options);
  // watchEffect 的配置 lazy 默认为 false
  if (!options.lazy) {
    effect();
  }
  return effect;
}

function createReactiveEffect(fn, options) {
  const effect = function reactiveEffect(...args) {
    // 如果没有配置 scheduler 执行 fn 触发依赖 getter
    if (!effect.active) {
      return options.scheduler ? undefined : fn(...args);
    }
    // 如果依赖函数栈存在当前的 effect 则不处理
    if (!effectStack.includes(effect)) {
      // 清空当前 effect 的所有依赖
      cleanup(effect);
      try {
        // 入栈
        trackStack.push(shouldTrack);
        effectStack.push(effect);
        shouldTrack = true;
        activeEffect = effect;
        // 执行
        return fn(...args);
      } finally {
        // 出栈
        effectStack.pop();
        const last = trackStack.pop();
        shouldTrack = last === undefined ? true : last;
        activeEffect = effectStack[effectStack.length - 1];
      }
    }
  };
  // 设置一些底层属性
  effect.id = uid++;
  effect._isEffect = true;
  effect.active = true;
  effect.raw = fn;
  effect.deps = [];
  effect.options = options;
  return effect;
}
```
