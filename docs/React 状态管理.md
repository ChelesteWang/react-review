## React Context

Context 需要嵌套 Provider 组件，一旦代码中使用多个 context，将会造成嵌套地狱，组件的可读性和纯粹性会直线降低，从而导致组件重用更加困难。
Context 可能会造成不必要的渲染。一旦 context 里的 value 发生改变，任何引用 context 的子组件都会被更新。

从 context 的解决方案里，其实可以得到一些启发。状态管理的流程可以简化成三个模型： Store（存储所有状态）、Hook （抽象公共逻辑，更改状态）、Component（使用状态的组件）。

订阅更新： 初始化执行 Hook 的时候，需要收集哪些 Component 使用了 Store
感知变更： Hook 中的行为能够改变 Store 的状态，也要能被 Store 所感知到
发布更新： Store 一旦变更，需要驱动所有订阅更新的 Component 更新

1. Proxy 监听状态
2. 传统 Redux
3. Hook 思想

## 贪婪更新与惰性更新

- 如之前提及 MobX 时所说，使用 proxy "监听" 的方案，虽然不够 React，但确实用起来简单，且最符合直觉。
- 本质上来说，React 是一种 "贪婪更新" 的策略，全量 re-render 然后 diff。
- 而 proxy 是一种 "惰性更新" 的策略，可以精准知道是哪个变量更新。所以利用 proxy，可以做一些 re-render 的性能优化。
- 而 React conf 上介绍的 React Forget，代表 React 自身也并不排斥在 "惰性更新" 的思路上做一些优化。
- 注意上面的 "贪婪更新" 和 "惰性更新" 是我自创的词，参考了正则中的贪婪和惰性概念。

参考文章：

https://juejin.cn/post/7063054916234772517

https://juejin.cn/post/7065875157268561957
