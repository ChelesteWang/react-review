通过 redux 源码学习我们知道了，redux 创建的 store 我们可以通过 `store. getState()` 来获取当前状态，通过 `store.dispatch(action)` 来更新状态，通过 `store.subscribe(listener)` 来注册一个当 dispatch 之后会被调用的监听函数。

而 react-redux 其实就是通过 `store.subscribe(listener)` 来把 redux 应用到 react 中的。

## Provider, useStore, useDispatch

在 react 中使用时，我们一般会如下去配置 Provider ，传入一个 store 值，然后把我们的 App 包裹起来，这样在 App 任意组件中我们都可以使用 useStore 和 useDispatch 了。

```js
<Provider store={store}>
  <App />
</Provider>
```

其实通过 useStore 拿到的值就是传给 Provider 的 store ，通过 useDispatch 拿到的值就是 store.dispatch 。

这背后的实现就是用的 react 提供的 context 功能。相关源码如下。

首先创建一个 context ，然后其它地方需要的时候就引用这个 context 即可。

[![2FCD88CDC6C144FCB20924D24A528421](https://user-images.githubusercontent.com/24750971/132233060-d5ad5807-a108-4b1e-b04c-952e0dfa4eec.png)](https://user-images.githubusercontent.com/24750971/132233060-d5ad5807-a108-4b1e-b04c-952e0dfa4eec.png)

在 Provider 里就有使用上面创建的 context 。

[![0B0D41833BDE452DBF75EEAF25F1F026](https://user-images.githubusercontent.com/24750971/132233091-9686f96a-e0f6-4b20-8abd-622dfc5e3cf9.png)](https://user-images.githubusercontent.com/24750971/132233091-9686f96a-e0f6-4b20-8abd-622dfc5e3cf9.png)

下面这个 hook 调用时的返回值就是上面传给 Context.Provider 的 value，也就是 contextValue ，也就是 `{store, subscription}` 。

[![01C81466D0CA4A8B967079BDDDB8A1D9](https://user-images.githubusercontent.com/24750971/132233110-27409675-6ca7-4085-9503-b22ff4cdb9d1.png)](https://user-images.githubusercontent.com/24750971/132233110-27409675-6ca7-4085-9503-b22ff4cdb9d1.png)

所以 useStore 的实现就很简单啦，就是去调用这个 hook 就好了。

[![35355A68B4704575A49FED1B757FB3B3](https://user-images.githubusercontent.com/24750971/132233123-efa14ac1-f39f-4fa2-8c8b-5d3a14b2ba0d.png)](https://user-images.githubusercontent.com/24750971/132233123-efa14ac1-f39f-4fa2-8c8b-5d3a14b2ba0d.png)

而 useDispatch 的返回值就是 store.dispatch 。

[![401AC893D7214AD5AF7CD4C1905040AF](https://user-images.githubusercontent.com/24750971/132233142-fd3d62b4-b932-4f5a-95f6-01b28a94b8ec.png)](https://user-images.githubusercontent.com/24750971/132233142-fd3d62b4-b932-4f5a-95f6-01b28a94b8ec.png)

所以简单点理解，Provider, useStore, useDispatch 就是使用的 react 的 context 来存储 store 和获取 store 。

涉及到需要全局共享数据的地方都会很容易见到 context 的身影，比如 antd 的 ConfigProvider 。

## useSelector

首先要去看看好一下 [Subscription](https://github.com/reduxjs/react-redux/blob/v7.2.4/src/utils/Subscription.js) 这个类的实现，在 Provider 和 useSelector 中都有用到。我把源码加了一些注释，如下。

```js
import { getBatch } from './batch'

// encapsulates the subscription logic for connecting a component to the redux store, as
// well as nesting subscriptions of descendant components, so that we can ensure the
// ancestor components re-render before descendants

const nullListeners = { notify() {} }

function createListenerCollection() {
  const batch = getBatch()
  let first = null
  let last = null

  return {
    clear() {
      first = null
      last = null
    },

    // 执行所有注册的监听器
    // 注意这里用到了 batch ，而这个 batch 默认会被设为 react 的 unstable_batchedUpdates ，看起来是用来做优化的，react 的源码看不懂 =.=
    notify() {
      batch(() => {
        let listener = first
        while (listener) {
          listener.callback()
          listener = listener.next
        }
      })
    },

    get() {
      let listeners = []
      let listener = first
      while (listener) {
        listeners.push(listener)
        listener = listener.next
      }
      return listeners
    },

    // 注册一个监听器，内部实现是用的双向链表，相对于单向链表来说会更容易去删除一个节点
    subscribe(callback) {
      let isSubscribed = true

      let listener = (last = {
        callback,
        next: null,
        prev: last,
      })

      if (listener.prev) {
        listener.prev.next = listener
      } else {
        first = listener
      }

      return function unsubscribe() {
        if (!isSubscribed || first === null) return
        isSubscribed = false

        if (listener.next) {
          listener.next.prev = listener.prev
        } else {
          last = listener.prev
        }
        if (listener.prev) {
          listener.prev.next = listener.next
        } else {
          first = listener.next
        }
      }
    },
  }
}

export default class Subscription {
  constructor(store, parentSub) {
    this.store = store
    this.parentSub = parentSub
    this.unsubscribe = null
    this.listeners = nullListeners

    this.handleChangeWrapper = this.handleChangeWrapper.bind(this)
  }

  // 添加一个监听器
  addNestedSub(listener) {
    this.trySubscribe()
    return this.listeners.subscribe(listener)
  }

  // 触发所有的监听器
  notifyNestedSubs() {
    this.listeners.notify()
  }

  handleChangeWrapper() {
    if (this.onStateChange) {
      this.onStateChange()
    }
  }

  isSubscribed() {
    return Boolean(this.unsubscribe)
  }

  // 这里面根据是否有 this.parentSub 执行了不同的代码
  // 在 Provider 中创建的 Subscription 实例不会有 parentSub ，所以这里执行的是 this.store.subscribe(this.handleChangeWrapper) ，而这个 this.store 就是 redux 的 store ，也就是每次 dispatch 后，都会执行 Provider 的 Subscription 实例的 handleChangeWrapper 方法
  // 而在 useSelector 中创建 Subscription 实例时，会传 parentSub ，并且 parentSub 就是 Provider 中创建的 Subscription 实例，所以这里执行的 this.parentSub.addNestedSub(this.handleChangeWrapper) 的效果是把 useSelector 的 Subscription 实例的 handleChangeWrapper 注册到 Provider 的 Subscription 实例的监听器列表中
  trySubscribe() {
    if (!this.unsubscribe) {
      this.unsubscribe = this.parentSub
        ? this.parentSub.addNestedSub(this.handleChangeWrapper)
        : this.store.subscribe(this.handleChangeWrapper)

      this.listeners = createListenerCollection()
    }
  }

  tryUnsubscribe() {
    if (this.unsubscribe) {
      this.unsubscribe()
      this.unsubscribe = null
      this.listeners.clear()
      this.listeners = nullListeners
    }
  }
}
```

下面是 Provider 源码，在 Provider 中创建 Subscription 实例时只传入了 store ，所以在 trySubscribe 时会执行的是 `this.store.subscribe(this.handleChangeWrapper)` 。传入的这个 store 其实就是 redux 的 store ，同时给 onStateChange 赋值为 notifyNestedSubs 了。那么当 dispatch 后，就会去执行 this.handleChangeWrapper ，也就是 onStateChange ， 也就是 notifyNestedSubs 了。这里 notifyNestedSubs 实际上执行的其实是所有 useSelector 中的 checkForUpdates ，再往下面会看到。

[![39462AFEC44C4DFFB349DBFAFF6B218E](https://user-images.githubusercontent.com/24750971/132233200-0e6c9925-d688-452a-aaa3-bf92d5d64d1f.png)](https://user-images.githubusercontent.com/24750971/132233200-0e6c9925-d688-452a-aaa3-bf92d5d64d1f.png)

对于 useSelector 来说，是把 store 和 subscription 都传给了 useSelectorWithStoreAndSubscription ，然后创建 Subscription 实例时是有 contextSub 的，所以 trySubscribe 时会执行的是 `this.parentSub.addNestedSub(this.handleChangeWrapper)` ，this.parentSub 就是 Provider 的 Subscription 实例，那也就是把 this.handleChangeWrapper 注册到 Provider 的 Subscription 实例的 listeners 中了。

[![2A94F87344F148B48D2C6BE5FAEC1F40](https://user-images.githubusercontent.com/24750971/132233235-62e2b8a0-269a-4b45-bba7-6084142d5908.png)](https://user-images.githubusercontent.com/24750971/132233235-62e2b8a0-269a-4b45-bba7-6084142d5908.png)

[![00C60AD72AF345F78D15FB2316F7ADA2](https://user-images.githubusercontent.com/24750971/132233249-48d68145-2c60-42a8-b84f-d82558baf2a0.png)](https://user-images.githubusercontent.com/24750971/132233249-48d68145-2c60-42a8-b84f-d82558baf2a0.png)

[![2E04A9F2D1254F9BA03CC52163E90250](https://user-images.githubusercontent.com/24750971/132233271-9d655122-acc1-44e2-bace-6baa871d6e9c.png)](https://user-images.githubusercontent.com/24750971/132233271-9d655122-acc1-44e2-bace-6baa871d6e9c.png)

在 useSelector 中 onStateChange 被赋值为 checkForUpdates 了，这个函数是在 dispatch 后，Provider 的 Subscription 实例执行所有注册的 listeners 然后就会执行到 checkForUpdates 。

以上就是 Provider 和 useSelector 结合 store.subscribe 的一个大概工作流程。

## forceRender

在 useSelector 中我们可以看到这样一行代码： `const [, forceRender] = useReducer((s) => s + 1, 0)` 。还是挺有意思的，当你希望自定义 hook 重新执行时，就调用一下 forceRender 。

## useIsomorphicLayoutEffect

在 Provider 和 useSelector 中我们都有看到 useIsomorphicLayoutEffect ，它其实就是浏览器环境就等同于 useLayoutEffect ，服务端就等同于 useEffect 。相关源码如下。

```js
import { useEffect, useLayoutEffect } from 'react'

// React currently throws a warning when using useLayoutEffect on the server.
// To get around it, we can conditionally useEffect on the server (no-op) and
// useLayoutEffect in the browser. We need useLayoutEffect to ensure the store
// subscription callback always has the selector from the latest render commit
// available, otherwise a store update may happen between render and the effect,
// which may cause missed updates; we also must ensure the store subscription
// is created synchronously, otherwise a store update may occur before the
// subscription is created and an inconsistent state may be observed

export const useIsomorphicLayoutEffect =
  typeof window !== 'undefined' &&
  typeof window.document !== 'undefined' &&
  typeof window.document.createElement !== 'undefined'
    ? useLayoutEffect
    : useEffect
```

为什么要去尽量使用 useLayoutEffect 呢？ useLayoutEffect 和 useEffect 有哪些区别呢？

[Hooks API Reference – React](https://reactjs.org/docs/hooks-reference.html#uselayouteffect) 文档中有写：

> useLayoutEffect: The signature is identical to useEffect, but it fires synchronously after all DOM mutations. Use this to read layout from the DOM and synchronously re-render. Updates scheduled inside useLayoutEffect will be flushed synchronously, before the browser has a chance to paint.
> 
> Prefer the standard useEffect when possible to avoid blocking visual updates.

简单点说就是 useLayoutEffect 会比 useEffect 先执行，并且 useLayoutEffect 中代码执行完，包括对 state 的更新执行完之后，才会把 state 变化更新到屏幕上去。

可以去 [这篇文章](https://juejin.cn/post/6844904008402862094) 去更直观感受下区别。

至于说 Provider 和 useSelector 中为什么要去尽量使用 useLayoutEffect ，猜测是某些不太常见的场景会有 bug 所以最后采用了 useLayoutEffect ？但是我没找到具体是啥场景 …

如果要强行解释的话，我只能说使用 useLayoutEffect 可以使得 trySubscribe 更早的执行。比如如果说在 useSelector 中是使用的 useEffect ，那么下面代码就会导致 dispatch 时 useSelector 中 checkForUpdates 还没注册：

```js
useEffect(() => {
  dispatch(xxxx)
}, [])

const xxx = useSelector(xx)
```

[![48A296A5BA7E4A1194C4909DFFD475CA](https://user-images.githubusercontent.com/24750971/132233327-07b75a3d-791b-435b-b5a2-e170f3258389.png)](https://user-images.githubusercontent.com/24750971/132233327-07b75a3d-791b-435b-b5a2-e170f3258389.png)

呃 … 可能以后遇到更复杂的场景会更理解一些这里对 useLayoutEffect 的应用。
