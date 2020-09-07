# Dive in React Fiber

`React Fiber`是react 16.0之后默认的Reconciler解决方案，在16.0之前使用的是`Stack Reconciler`，它存在一定的性能问题，于是react提出了react fiber去优化。

## Terms

### 


### work-in-progress tree

React会一直持有两棵Fiber树，一棵是当前在屏幕中渲染的current tree，一棵是未来将会flush屏幕的work-in-progress tree（下面简称wip tree）。这两棵树会根据状态的更新进行身份的转换。

```
current tree => workInProgress tree => current tree
```

## Data struct

Fiber在实现中表现为一棵链表树的数据结构，首先来看一下Fiber Node的定义：

```js
function FiberNode(
  tag: WorkTag,
  pendingProps: mixed,
  key: null | string,
  mode: TypeOfMode,
) {
  // Instance
  this.tag = tag;
  this.key = key;
  this.elementType = null;
  this.type = null;
  this.stateNode = null;

  // Fiber
  this.return = null;
  this.child = null;
  this.sibling = null;
  this.index = 0;

  this.ref = null;

  this.pendingProps = pendingProps;
  this.memoizedProps = null;
  this.updateQueue = null;
  this.memoizedState = null;
  this.dependencies = null;

  this.mode = mode;

  // Effects
  this.flags = NoFlags;
  this.subtreeFlags = NoFlags;
  this.deletions = null;

  this.lanes = NoLanes;
  this.childLanes = NoLanes;

  this.alternate = null;
  // ... 
}
```

* type：fiber的类型，是从组件中复制的
* key：用于判断是否可以复用fiber Node，是从组件中复制过来的
* return：当前fiber需要返回的父节点
* child：当前fiber的子节点
* sibling：当前fiber的兄弟节点
* pendingProps：最新的props
* memoizedProps：保存上一次渲染的props
* updateQueue：需要进行更新的操作队列
* memoizedState：保存上一次渲染的state
* alternate：指向wip树

其中fiber使用return、child和sibling连接成了一棵链表树

 
