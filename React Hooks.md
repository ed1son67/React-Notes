# React Hooks

Hooks的优点在于将逻辑和UI进行进一步的解耦，使代码更容易维护。

## useState

useState更新一个state会替换掉整个state的内容，而不是自动合并

## useEffect

useEffect主要是用于处理具有副作用的操作，它接受两个参数，第一个参数是函数，具有副作用操作的代码都会放在里面，第二个参数是一个数组，用于存放数据依赖，当依赖发生变化的时候，会触发钩子的重新执行。

```js
useEffect(() => {
    // effect action
}, [])
``` 

### hooks模拟react生命周期

可以把 useEffect Hook 看做 componentDidMount、componentDidUpdate 和 componentWillUnmount 这三个函数的组合。

#### componentDidMount

组件每一次渲染都会触发effect函数

```js
useEffect(() => {

});
```

#### componentWillUnmout

如果effect函数返回了一个函数，那这个函数会在组件unmount的时候执行。

```js
useEffect(() => {
    return () => {

    }
});
```

#### componentDidUpdate

useEffect的第二个参数是一个数组，数组的内容称为依赖，如果两次渲染后依赖发生了改变，就会重新触发effect函数。

```js
useEffect(() => {
    return () => {

    }
}, []);
```
## useCallback



## 自定义hook

