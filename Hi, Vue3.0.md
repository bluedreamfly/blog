---
title: Hi, Vue3.0
date: 2020-01-22 09:00
categories: ["技术", "前端", "vue"]
tags: ["vue", "vue3.0"]
---

&emsp;&emsp;Vue3.0已经规划许久了吧，这一次终于来了，虽然还是beta版本，但作为技术爱好者，怎么能不先体验一把呢？当然现在啥文档也没有，是不好入手。不过尤大神已经在多个场合下谈论到了Vue3.0的一些特性，所以不算一无所知，至少大体的技术特性我们是了解的。当然，就算啥也没有，源码是已经有了，还有啥能难倒我们。当然我们今天只是开个头，并不会讲都有哪些特性，然后怎么用，我是想分几个部分，首先是跟Vue2的响应式原理差别较大实现部分，后续的一些像Suspense、Portal、还有静态提升等等这些都会单独拿出来讲，毕竟我也还没看完，囧！一步步来，心急吃不了热豆腐，我们先讲讲reactivity。

&emsp;&emsp;要讲新的实现方式，首先我们当然免不了要跟Vue2的实现方式进行对比，然后带着下面几个问题来学习可能会更好：
- Vue2的实现方式有哪些问题？
- 为什么要通过Proxy来实现，有什么好处？
- 它们之间又有哪些相同点？

&emsp;&emsp;带着上面的这几个问题我们开始我们的主题。

### Vue2的实现方式有哪些问题？

&emsp;&emsp;Vue2是使用`defineProperty`来实现拦截的，但是`defineProperty`只能拦截对象，所以对于数组只能通过hook数组的方法来变相实现拦截，当然还是有很多数组的使用方式的是没法模拟的，比如通过下标对数组某个元素赋值。就是限制特别多，一不小心就写出了更新数据但界面没有相应更新的代码。虽说有这么多毛病，但是只要规避掉这些问题，虽然在使用上存在一些不和谐，但总归还不算难用。但我们能满足于此吗？显然不能，这时候就改Proxy上场了。

### 为什么要通过Proxy来实现，有什么好处？

&emsp;&emsp;其实ES6在2015年已经发布了，但是发布了浏览器支持度很低，所以就算想也不能用Proxy。但现在呢，Proxy使用虽然不算理想，但至少我们可以忽略一些浏览器，比如IE系列，微软新浏览器都用`chromium`了，你还在担心什么。至于为什么用Proxy，除了能解决Vue2的问题，当然在性能上也是有提升的，这里我们也就不细说了。我们来说说它们之间实现响应式有什么相同点。

### 它们之间又有哪些相同点？

&emsp;&emsp;其实嘛，它们之间最本质的相同点就是通过拦截来跟踪收集依赖，等到数据更新，在调用相应的代码。在Vue2当中，依赖数据的是Watcher对象，而在3.0当中则是一个effect对象，当然本质上它们是没有太大区别的。

&emsp;&emsp;说了这么多我们就来看看3.0源码是如何实现的，当然，如果对2.0的响应式原理感兴趣的也可以去看看对应的文章，我这里就不再赘述了。

&emsp;&emsp;首先先看reactivity源码的目录结构：

```
├── __tests__ // 测试
├── src
│   ├── baseHandlers.ts // 非集合的proxy handlers
│   ├── collectionHandlers.ts // 集合的proxy handlers
│   ├── computed.ts // 计算属性
│   ├── effect.ts //相当于2.0的watcher
│   ├── index.ts 
│   ├── lock.ts //
│   ├── operations.ts // 操作枚举
│   ├── reactive.ts // 响应式核心入口
│   └── ref.ts // 这个ref不是我们理解的ref

```
理解响应式的核心只要记住一点，数据都有哪些依赖，只要把依赖这些的数据的使用方存起来，在更新这些数据在做对应的操作就行了。看起来是跟发布跟订阅有点类似的，我对哪些数据感兴趣，然后数据变化通知我，而订阅这个过程就在拦截过程当中产生的，不需要我们主动去订阅。那我们现在具体来分析一下具体的文件，其实核心是`baseHandlers.ts`和`collectionHandlers.ts`，那我们先来看看这两个文件，当然需要你先去了解一下`Proxy`的使用。

### baseHandlers.ts
```javascript
function createGetter(isReadonly = false, shallow = false) {
  return function get(target: object, key: string | symbol, receiver: object) {
    
    const res = Reflect.get(target, key, receiver)
    if (isSymbol(key) && builtInSymbols.has(key)) {
      return res
    }
    // shallow为false，则表示不做深层代理
    if (shallow) {
      // 收集依赖
      track(target, TrackOpTypes.GET, key)
      // TODO strict mode that returns a shallow-readonly version of the value
      return res
    }
    // 如果是ref, 就是res._isRef为true, 则直接返回，因为ref这一层本身自己做了拦截
    if (isRef(res)) {
      return res.value
    }
    // 收集依赖
    track(target, TrackOpTypes.GET, key)
    // 如果返回的是对象，并且shallow为false，则深层代理
    return isObject(res)
      ? isReadonly
        ? // need to lazy access readonly and reactive here to avoid
          // circular dependency
          // 只读
          readonly(res)
        : reactive(res)
      : res
  }
}
```

`createGetter`就是用来创建`proxy handlers`所需要的`get`拦截方法，对对象、数组的属性访问都会先从`get`这个拦截器通过，如果是`Symbol`或者`Symbol`对应的一些内置的属性：比如hasInstance、iterate等等这些是不做处理。这个函数一共有两个参数`isReadonly`和`shallow`，`isReadonly`创建一个只读`Getter`，就是不能对数据进行修改。`shallow`是不做深层代理，因为`Proxy`只做一层拦截。当然其实这些也还都不是最关键的部分，我们前面说过，拦截是为了收集依赖，这个才是重点。
也就是`track(target, TrackOpTypes.GET, key)`, 这个我们现在先不讲。大概代码里面我都做了注释，不过针对`Ref`做特殊处理的原因，我们稍后讲到`Ref`在说。

#### createSetter

```javascript
function createSetter(isReadonly = false, shallow = false) {
  return function set(
    target: object,
    key: string | symbol,
    value: unknown,
    receiver: object
  ): boolean {
    // 如果是只读，则不会操作成功
    if (isReadonly && LOCKED) {
      if (__DEV__) {
        console.warn(
          `Set operation on key "${String(key)}" failed: target is readonly.`,
          target
        )
      }
      return true
    }

    const oldValue = (target as any)[key]
    if (!shallow) {
      value = toRaw(value)
      // 这里对Ref做特殊处理，因为对Ref的赋值本身就会触发effect执行
      // 如果原本是Ref，而现在不是Ref赋值的话就只会更改Ref的value
      if (isRef(oldValue) && !isRef(value)) {
        oldValue.value = value
        return true
      }
    } else {
      // in shallow mode, objects are set as-is regardless of reactive or not
    }

    const hadKey = hasOwn(target, key)
    const result = Reflect.set(target, key, value, receiver)
    // don't trigger if target is something up in the prototype chain of original
    /**
        const proxy = new Proxy({}, {});
        const obj = Object.create(proxy);
        obj.aa = 'gggg' target跟receiver就不一致，这种是不会更新的
    **/

    if (target === toRaw(receiver)) {
      /* istanbul ignore else */
      if (__DEV__) {
        const extraInfo = { oldValue, newValue: value }
        if (!hadKey) {
          trigger(target, TriggerOpTypes.ADD, key, extraInfo)
        } else if (hasChanged(value, oldValue)) {
          trigger(target, TriggerOpTypes.SET, key, extraInfo)
        }
      } else {
        if (!hadKey) {
            // 如果原来对象没有这个属性的，则type则为ADD，可以根据type做一些区分的事情
          trigger(target, TriggerOpTypes.ADD, key)
        } else if (hasChanged(value, oldValue)) {
            // 如果原来对象有这个属性的，则type则为SET
          trigger(target, TriggerOpTypes.SET, key)
        }
      }
    }
    return result
  }
}

```
这个文件里面的其他代码逻辑差不多就是类似的，大家自行理解，接下来我们讲讲`collectionHandlers.ts`

### collectionHandlers.ts

```javascript

```










