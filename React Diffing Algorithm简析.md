# React Diffing Algorithm简析

## Stack Reconciler

Stack Reconciler是React 16之前的实现，其中更新子节点的部分就使用了著名的diff算法，让我们看看facebook是如何实现diff算法的。

首先会在每一个`DOMComponent`和`CompositeComponent`上实现一个`receive(nextElement)`方法：

```js
class CompositeComponent {
  // ...
  receive(nextElement) {
    // ...
  }
}

class DOMComponent {
  // ...
  receive(nextElement) {
    // ...
  }
}
```

### Updating Composite Components

当一个`CompositeComponents`收到了一个新的`element` ，首先会调用`ComponentsWillUpdate`方法，然后会使用新的props去re-render整个组件并且获得最新渲染出来的element。

```js
class CompositeComponent {
  // ...
  receive(nextElement) {
    var prevProps = this.currentElement.props;
    var publicInstance = this.publicInstance;
    var prevRenderedComponent = this.renderedComponent;
    var prevRenderedElement = prevRenderedComponent.currentElement;

    // Update *own* element
    this.currentElement = nextElement;
    var type = nextElement.type;
    var nextProps = nextElement.props;

    // Figure out what the next render() output is
    var nextRenderedElement;
    if (isClass(type)) {
      // Component class
      // Call the lifecycle if necessary
      if (publicInstance.componentWillUpdate) {
        publicInstance.componentWillUpdate(nextProps);
      }
      // Update the props
      publicInstance.props = nextProps;
      // Re-render
      nextRenderedElement = publicInstance.render();
    } else if (typeof type === 'function') {
      // Component function
      nextRenderedElement = type(nextProps);
    }
    // ...
```

然后判断最新渲染的nextRenderedElement和最后一次render的element的type是否有所改变。如果没有改变，将会调用`prevRenderedComponent.receive(nextRenderedElement)`方法去进行更深入的比较。

```js
    // If the rendered element type has not changed,
    // reuse the existing component instance and exit.
    if (prevRenderedElement.type === nextRenderedElement.type) {
      prevRenderedComponent.receive(nextRenderedElement);
      return;
    }
```
如果type改变了，react会认为这是两个完全不同的节点，不会深入进行更新操作。而是会unmount现在存在的实例和mount最新的element。

```js
    // ...

    // If we reached this point, we need to unmount the previously
    // mounted component, mount the new one, and swap their nodes.

    // Find the old node because it will need to be replaced
    var prevNode = prevRenderedComponent.getHostNode();

    // Unmount the old child and mount a new child
    prevRenderedComponent.unmount();
    var nextRenderedComponent = instantiateComponent(nextRenderedElement);
    var nextNode = nextRenderedComponent.mount();

    // Replace the reference to the child
    this.renderedComponent = nextRenderedComponent;

    // Replace the old node with the new one
    // Note: this is renderer-specific code and
    // ideally should live outside of CompositeComponent:
    prevNode.parentNode.replaceChild(nextNode, prevNode);
  }
}
```

除了上述条件外，当key不同时也会导致出现上述直接进入re-render的情况。


### Updating Host Components

像`DomComponent`这一类的`Host Components`对于更新的实现又有所不同，因为当它们拿到一个element的时候，需要去更新不同平台下的视图，假如是`React DOM`，需要更新DOM的属性。

```js
class DOMComponent {
  // ...

  receive(nextElement) {
    var node = this.node;
    var prevElement = this.currentElement;
    var prevProps = prevElement.props;
    var nextProps = nextElement.props;    
    this.currentElement = nextElement;

    // Remove old attributes.
    Object.keys(prevProps).forEach(propName => {
      if (propName !== 'children' && !nextProps.hasOwnProperty(propName)) {
        node.removeAttribute(propName);
      }
    });
    // Set next attributes.
    Object.keys(nextProps).forEach(propName => {
      if (propName !== 'children') {
        node.setAttribute(propName, nextProps[propName]);
      }
    });

    // ...
```

然后需要更新它的子元素，简单来讲，会通过type和key去计算和跟踪更新子节点需要进行的插入和删除操作，同时会用一个数组来存放这些操作。

```js
    // ...

    // These are arrays of React elements:
    var prevChildren = prevProps.children || [];
    if (!Array.isArray(prevChildren)) {
      prevChildren = [prevChildren];
    }
    var nextChildren = nextProps.children || [];
    if (!Array.isArray(nextChildren)) {
      nextChildren = [nextChildren];
    }
    // These are arrays of internal instances:
    var prevRenderedChildren = this.renderedChildren;
    var nextRenderedChildren = [];

    // As we iterate over children, we will add operations to the array.
    var operationQueue = [];

    // Note: the section below is extremely simplified!
    // It doesn't handle reorders, children with holes, or keys.
    // It only exists to illustrate the overall flow, not the specifics.

    for (var i = 0; i < nextChildren.length; i++) {
      // Try to get an existing internal instance for this child
      var prevChild = prevRenderedChildren[i];

      // If there is no internal instance under this index,
      // a child has been appended to the end. Create a new
      // internal instance, mount it, and use its node.
      if (!prevChild) {
        var nextChild = instantiateComponent(nextChildren[i]);
        var node = nextChild.mount();

        // Record that we need to append a node
        operationQueue.push({type: 'ADD', node});
        nextRenderedChildren.push(nextChild);
        continue;
      }

      // We can only update the instance if its element's type matches.
      // For example, <Button size="small" /> can be updated to
      // <Button size="large" /> but not to an <App />.
      var canUpdate = prevChildren[i].type === nextChildren[i].type;

      // If we can't update an existing instance, we have to unmount it
      // and mount a new one instead of it.
      if (!canUpdate) {
        var prevNode = prevChild.getHostNode();
        prevChild.unmount();

        var nextChild = instantiateComponent(nextChildren[i]);
        var nextNode = nextChild.mount();

        // Record that we need to swap the nodes
        operationQueue.push({type: 'REPLACE', prevNode, nextNode});
        nextRenderedChildren.push(nextChild);
        continue;
      }

      // If we can update an existing internal instance,
      // just let it receive the next element and handle its own update.
      prevChild.receive(nextChildren[i]);
      nextRenderedChildren.push(prevChild);
    }

    // Finally, unmount any children that don't exist:
    for (var j = nextChildren.length; j < prevChildren.length; j++) {
      var prevChild = prevRenderedChildren[j];
      var node = prevChild.getHostNode();
      prevChild.unmount();

      // Record that we need to remove the node
      operationQueue.push({type: 'REMOVE', node});
    }

    // Point the list of rendered children to the updated version.
    this.renderedChildren = nextRenderedChildren;

    // ...
```

最后会执行操作队列内的所有DOM操作，实际上会比以下的代码要更复杂，因为涉及到移动节点的操作。

```js
    // ...

    // Process the operation queue.
    while (operationQueue.length > 0) {
      var operation = operationQueue.shift();
      switch (operation.type) {
      case 'ADD':
        this.node.appendChild(operation.node);
        break;
      case 'REPLACE':
        this.node.replaceChild(operation.nextNode, operation.prevNode);
        break;
      case 'REMOVE':
        this.node.removeChild(operation.node);
        break;
      }
    }
  }
}
```


### 不足之处

`Stack reconciler`有着先天的缺陷：更新操作需要同步执行，而且不可以终止更新的过程以及不能将任务拆分为更小的粒度执行，这就说明将会占用线程较长的时间，基于这些原因，react16使用了`Fiber reconciler`替代了`Stack reconciler`。

## Fiber reconciler

`Fiber reconciler`是react团队为了解决`Stack reconciler`的缺点而提出的解决方案，其中最关键的功能是*增量更新*，可以将一次更新分割为更小的粒度并在多个帧内完成这一次更新。

其他关键性的功能包括：暂停、取消或者重用更新的过程、根据更新类型不同进行优先级的调度，以及新的并发原语。

Fiber尽量保持了react对于diff算法的实现，比如当两个元素的type完全不同时，react会认为这是两棵完全不同的树从而不会继续深入进行diff，又比如diff需要使用key进行比较。

### Fiber是什么

* 在UI中，没有必要立即执行每次更新，实际上这样会造成性能浪费导致掉帧影响用户体验。
* 不同的类型的更新有不同的优先级，比如动画更新的优先级会更高
* 基于push模型的实现需要开发者自行去决定如何调度任务，而基于pull模型的实现允许框架为开发者完成这些工作。

通常，每一个函数被调用的时候会在调用栈中生成一个栈帧。而fiber是专门为了react组件对调用栈的重新实现。每一个fiber都是一个虚拟栈帧，这样的好处是可以将栈帧保存在内存中并且可以随心所欲地调用它。

从结构上看，每一个fiber都是一个包含了react元素的一些信息的js对象，每一个fiber相当于一个栈帧，同时又相当于一个组件的实例。fiber代表了一个react元素的work，比如composite component的work就是render、还有一些生命周期的方法，而host component，比如dom节点，它们的work就是各种操作dom的方法，而fiber提供了追踪、暂停、取消这些work的途径。

当一个react元素第一次转换为fiber节点的时候，react会用元素的data中去创建一个fiber。react首次render产生的fiber节点会组成一棵fiber树，这棵树叫current树，树的结构和react元素树的结构是一致的。当react的状态开始更新的时候，会生成一棵workInProgress树，这棵树代表了未来的状态，当react更新的时候，会遍历current树，为每一个存在的fiber节点根据render方法最新output去创建一个替代节点，这些节点会组成workInProgress树，当更新完成的时候，react会使用workInProgress树去flush屏幕，render完成后workInProgress树会变为current树。



fiber节点之间使用链表进行链接，每个节点都有三个属性：child、sibling、return。

#### Fiber的关键概念

`pendingProps`和`memoizedProps`是一个函数的参数，前者在被执行的时候设置，后者会在最后被设置，当两者相同的时候，react会认为这个fiber之前的输出可以被重用从而避免重复执行。


`pendingWorkPriority`是一个用来表示任务优先级的数字。

fiber拥有几种状态：
* flush：将一个fiber的输出渲染到屏幕上
* work-in-progress：一个还没完成的fiber

在任何时候，一个组件的实例里最多只有两个交替的fiber，

`output`：一个fiber的输出是一个函数的返回值，每一个fiber都有output，但只有叶子节点才会创建output，也就是`host component`，output最后会最后传递到renderer，由它根据output去更新视图。







