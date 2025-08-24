# Render


基于react-native 0.67.3，其依赖的react-native-renderer版本为18.0.0-experimental-568dc3532，代码位于https://github.com/facebook/react



## render函数
ReactNativeRenderer

入口函数为render函数

```
function render(
  element: Element<ElementType>,
  containerTag: number,
  callback: ?() => void,
): ?ElementRef<ElementType>
```

renderApplication.js会调用该函数，传入的element为AppContainer(子元素为传入的component)以及containerTag(native RCTRootView的tag)

- 从roots中获取对应tag的container赋值给root
- 没有缓存，则调用createContainer创建container
- updateContainer调用，传入element和root
- 调用getPublicRootInstance进行return

createContainer调用createFiberRoot



createFiberRoot
- 根据tag创建FiberRootNode
- 调用createHostRootFiber根据tag创建FiberNode，这里最终调用的createFiber函数
- Fiber赋值给root.current，root node 赋值给fiber.stateNode

FiberNode
```
function FiberNode(tag, pendingProps, key, mode) {
  // Instance
  this.tag = tag;
  this.key = key;
  this.elementType = null;
  this.type = null;
  this.stateNode = null; // 对应native节点

  this.return = null; // 父节点
  this.child = null;
  this.sibling = null;
  this.index = 0;
  this.ref = null;
  this.pendingProps = pendingProps; // createFiber时传入的属性
  this.memoizedProps = null; // performUnitOfWork之后，把pendingProps赋值给memoizedProps
  this.updateQueue = null; // state处理队列
  this.memoizedState = null; // processUpdateQueue处理，把newState赋值给memoizedState
  this.dependencies = null;
  this.mode = mode; // Effects 
  // fiber mode 包括ConcurrentMode、ProfileMode、DebugTracingMode、StrictLegacyMode

  this.flags = NoFlags;
  this.subtreeFlags = NoFlags;
  this.deletions = null;
  this.lanes = NoLanes;
  this.childLanes = NoLanes;
  this.alternate = null;
}
```

updateQueue的初始化
```
function initializeUpdateQueue(fiber) {
  var queue = {
    baseState: fiber.memoizedState,
    firstBaseUpdate: null, // 头指针
    lastBaseUpdate: null, // 尾指针
    shared: {
      pending: null,
      interleaved: null,
      lanes: NoLanes
    },
    effects: null
  };
  fiber.updateQueue = queue;
}
```

```
function createUpdate(eventTime, lane) {
  var update = {
    eventTime: eventTime,
    lane: lane,
    tag: UpdateState,
    payload: null,
    callback: null,
    next: null
  };
  return update;
}
```


FiberRootNode
```
function FiberRootNode(containerInfo, tag, hydrate) {
  this.tag = tag;
  this.containerInfo = containerInfo;
  this.pendingChildren = null;
  this.current = null;
  this.pingCache = null;
  this.finishedWork = null;
  this.timeoutHandle = noTimeout;
  this.context = null;
  this.pendingContext = null;
  this.hydrate = hydrate;
  this.callbackNode = null;
  this.callbackPriority = NoLane;
  this.eventTimes = createLaneMap(NoLanes);
  this.expirationTimes = createLaneMap(NoTimestamp);
  this.pendingLanes = NoLanes;
  this.suspendedLanes = NoLanes;
  this.pingedLanes = NoLanes;
  this.expiredLanes = NoLanes;
  this.mutableReadLanes = NoLanes;
  this.finishedLanes = NoLanes;
  this.entangledLanes = NoLanes;
  this.entanglements = createLaneMap(NoLanes);
}
```

updateContainer的流程
- requestUpdateLane
- createUpdate
- enqueueUpdate，传入container.current（即fiber）和update，通过fiber找到对应的updateQueue，把当前的update赋值给updateQueue的pending.next，以排队执行
- scheduleUpdateOnFiber，传入fiber
  - markUpdateLaneFromFiberToRoot，根据fiber找到root（FiberRootNode）
  - markRootUpdated
  - SyncLane
    - schedulePendingInteractions
  - 非SyncLane
    - ensureRootIsScheduled，传入root
    - schedulePendingInteractions
- entangleTransitions



Lane，一共有31个lane，值从2的0次方，到2的30次方。



## react-reconciler

react-reconciler的层级

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*ofQJS_Pp3g9rtzO7OQJ3dQ.png)

React Component API:提供component API和生命周期
react-reconciler，是管理声明式UI后的命令式更新的核心diff算法。reconciler就是通常所说的虚拟DOM背后的算法
React Renderer：用来实现react-reconciler定义的函数。


[react-reconciler readme](https://github.com/facebook/react/tree/17.0.2/packages/react-reconciler)

[React is also the LLVM for creating declarative UI frameworks](https://agent-hunt.medium.com/react-is-also-the-llvm-for-creating-declarative-ui-frameworks-767e75ce1d6a)







fiber

Fiber是reconciler的重要组成部分，[React Fiber Architecture](https://github.com/acdlite/react-fiber-architecture)

设计上，Fiber考虑了几个能力
- 能够暂停工作，并且之后恢复
- 为不同类型的工作分配优先级
- 复用以前完成的工作
- 中止不需要的工作

为了实现上述目标，需要把工作分解成单元，一个Fiber就相当于一个工作单元。

UI处理一次性不能处理太多任务，否则会掉帧、卡顿。实现以下API有助于解决问题，
requestIdleCallback，在空闲期间处理低优先级任务
requestAnimationFrame，在下一个个动画帧处理高优先级任务

一个fiber对应一个组件。
fiber包含child、sibling、return，子节点（单链表），兄弟节点，父节点。
当前fiber的alternate是work-in-progress，work-in-progress的alternate是当前fiber，一个组件实例会对应当前fiber、以及work-in-progress fiber。

具体逻辑见createWorkInProgress
```
workInProgress.alternate = current;
current.alternate = workInProgress;
```




## workLoop
### renderRootSync

核心逻辑

```
do {
  try {
    workLoopSync();
    break;
  } catch (thrownValue) {
    handleError(root, thrownValue);
  }
} while (true);
```

在workLoopSync之前，会调用prepareFreshStack

prepareFreshStack
- root赋值给workInProgressRoot
- 根据root.current，调用createWorkInProgress函数，创建workInProgress

createWorkInProgress逻辑
- 如果current.alternate存在，则使用其作为workInProgress
- 如果current.alternate不存在，则调用createFiber创建fiber，赋值给workInProgress

### workLoopSync

```
function workLoopSync() {
  // Already timed out, so perform work without checking if we need to yield.
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}
```
通过loop不断循环，从performUnitOfWork不断取出next的workInProgress，进行loop任务

而并发的work loop与sync的区别是判断shouldYield为false才执行
```

function workLoopConcurrent() {
  // Perform work until Scheduler asks us to yield
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

shouldYield来自sceduler库，根据帧刷新来进行任务
```
exports.unstable_shouldYield = function () {
  return exports.unstable_now() >= deadline;
};

var performWorkUntilDeadline = function () {
  deadline = currentTime + yieldInterval;
}
if (fps > 0) {
  yieldInterval = Math.floor(1000 / fps);
} else {
  // reset the framerate
  yieldInterval = 5;
}


var channel = new MessageChannel();
var port = channel.port2;
channel.port1.onmessage = performWorkUntilDeadline;
```


### scheduleWorkOnFiber

```
function scheduleWorkOnFiber(fiber, renderLanes) {
  fiber.lanes = mergeLanes(fiber.lanes, renderLanes); // 处理fiber的lanes
  var alternate = fiber.alternate;

  if (alternate !== null) {
    alternate.lanes = mergeLanes(alternate.lanes, renderLanes);
  }

  scheduleWorkOnParentPath(fiber.return, renderLanes); // 往父节点一路往上处理对应的childLanes
}
```


## work单元


### performUnitOfWork
- 通过传入的unitOfWork，调用setCurrentFiber
- 调用beginWork传入current（unitOfWork.alternate）、unitWork，之后会返回next
- next赋值给workInProgress
- 如果没有next，则调用completeUnitOfWork取next

completeUnitOfWork
- 取sibling赋值给workInProgress进行返回
- unitOfWork的return返回父节点进行任务



### completeWork

组件类型
```
var FunctionComponent = 0;
var ClassComponent = 1;
var IndeterminateComponent = 2; // Before we know whether it is function or class

var HostRoot = 3; // Root of a host tree. Could be nested inside another node.

var HostPortal = 4; // A subtree. Could be an entry point to a different renderer.

var HostComponent = 5;
var HostText = 6;
var Fragment = 7;
var Mode = 8;
var ContextConsumer = 9;
var ContextProvider = 10;
var ForwardRef = 11;
var Profiler = 12;
var SuspenseComponent = 13;
var MemoComponent = 14;
var SimpleMemoComponent = 15;
var LazyComponent = 16;
var IncompleteClassComponent = 17;
var DehydratedFragment = 18;
var SuspenseListComponent = 19;
var ScopeComponent = 21;
var OffscreenComponent = 22;
var LegacyHiddenComponent = 23;
var CacheComponent = 24;
```











### processUpdateQueue

- 循环内判断!isSubsetOfLanes(renderLanes, updateLane)，为false的情况下，则先不处理任务
  - 构造一个clone对象，lane为updateLane，加入队列newLastBaseUpdate = newLastBaseUpdate.next = clone;
  - 通过newLanes = mergeLanes(newLanes, updateLane)，得到newLanes赋值给workInProgress.lanes
- 如果!isSubsetOfLanes(renderLanes, updateLane)为true，
  - 同样构造一个clone对象，不同的是lane为NoneLane，NoneLane是所有lane的子集，所以上面子集的检查是不会跳过的，
  - 则调用getStateFromUpdate获取到newState，然后queue.baseState = newBaseState;




### finishClassComponent

这里是调用到组件render方法的地方

finishClassComponent的实现，这里会调用setIsRendering为true，并且调用instance.render()，完成之后再次调用setIsRendering为false

finishClassComponent为updateClassComponent所调用

整个调用链路可以描述为

runApplication->Runnable的run函数->renderApplication->ReactNativeRenderer的render函数->updateContainer->scheduleUpdateOnFiber->performConcurrentWorkOnRoot/performSyncWorkOnRoot->renderRootConcurrent/Sync->workLoopConcurrent/Sync->performUnitOfWork->beginWork->updateClassComponent->finishClassComponent->组件render函数


### reconcileChildren(ReactFiberBeginWork.js)


finishClassComponent会调用reconcileChildren


调用链
reconcileChildren->reconcileChildFibers->reconcileSingleElement->createFiberFromElement->createFiberFromTypeAndProps->createFiber

最终创建fiber，其会赋值给workInProgress.child








## Component的生命周期

Component的生命周期如下

![](./renderphase.png)

参考Lin Clark - A Cartoon Intro to Fiber - React Conf 2017



```
componentDidMount
componentWillUnmount
```

以上两个操作都在commitRoot阶段进行

componentDidMount
- commitRoot
- commitLayoutMountEffects_complete，这里会一次调用commitLayoutEffectOnFiber，传入的参数为fiber，而fiber的遍历顺序为，首先通过sibling找兄弟节点，然后通过return往父节点上找
- commitLayoutEffectOnFiber
- 组件componentDidMount的调用

componentWillUnmount
- commitRoot
- commitMutationEffects_begin
- commitDeletion
- unmountHostComponents，这里调用commitUnmount的顺序是从父节点开始往下调用
- commitUnmount
- safelyCallComponentWillUnmount
- componentWillUnmount



commitLayoutEffects，最终调用commitHookEffectListMount，commitHookEffectListMount会调用effect.create函数执行effect。

其调用链路commitLayoutEffects->commitLayoutEffects_begin->ccommitLayoutMountEffects_complete->ommitLayoutEffectOnFiber->commitHookEffectListMount




## 宿主环境

### ReactNativeFiberHostComponent
这个类会调用UIManager管理的一个view，包含如下几个属性，_nativeTag，_children，viewConfig（包含组件的类信息，属性信息）
- measure方法，调用UIManager的measure方法，返回位置和大小
- setNativeProps，调用UIManager的updateView方法更新属性

### UIManager

createInstance（由completeWork调用），这里会返回一个ReactNativeFiberHostComponent，调用这个函数之后，会把它赋值给FiberNode的stateNode，而这个instance对应native UIManager管理的一个view
```
var instance = createInstance(
  type,
  newProps,
  rootContainerInstance,
  currentHostContext,
  workInProgress
);
appendAllChildren(instance, workInProgress, false, false);
workInProgress.stateNode = instance;
```

createInstance的实现
- 通过allocateTag()，创建一个tag，与10相除余1表示为rootTag，被2整除为Fabric tag，其他情况是以3为起点，2为递增的步长。
- 调用UIManager的createView方法，这里传入reacttag，viewName，rootTag，props，viewName和props来自requireNativeComponent函数
- 创建ReactNativeFiberHostComponent
- 返回前面创建的ReactNativeFiberHostComponent

```
// % 10 === 1 means it is a rootTag.
// % 2 === 0 means it is a Fabric tag.

var nextReactTag = 3;

function allocateTag() {
  var tag = nextReactTag;

  if (tag % 10 === 1) {
    tag += 2;
  }

  nextReactTag = tag + 2;
  return tag;
}
```


hideInstance，调用UIManager的updateView方法
```
var updatePayload = create(
  {
    style: {
      display: "none"
    }
  },
  viewConfig.validAttributes
);
ReactNativePrivateInterface.UIManager.updateView(
  instance._nativeTag,
  viewConfig.uiViewClassName,
  updatePayload
);
```

appendChild

入参是parentInstance和child，这里会调用UIManager的manageChildren


## Event

事件遵循了w3c的事件接口，见http://www.w3.org/TR/DOM-Level-3-Events/，react-native的实现则见EventInterface、SyntheticEvent

注册两个事件Plugin

```
injectEventPluginsByName({
  ResponderEventPlugin: ResponderEventPlugin,
  ReactNativeBridgeEventPlugin: ReactNativeBridgeEventPlugin
});
```


入口从native bridge接收事件
```
ReactNativePrivateInterface.RCTEventEmitter.register({
  receiveEvent: receiveEvent,
  receiveTouches: receiveTouches
});
```

_receiveRootNodeIDEvent
- 根据rootNodeID，调用getInstanceFromTag获取到对应的instance
- 调用runExtractedPluginEventsInBatch处理instance的nativeEvent
  - extractPluginEvents，遍历前面注入的两个plugin，ResponderEventPlugin、ReactNativeBridgeEventPlugin，分别调用对应的plugin的extractEvents
    - 通过nativeEvent拿到对应的ResponderSyntheticEvent（RespinderEventPlugin）/SyntheticEvent（ReactNativeBridgeEventPlugin）
    - 最后拿到组装的事件数组
  - runEventsInBatch，这里会执行executeDispatchesAndReleaseTopLevel进行事件派发，最终调用的是executeDispatch派发到对应的事件接收者上

