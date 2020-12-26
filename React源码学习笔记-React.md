## React 源码学习笔记-React

React这个package定义了所有顶层的API，也就是我们平时使用的APi，其中所有的方法都被挂在React这个namespace下面，比如React.Component。

### React.js

React.js是package的总入口，收敛了所有的方法并export出去。

### ReactElement.js

JSX语法会被Babel编译为React.createElement，所以使用了JSX就一定要引入React。

React.createElement方法返回一个工厂函数ReactElement，ReactElement的作用是生成一个ReactElement对象。

```js
const ReactElement = function(type, key, ref, self, source, owner, props) {
    const element = {
        $$typeof: REACT_ELEMENT_TYPE,
        type: type,
        key: key,
        ref: ref,
        props: props,
        _owner: owner,
    };

    return element;
}
```

### ReactBaseClasses.js

ReactBaseClasses.js定义了React Component的相关API。

```js
function Component(props, context, updater) {
  this.props = props;
  this.context = context;
  // If a component has string refs, we will assign a different object later.
  this.refs = emptyObject;
  // We initialize the default updater but the real one gets injected by the
  // renderer，不同平台有不同renderer，但他们要实现同样的api，也就ReactNoopUpdateQueue的定义
  this.updater = updater || ReactNoopUpdateQueue;
}
// 用于判断是classComponent还是functionComponent，因为instanceof都等于Function
Component.prototype.isReactComponent = {};
// setState 的定义
Component.prototype.setState = function(partialState, callback) {
  // setState只是简单做了一些判断工作，然后推进更新队列里面
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
};
// forceUpdate 的定义
Component.prototype.forceUpdate = function(callback) {
  this.updater.enqueueForceUpdate(this, callback, 'forceUpdate');
};
```

PureComponent利用ComponentDummy实现了对Component的继承，然后在其他地方实现了scu的浅对比

```js
function ComponentDummy() {}
ComponentDummy.prototype = Component.prototype;

function PureComponent(props, context, updater) {
  this.props = props;
  this.context = context;
  // If a component has string refs, we will assign a different object later.
  this.refs = emptyObject;
  this.updater = updater || ReactNoopUpdateQueue;
}

const pureComponentPrototype = (PureComponent.prototype = new ComponentDummy());
pureComponentPrototype.constructor = PureComponent;
// Avoid an extra prototype jump for these methods.
Object.assign(pureComponentPrototype, Component.prototype);
pureComponentPrototype.isPureReactComponent = true;

```

### ReactCreateRef.js

可以看到，ref的定义很简单，就是一个拥有current属性的对象。

```js
export function createRef(): RefObject {
  const refObject = {
    current: null,
  };
  return refObject;
}
```

### ReactForwardRef.js

由于函数组件没有实例，所以不能在内部使用ref调用DOM，要用forwardRef实现Refs转发。

```js
export function forwardRef<Props, ElementType: React$ElementType>(
  render: (props: Props, ref: React$Ref<ElementType>) => React$Node,
) {
    const elementType = {
        $$typeof: REACT_FORWARD_REF_TYPE,
        render,
    };
    return elementType;
}
```

来看一个例子，当ref挂载完成，ref.current 将指向 <button> DOM 节点。

```js
const FancyButton = React.forwardRef((props, ref) => (
  <button ref={ref} className="FancyButton">
    {props.children}
  </button>
));

// 你可以直接获取 DOM button 的 ref：
const ref = React.createRef();
<FancyButton ref={ref}>Click me!</FancyButton>;
```

### ReactChildren.js

这个文件定义了React.Children的所有工具方法。

```js
export {
    forEachChildren as forEach,
    mapChildren as map,
    countChildren as count,
    onlyChild as only,
    toArray,
};
```

这几个方法的核心方法都是mapChildren方法，它是这样定义的：

```js
function mapChildren(
  children: ?ReactNodeList,
  func: MapFunc,
  context: mixed,
): ?Array<React$Node> {
  if (children == null) {
    return children;
  }
  const result = [];
  let count = 0;
  mapIntoArray(children, result, '', '', function(child) {
    return func.call(context, child, count++);
  });
  return result;
}
```

可以看到，调用了mapIntoArray方法返回了result，mapIntoArray的作用是递归遍历子节点，调用map函数并将children数组拍平。

### ReactHooks.js

hooks的API定义比较简单：

```js
export function useState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  const dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}
```

基本上就是调用resolveDispatcher方法获取dispatcher，然后调用dispatcher上相对应的hooks。dispatcher是react-reconciler中import进来的。

