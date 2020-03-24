# React Scheduler原理分析

这篇文章我们会从两个方面来进行讲解，首先会从整体的一个原理来讲述`Scheduler`是如何工作的，然后再从具体的代码去解释它是怎么来实现的。说实话`Scheduler`代码量并不算多，而且整体原理呢，也比较清晰。但是如果想读懂`React`的整体代码，那么这一部分是要理解透彻，这样一来在理解后面模块之间是如何调用及整体的工作流程是有帮助的。

## Scheduler原理
首先`Scheduler`调度的最小粒度的是一个任务，在整个程序运行过程当中，会产生很多这这样的任务，而且任务有自己的优先级，优先级高的比优先级低的先调度。而且浏览器的屏幕刷新频率是固定的，所以，就需要有一个执行任务的终止时间，不然就会出现卡顿。总体的原理还是相对简单的，没有什么黑科技。下面我们从源码层面来看如何实现任务调度，这样或许会更清晰一点


## Scheduler源码分析


### unstable_scheduleCallback

```javascript
// 创建任务的函数，Scheduler就是从这里开始的
function unstable_scheduleCallback(priorityLevel, callback, options) {
  var currentTime = getCurrentTime();

  var startTime;
  var timeout;
  if (typeof options === 'object' && options !== null) {
    var delay = options.delay;
    // 如果有延时，则任务的开始时间为当前时间加上延时的时间
    if (typeof delay === 'number' && delay > 0) {
      startTime = currentTime + delay;
    } else {
      startTime = currentTime;
    }
    // 如果有传入超时时间，则为传入的值，否则就是通过优先级来获取超时时间
    timeout =
      typeof options.timeout === 'number'
        ? options.timeout
        : timeoutForPriorityLevel(priorityLevel);
  } else {
    timeout = timeoutForPriorityLevel(priorityLevel);
    startTime = currentTime;
  }

  // 过期时间为开始时间加上超时时间
  var expirationTime = startTime + timeout;

  // 任务对象
  var newTask = {
    id: taskIdCounter++, // id
    callback, // 任务的逻辑
    priorityLevel, // 任务的优先级
    startTime, // 任务开始时间
    expirationTime, // 任务过期时间
    sortIndex: -1, // 任务排序字段（后期需要通过这个字段来获取优先执行的任务）
  };
  if (enableProfiling) {
    newTask.isQueued = false;
  }

  if (startTime > currentTime) {
    // This is a delayed task.
    // 如果开始时间大于当前时间，则是一个延时任务
    newTask.sortIndex = startTime;
    // 往延时队列里面
    push(timerQueue, newTask); // 通过最小堆来实现排序
    // 如果任务队列是空的，而且延时队列的优先级最高的是当前新创建的这个任务
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      // All tasks are delayed, and this is the task with the earliest delay.
      if (isHostTimeoutScheduled) {
        // Cancel an existing timeout.
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      // 开启定时器
      // Schedule a timeout.
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    //如果不是延时任务，则把过期时间作为排序标准
    newTask.sortIndex = expirationTime;
    // 将任务加入到任务队列当中
    push(taskQueue, newTask);
    if (enableProfiling) {
      markTaskStart(newTask, currentTime);
      newTask.isQueued = true;
    }
    // Schedule a host callback, if needed. If we're already performing work,
    // wait until the next time we yield.
    // 如果当前没有调度，并且没有在做任务，则发起一个调度，而且如果不在循环当中，则开始一个循环
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);
    }
  }

  return newTask;
}

```

### requestHostCallback
```javascript
requestHostCallback = function(callback) {
    scheduledHostCallback = callback;
    // 然后通过消息，就是MessageChannel来异步循环
    if (!isMessageLoopRunning) {
      isMessageLoopRunning = true;
      // 执行postMessage, 就会执行performWorkUntilDeadline(异步)
      port.postMessage(null);
    }
  };

```

### performWorkUntilDeadline

```javascript
// 循环执行work，直到时间终止
const performWorkUntilDeadline = () => {
    if (scheduledHostCallback !== null) {
      const currentTime = getCurrentTime();
      // Yield after `yieldInterval` ms, regardless of where we are in the vsync
      // cycle. This means there's always time remaining at the beginning of
      // the message event.
      // 从当前时间开始算yieldInterval ms, 任务终止时间
      deadline = currentTime + yieldInterval;
      const hasTimeRemaining = true;
      try {
        // scheduledHostCallback实际上就是flushWork
        const hasMoreWork = scheduledHostCallback(
          hasTimeRemaining,
          currentTime,
        );
        if (!hasMoreWork) {
            // 如果没有更多work，则消息循环为false
          isMessageLoopRunning = false;
          scheduledHostCallback = null;
        } else {
          // If there's more work, schedule the next message event at the end
          // of the preceding one.
          // 如果有，则调度下一次消息循环
          port.postMessage(null);
        }
      } catch (error) {
        // If a scheduler task throws, exit the current browser task so the
        // error can be observed.
        port.postMessage(null);
        throw error;
      }
    } else {
      isMessageLoopRunning = false;
    }
    // Yielding to the browser will give it a chance to paint, so we can
    // reset this.
    needsPaint = false;
  };
```

### flushWork

上面说的调用`scheduledHostCallback`实际上就是执行的flushWork，那我们看看它是怎么执行的

```javascript
function flushWork(hasTimeRemaining, initialTime) {
  if (enableProfiling) {
    markSchedulerUnsuspended(initialTime);
  }

  // We'll need a host callback the next time work is scheduled.
  // 我们需要把该标志位置为false
  isHostCallbackScheduled = false;
  if (isHostTimeoutScheduled) {
    // 取消任务定时器
    // We scheduled a timeout but it's no longer needed. Cancel it.
    isHostTimeoutScheduled = false;
    cancelHostTimeout();
  }
  
  // 开始工作
  isPerformingWork = true;
  const previousPriorityLevel = currentPriorityLevel;
  try {
    if (enableProfiling) {
      try {
        // 循环执行work
        return workLoop(hasTimeRemaining, initialTime);
      } catch (error) {
        if (currentTask !== null) {
          const currentTime = getCurrentTime();
          markTaskErrored(currentTask, currentTime);
          currentTask.isQueued = false;
        }
        throw error;
      }
    } else {
      // No catch in prod codepath.
      return workLoop(hasTimeRemaining, initialTime);
    }
  } finally {
    currentTask = null;
    currentPriorityLevel = previousPriorityLevel;
    isPerformingWork = false;
    if (enableProfiling) {
      const currentTime = getCurrentTime();
      markSchedulerSuspended(currentTime);
    }
  }
}
```

### workLoop

```javascript
// 循环执行任务，只要时间还够，或者时间不够，但是已经过期的
function workLoop(hasTimeRemaining, initialTime) {
  let currentTime = initialTime;
  // 开始执行之前，先处理一下任务，把在timerQueue中符合执行条件的任务放到
  // taskQueue当中
  advanceTimers(currentTime);
  currentTask = peek(taskQueue);
  while (
    currentTask !== null &&
    !(enableSchedulerDebugging && isSchedulerPaused)
  ) {
    // 如果当前任务还没过期，然后也没有时间了，就返回去执行其他事情
    if (
      currentTask.expirationTime > currentTime &&
      (!hasTimeRemaining || shouldYieldToHost())
    ) {
      // This currentTask hasn't expired, and we've reached the deadline.
      break;
    }
    const callback = currentTask.callback;
    
    if (callback !== null) {
      currentTask.callback = null;
      currentPriorityLevel = currentTask.priorityLevel;
      // 计算是否超时
      const didUserCallbackTimeout = currentTask.expirationTime <= currentTime;
      markTaskRun(currentTask, currentTime);
      const continuationCallback = callback(didUserCallbackTimeout);
      currentTime = getCurrentTime();
      if (typeof continuationCallback === 'function') {
        // 如果返回的是一个函数，则添加到当前任务的callback当中, 然后不会从队列里剔除
        currentTask.callback = continuationCallback;
        markTaskYield(currentTask, currentTime);
      } else {
        if (enableProfiling) {
          markTaskCompleted(currentTask, currentTime);
          currentTask.isQueued = false;
        }
        // 如果不是返回一个函数，则判断当前任务是否跟队列的第一个任务是同一个任务，如果是，则删除
        if (currentTask === peek(taskQueue)) {
          pop(taskQueue);
        }
      }
      // 从timerQueue里取出过期的任务
      advanceTimers(currentTime);
    } else {
      // 如果callback为null, 则认为是任务已经取消了，直接从任务中删除
      pop(taskQueue);
    }
    // 继续下一个任务，直到返回
    currentTask = peek(taskQueue);
  }
  // Return whether there's additional work
  // 如果当前任务不为空，则还有额外的任务，返回true，进入下一次循环
  if (currentTask !== null) {
    return true;
  } else {
    // 如果当前任务为空，则任务没有任务，那么从timerQueue里面取出一个timer, 然后起一个定时器，
    let firstTimer = peek(timerQueue);
    if (firstTimer !== null) {
      requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime);
    }
    return false;
  }
}

```


