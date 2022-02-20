## layout

实现 layout 组件包含布局内容的组件 props 传入

## List

## modal

## request

API 抽离出来，request 请求封装如 swr ，react-query , ahooks(useRequest), 使用 useEffect 获取数据

## 区分容器组件与视图组件

做到视图组件无副作用，不包含业务状态，只包含视图相关状态，容器组件用于管理状态，常使用 HOC 

## 受控组件与非受控组件

## 组件方法传入

使用具名函数传入组件，防止箭头函数每次传入后都重新初始化，可以配合 useCallback 一起使用
