---
title: React源码阅读(16.13.1) -- render/hydrate
date: 2020-05-19 10:21:16
categories:
- web前端
tags:
- js
- React
---


#### ReactDom.render/ReactDom.hydrate 调用legacyRenderSubtreeIntoContainer方法创建ReactRoot

- render方法返回了一个函数 legacyRenderSubtreeIntoContainer
- hydrate和render的唯一区别是传入的第四个参数的boolean值不一样。即表示在服务端是否尽可能复用节点，提高性能。false表示浏览器不会去复用节点。
``` bash
export function render(
  element: React$Element<any>,   // react根组件(App)
  container: Container,          // 组件挂在的容器
  callback: ?Function,           // 组件渲染之后的回调
) {
  return legacyRenderSubtreeIntoContainer(
    null,                        // render方法渲染的是根组件，没有parentComponent(父组件)
    element,
    container,
    false,                       // 是否复用服务端渲染的内容                   
    callback,
  );
}

export function hydrate(
  element: React$Node,
  container: Container,
  callback: ?Function,
) {
  return legacyRenderSubtreeIntoContainer(
    null,
    element,
    container,
    true,
    callback,
  );
}
```

- 创建ReactRoot/fiberRoot
```bash
function legacyRenderSubtreeIntoContainer(
  parentComponent: ?React$Component<any, any>,
  children: ReactNodeList,
  container: Container,
  forceHydrate: boolean,
  callback: ?Function,
) {
  # 页面初次挂载渲染时，container是没有_reactRootContainer属性的
  let root: RootType = (container._reactRootContainer: any);
  let fiberRoot;
  if (!root) {
    # Initial mount 创建ReactRoot及fiberRoot
    root = container._reactRootContainer = legacyCreateRootFromDOMContainer(
      container,
      forceHydrate,
    );

    fiberRoot = root._internalRoot;
    # 判断是否传入callback，处理callback
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function() {
        // 通过fiberRoot获取公共的Root实例，实际为fiberRoot.current.child.stateNode，并通过该实例去调用callback
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    # 页面初次渲染，需要尽快渲染完成，不使用批量更新
    unbatchedUpdates(() => {
      updateContainer(children, fiberRoot, parentComponent, callback);
    });
  } else {
    fiberRoot = root._internalRoot;
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function() {
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    # Update
    updateContainer(children, fiberRoot, parentComponent, callback);
  }
  return getPublicRootInstance(fiberRoot);
}

function legacyCreateRootFromDOMContainer(
  container: Container,
  forceHydrate: boolean,
): RootType {
  const shouldHydrate =
    forceHydrate || shouldHydrateDueToLegacyHeuristic(container);
  # 不是服务端渲染且是第一次渲染时，移除container的所有子节点.
  if (!shouldHydrate) {
    let warned = false;
    let rootSibling;
    while ((rootSibling = container.lastChild)) {
      container.removeChild(rootSibling);
    }
  }

  return createLegacyRoot(
    container,
    shouldHydrate
      ? {
          hydrate: true,
        }
      : undefined,
  );
}

export function createLegacyRoot(
  container: Container,
  options?: RootOptions,
): RootType {
  return new ReactDOMBlockingRoot(container, LegacyRoot, options);
}

function ReactDOMBlockingRoot(
  container: Container,
  tag: RootTag,
  options: void | RootOptions,
) {
  # 将创建的root挂载到ReactDOMBlockingRoot类的_internalRoot
  this._internalRoot = createRootImpl(container, tag, options);
}

function createRootImpl(
  container: Container,
  tag: RootTag,
  options: void | RootOptions,
) {
  // Tag is either LegacyRoot or Concurrent Root
  const hydrate = options != null && options.hydrate === true;
  const hydrationCallbacks =
    (options != null && options.hydrationOptions) || null;
  const root = createContainer(container, tag, hydrate, hydrationCallbacks);
  markContainerAsRoot(root.current, container);
  const containerNodeType = container.nodeType;

  if (hydrate && tag !== LegacyRoot) {
    const doc =
      containerNodeType === DOCUMENT_NODE ? container : container.ownerDocument;
    // We need to cast this because Flow doesn't work
    // with the hoisted containerNodeType. If we inline
    // it, then Flow doesn't complain. We intentionally
    // hoist it to reduce code-size.
    eagerlyTrapReplayableEvents(container, ((doc: any): Document));
  } else if (
    enableModernEventSystem &&
    containerNodeType !== DOCUMENT_FRAGMENT_NODE &&
    containerNodeType !== DOCUMENT_NODE
  ) {
    ensureListeningTo(container, 'onMouseEnter');
  }
  return root;
}
```

- createContainer调用createFiberRoot创建fiberRoot
```bash
export function createContainer(
  containerInfo: Container,
  tag: RootTag,
  hydrate: boolean,
  hydrationCallbacks: null | SuspenseHydrationCallbacks,
): OpaqueRoot {
  return createFiberRoot(containerInfo, tag, hydrate, hydrationCallbacks);
}
```

- 实例化FiberRootNodem,并创建对应的RootFiber
```bash
export function createFiberRoot(
  containerInfo: any,
  tag: RootTag,
  hydrate: boolean,
  hydrationCallbacks: null | SuspenseHydrationCallbacks,
): FiberRoot {
  const root: FiberRoot = (new FiberRootNode(containerInfo, tag, hydrate): any);
  if (enableSuspenseCallback) {
    root.hydrationCallbacks = hydrationCallbacks;
  }

  // Cyclic construction. This cheats the type system right now because
  // stateNode is any.

  # 创建FiberRoot对应的RootFiber,作为其current属性的值
  const uninitializedFiber = createHostRootFiber(tag);
  root.current = uninitializedFiber;
  uninitializedFiber.stateNode = root;

  initializeUpdateQueue(uninitializedFiber);

  return root;
}
```
- FiberRootNode的数据结构
```bash
function FiberRootNode(containerInfo, tag, hydrate) {
  this.tag = tag;

  # root节点，render方法接收的第二个参数
  this.containerInfo = containerInfo;
  this.pendingChildren = null;

  # Fiber对象，当前应用对应的Fiber对象，是Root Fiber
  this.current = null;                   
  this.pingCache = null;

  # 已经完成的任务的FiberRoot对象，如果你只有一个Root，那他永远只可能是这个Root对应的Fiber，或者是null
  # 在commit阶段只会处理这个值对应的任务
  this.finishedWork = null;

  # 在任务被挂起的时候通过setTimeout设置的返回内容，用来下一次如果有新的任务挂起时清理还没触发的timeout
  this.timeoutHandle = noTimeout;

  # 顶层context对象，只有主动调用`renderSubtreeIntoContainer`时才会有用
  this.context = null;
  this.pendingContext = null;
  this.hydrate = hydrate;
  this.callbackNode = null;
  this.callbackId = NoLanes;
  this.callbackPriority_new = NoLanePriority;
  this.expirationTimes = createLaneMap(NoTimestamp);

  this.pendingLanes = NoLanes;
  this.suspendedLanes = NoLanes;
  this.pingedLanes = NoLanes;
  this.expiredLanes = NoLanes;
  this.mutableReadLanes = NoLanes;

  this.finishedLanes = NoLanes;

  this.entangledLanes = NoLanes;
  this.entanglements = createLaneMap(NoLanes);

  if (enableSchedulerTracing) {
    this.interactionThreadID = unstable_getThreadID();
    this.memoizedInteractions = new Set();
    this.pendingInteractionMap_new = new Map();
  }
  if (enableSuspenseCallback) {
    this.hydrationCallbacks = null;
  }
}
```
- Fiber的数据结构
```bash
function FiberNode(
  tag: WorkTag,
  pendingProps: mixed,
  key: null | string,
  mode: TypeOfMode,
) {
  # 组件的类型
  this.tag = tag;
  this.key = key;

  # ReactElement.type，也就是我们调用`createElement`的第一个参数
  this.elementType = null;
  this.type = null;

  # 当前Fiber相关本地状态
  this.stateNode = null;

  // Fiber
  # 指向他在Fiber节点树中的`parent`，用来在处理完这个节点之后向上返回
  this.return = null;

  # 单链表树结构
  # 指向自己的第一个子节点
  this.child = null;

  # 指向自己的兄弟结构
  # 兄弟节点的return指向同一个父节点
  this.sibling = null;
  this.index = 0;

  this.ref = null;

  # 新的变动带来的新的props
  this.pendingProps = pendingProps;

  # 上一次渲染完成之后的props
  this.memoizedProps = null;

  # 该Fiber对应的组件产生的Update会存放在这个队列里面
  this.updateQueue = null;

  # 上一次渲染的时候的state
  this.memoizedState = null;
  this.dependencies_new = null;

  this.mode = mode;

  # Effects
  # 用来记录Side Effect
  this.effectTag = NoEffect;

  # 单链表用来快速查找下一个side effect
  this.nextEffect = null;

  # 子树中第一个side effect
  this.firstEffect = null;

  # 子树中最后一个side effect
  this.lastEffect = null;

  this.lanes = NoLanes;
  this.childLanes = NoLanes;

  # 在Fiber树更新的过程中，每个Fiber都会有一个跟其对应的Fiber
  # 我们称他为`current <==> workInProgress`
  # 在渲染完成之后他们会交换位置
  this.alternate = null;

  if (enableProfilerTimer) {
    this.actualDuration = Number.NaN;
    this.actualStartTime = Number.NaN;
    this.selfBaseDuration = Number.NaN;
    this.treeBaseDuration = Number.NaN;
    this.actualDuration = 0;
    this.actualStartTime = -1;
    this.selfBaseDuration = 0;
    this.treeBaseDuration = 0;
  }
}
```


- 创建update对象，并挂载到更新对列中，然后进入调度
```bash
export function updateContainer(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  callback: ?Function,
): Lane {
  const current = container.current;
  const eventTime = requestEventTime();
  const suspenseConfig = requestCurrentSuspenseConfig();
  const lane = requestUpdateLane(current, suspenseConfig);

  const context = getContextForSubtree(parentComponent);
  if (container.context === null) {
    container.context = context;
  } else {
    container.pendingContext = context;
  }

  const update = createUpdate(eventTime, lane, suspenseConfig);
  // Caution: React DevTools currently depends on this property
  // being called "element".
  update.payload = {element};

  callback = callback === undefined ? null : callback;
  if (callback !== null) {
    update.callback = callback;
  }

  enqueueUpdate(current, update);
  scheduleUpdateOnFiber(current, lane, eventTime);

  return lane;
}
```