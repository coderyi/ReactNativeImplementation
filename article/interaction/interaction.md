
# Interaction

InteractionManager可以支持允许长时间的同步任务，任务会安排到所有互动或动画完成之后再进行。
```
 InteractionManager.runAfterInteractions(() => {
   // ...long-running synchronous task...
 });
```

与其他调度方案对比

- setImmediate/setTimeout()，延迟执行，但会delay动画
- runAfterInteractions，延迟执行，但不是delay动画


runAfterInteractions流程

- new 一个 Promise，并且初始化一个空的tasks数组
    - _scheduleUpdate，调用setTimeout/setImmediate，func为_processUpdate
    - tasks push task
    - tasks push 一个新的task，run为该Promise的resolve。
    - 把tasks入队_taskQueue
- return相关返回值
    - then，promise.then
    - done，promise.done
    - cacel，_taskQueue调用cancelTasks取消tasks


_processUpdate，主要是通知监听者并且处理队列
- _addInteractionSet遍历添加至_interactionSet
- _deleteInteractionSet遍历，依次从_interactionSet移除handle
- emit interactionComplete/interactionStart
- 只有当没有要处理的handle时候，才会循环调用_taskQueue.processNext()，这里最终会执行runAfterInteractions任务
- _addInteractionSet、_deleteInteractionSet均清空


createInteractionHandle
- _scheduleUpdate
- handle增加一个计数
- _addInteractionSet增加handle（number）

createInteractionHandle的调用方
- 在animate函数的开始，以及动画结束时，若__isInteraction为true分别调用createInteractionHandle、clearInteractionHandle
- 在PanResponder的onResponderGrant函数，会调用createInteractionHandle。然后在onResponderEnd时会调用clearInteractionHandle

clearInteractionHandle

- _scheduleUpdate
- _addInteractionSet删除传入handle
- _deleteInteractionSet增加该handle

## TaskQueue

_getCurrentQueue
- 取出_queueStack（数组）的最后一个元素设置为queue
- 若queue的popable为true且queue元素为空，则_queueStack pop，删除最后一个元素
    - 接着递归调用_getCurrentQueue
- 否则返回queue.tasks

enqueue，调用_getCurrentQueue入队task

enqueueTasks，分别调用enqueue入队task


processNext

- 调用_getCurrentQueue获取到栈顶的queue的tasks
- 取出tasks的第一个元素
    - 若task为函数，则直接执行task函数
    - 若task为SimpleTask，则调用task.run()
    - 若task为PromiseTask,则执行_genPromise(task)

_genPromise

- 取出_queueStack栈顶的元素item
- 调用入参task的gen函数，之后把前面的item的popable设置为true


## Batchinator

```
class Widget extends React.Component {
  constructor(props) {
    super(props);
    this._batchedSave = new Batchinator(() => this._saveState(), 1000);
  }

  _saveState() {
    // save this.state to disk
  }

  componentDidUpdate() {
    this._batchedSave.schedule();
  }

  componentWillUnmount() {
    this._batchedSave.dispose();
  }

}
```

具体看schedule方法
- 调用setTimeout
- timeout的callback为InteractionManager.runAfterInteractions，task内会回调Batchinator传入的callback



## JSEventLoopWatchdog

install逻辑
- 定义iteration函数
    - 获取busyTime，这个为一次setTimeout(iteration, thresholdMS / 5)的时间
    - 若busyTime大于thresholdMS，则认为是卡顿
        - 遍历handlers，调用onStall函数，进行卡顿回调
    - 遍历handlers，并且调用handler的onIterate函数
    - setTimeout(iteration, thresholdMS / 5);
- 调用iteration函数


BridgeSpyStallHandler

设置MessageQueue的spy，spy的实现是在enqueueNativeCall/__callFunction进行调用

当spy调用时会把此次调用加入spyBuffer

JSEventLoopWatchdog增加handler，onIterate时会把spyBuffer清空，在onStall则输出日志展示卡顿时相关的bridge调用




