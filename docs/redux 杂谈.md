# redux 杂谈

## compose 的实现

```js
export default function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }

  const last = funcs[funcs.length - 1]
  const rest = funcs.slice(0, -1)
  return (...args) => rest.reduceRight((composed, f) => f(composed), last(...args))
}
```
## redux 相关

- react-redux
- redux-thunk
- redux-promise
- redux-saga

## 学习参考

[关于react 中间件 react-sage 和 react-thunk的使用](https://juejin.cn/post/6844903989314584590)
