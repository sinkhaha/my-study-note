> 本文 Node.js 的版本 v18.15.0 、libuv 的版本 v1.44.2 (libuv 相关只涉及 unix 平台)

# 1 setImmediate()

## 1.1 setImmediate() 源码

Node.js 中 [setImmediate()](https://github.com/nodejs/node/blob/v18.15.0/lib/timers.js#L279-L303) 的源码如下

```js
// node-18.15.0/lib/timers.js
function setImmediate(callback, arg1, arg2, arg3) {
  validateFunction(callback, 'callback');

  let i, args;
  switch (arguments.length) {
    case 1:
      break;
    case 2:
      args = [arg1];
      break;
    case 3:
      args = [arg1, arg2];
      break;
    default: // 跟setTimeout一样，处理多余3个的参数
      args = [arg1, arg2, arg3];
      for (i = 4; i < arguments.length; i++) {
        args[i - 1] = arguments[i];
      }
      break;
  }

  // 直接返回Immediate实例
  return new Immediate(callback, args);
}
```

可以看出，首先处理回调函数的参数 `args`，然后实例化 `Immediate`（在里面会把 `immediate` 插入链表中），最后直接返回 `Immediate` 实例



### 1.1.2 Immediate 管理类

Node.js 中 [`Immediate`](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/timers.js#L611-L649) 类的源码如下

```js
const immediateQueue = new ImmediateList(); // 存储immediate的链表

// node-18.15.0/lib/internal/timers.js
class Immediate {
  constructor(callback, args) {
    this._idleNext = null;
    this._idlePrev = null;
    this._onImmediate = callback;
    this._argv = args;
    this._destroyed = false;
    this[kRefed] = false;

    initAsyncResource(this, 'Immediate');

    this.ref();
    immediateInfo[kCount]++; // immediate链表的节点数

    immediateQueue.append(this); // 插入链表
  }
  
  ref() {
    if (this[kRefed] === false) {
      this[kRefed] = true;
      if (immediateInfo[kRefCount]++ === 0)
        toggleImmediateRef(true);
    }
    return this;
  }

  unref() {
    if (this[kRefed] === true) {
      this[kRefed] = false;
      if (--immediateInfo[kRefCount] === 0)
        toggleImmediateRef(false);
    }
    return this;
  }

  hasRef() {
    return !!this[kRefed];
  }
}
```

* `Immediate` 类也与前一章讲解的 `setTimeout()` 中的  `Timeout` 类似，都存储了一些元信息；`Immediate` 中还有 `ref()` 与 `unref()`方法 

* 最后会将自己插入一个 [ `ImmediateList` 链表 ](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/timers.js#L267-L304)中

  > 这是一条普通的链表，`ImmediateList` 只有 `append()`（插入到链表最后）和 `remove()`（移除某个元素）两个操作



## 1.2 setImmediate() 和空转事件的关系

在了解 `setImmediate()` 函数的执行时机前，先看下它和空转事件的关系

### 1.2.1 空转事件

**libuv 的事件循环内部顺序图如下**



![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/20230405205556.png)





**即一次事件循环的逻辑顺序依次为：**

* 定时器事件
* `Pending 态 I/O `事件
* 空转事件
* 准备事件
* `Poll I/O` 事件
* 复查事件
* 扫尾

由此可知，空转事件是属于事件循环的第3个阶段。`Node.js v18.15.0`中，只有一个地方用到了空转事件，那就是 `setImmediate()`。



### 1.2.2 immediate handle 插入 idle 空转队列

在 `Immediate` 实例化时，会调用 `ref()`方法，其作用是：

因为事件循环 `Poll I/O` 阶段计算阻塞事件时，不会考虑 `check` 复查阶段的任务，但会考虑 `idle` 空转阶段的任务，所以当插入第一个 `Immediate` 任务时，Node.js 会把 `immediate_idle_handle ` 插入 `idle` 空转队列中（`idle`阶段并不会去执行 `Immediate` 实例），插进去只起到标记作用，表示有任务处理，不能阻塞 `Poll I/O` 阶段。



`ref()` 代码如下

```js
// node-18.15.0/lib/internal/timers.js
class Immediate {
  constructor(callback, args) {
    ...
    this[kRefed] = false; // ref标记为false

    this.ref(); // 调用ref()
    
    // Immediate 链表的节点个数，包括 ref 和 unref 状态  
    immediateInfo[kCount]++;  
    
    // 加入链表中  
    immediateQueue.append(this);  
  }

  // 打上 ref 标记，往 Libuv 的 idle 链表插入一个激活状态的节点，如果还没有的话    
  ref() {
    if (this[kRefed] === false) {
      this[kRefed] = true; // ref标记为true
      
      // immediateInfo[kRefCount] 为 0，即 ref 个数为0，则说明还没有往 libuv 的 idle 队列里插入idle 节点，则把immediate_idle_handel插入idle空转队列，只是标记作用，idle并不会执行immediate
      if (immediateInfo[kRefCount]++ === 0) 
        toggleImmediateRef(true); // 对应c++侧timers.cc的toggleImmediateRef
    }
    return this;
  }
  ......
}
```

由代码可知，`ref()` 调用了 C++ 侧 `src/timers.cc` 中的 [`ToggleImmediateRef`](https://github.com/nodejs/node/blob/v18.15.0/src/timers.cc#L41-L43) 函数，代码如下

```c++
// node-18.15.0/src/timers.cc
void ToggleImmediateRef(const FunctionCallbackInfo<Value>& args) { 
  // 调用了env的ToggleImmediateRef函数
  Environment::GetCurrent(args)->ToggleImmediateRef(args[0]->IsTrue())
}  
```

由代码可知，`timers.cc` 里又进一步调用了 C++ 侧 `src/env.cc` 中的 [`ToggleImmediateRef`](https://github.com/nodejs/node/blob/v18.15.0/src/env.cc#L1256-L1265) 函数，代码如下

```c++
void Environment::ToggleImmediateRef(bool ref) {
  if (started_cleanup_) return;

  if (ref) { // 插入 idle 队列，设置为 active 状态，防止在 Poll IO 阶段阻塞和事件循环的退出 ，回调是一个空的匿名函数
    uv_idle_start(immediate_idle_handle(), [](uv_idle_t*){ }); // libuv的uv_idle_start函数
  } else {
    uv_idle_stop(immediate_idle_handle()); // 停止immediate_idle_handle句柄
  }
}
```

由代码可知，实际 `ref()` 最后调用的是 `libuv` 的 `uv_idle_start()` 插入 `idle` 队列，此时 `uv_idle_start` 的事件回调是一个空的匿名函数。

这个 `ToggleImmediateRef` 函数的作用是当有一个 `Immediate` 被 `ref` 了，Node.js 就会激活这个 `idle  ` 空转句柄，让 `Poll I/O`  不阻塞；如果没有被 `ref` 了，则停止该句柄。



### 1.2.3 空转事件不会阻塞事件循环

我们知道空转事件不会阻塞事件循环，因为在事件循环中，在 `Poll for I/O` 阶段时，阻塞的最大等待时间是根据 [`uv_backend_timeout()`](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/src/unix/core.c#L404)函数的返回值判断的，其规则如下

* 如果事件循环中存在空转事件，此函数会返回 `0`，即不阻塞 `Poll I/O`等待，可以马上开始进入下一轮轮回
* 如果没有空转事件，则返回下一个最快超时的 `Timeout` 定时器的过期时间，此过期时间会做为事件循环的最大阻塞时间（因为有即将超时的定时器，说明事件循环中还有定时器需要处理，不能一直阻塞）

`uv_backend_timeout()` 源码如下

```c++
// node-18.15.0/deps/uv/src/unix/core.c
static int uv__backend_timeout(const uv_loop_t* loop) {
  if (loop->stop_flag == 0 &&
      /* uv__loop_alive(loop) && */
      (uv__has_active_handles(loop) || uv__has_active_reqs(loop)) &&
      QUEUE_EMPTY(&loop->pending_queue) &&
      QUEUE_EMPTY(&loop->idle_handles) &&
      loop->closing_handles == NULL)
    return uv__next_timeout(loop);
  return 0;
}

int uv_backend_timeout(const uv_loop_t* loop) {
  if (QUEUE_EMPTY(&loop->watcher_queue))
    return uv__backend_timeout(loop);
  /* Need to call uv_run to update the backend fd state. */
  return 0;
}
```

注意：这里判断是否阻塞时，并不会考虑 `check` 复查阶段的任务



**空转事件（idle节点）存在的意义：**是**为了标记是否有 `immediate` 任务需要处理**（为了让 `Poll for I/O ` 阶段不阻塞）；因为有 `immediate` 任务的话就事件循环就不能一直阻塞在 `Poll I/O` 阶段等待 `I/O`，并且不能退出事件循环。



## 1.3 setImmediate() 的启动和执行

`setImmediate()` 执行时机是在事件循环的[`check` 复查事件阶段](https://github.com/nodejs/node/blob/v18.15.0/src/env.cc#LL864C15-L864C71)，从事件循环中我们知道该阶段是在 `Poll for I/O` 阶段之后(这里只针对同一个`tick`有效，即同一轮事件循环内)。



### 1.3.1 激活 Immediate 并设置 idle节点

在 Node.js 初始化时会调用 [`InitializeLibuv()`](https://github.com/nodejs/node/blob/v18.15.0/src/env.cc#L852-L903) 初始化 `libuv` 的一些东西，里面调用了 `libuv` 的[`uv_check_start()`](https://github.com/nodejs/node/blob/v18.15.0/src/env.cc#L864)。`uv_check_start() `的作用就是激活 `immediate check handle` 句柄，并设置回调为 [`CheckImmediate`](https://github.com/nodejs/node/blob/v18.15.0/src/env.cc#L1231-L1254)。还调用了 `uv_idle_init()` 初始化一个 `idle`  阶段的节点

> `libuv-1.44.2`中， `uv_check_start` 是用宏的方式定义在 `deps/uv/src/unix/loop-watcher.c` 文件中(unix平台)

```c
// node-18.15.0/src/env.cc
// 在 Node.js 初始化时调用
void Environment::InitializeLibuv() {
    ...
    // 初始化 immediate 相关的 handle   
    CHECK_EQ(0, uv_check_init(event_loop(), immediate_check_handle()));
  
    // 修改 immediate_check_handle 状态为 unref，避免没有任务时，影响事件循环的退出
    // immediate_check_handle是uv_check_t类型，它继承uv_handle_t
    uv_unref(reinterpret_cast<uv_handle_t*>(immediate_check_handle()));

    // 激活immediate check handle，设置CheckImmediate回调  
    CHECK_EQ(0, uv_check_start(immediate_check_handle(), CheckImmediate));
  
    // 这里初始化一个 idle 阶段的节点
    CHECK_EQ(0, uv_idle_init(event_loop(), immediate_idle_handle()));
    ...
}
```



**为什么 Node.js 会初始化一个 idle 阶段的节点**

* 因为 `InitializeLibuv()` 中会先通过 `uv_unref()` 方法把 `immediate_check_handle` 状态设置成了 `unref`，由于 `unref ` 状态的 ` handle` 不会维持事件循环的持续运行，这样就会带来一个问题：可能 Node.js 里还有 `immediate` 任务，而事件循环却退出了，所以得想办法维持事件循环的运行，让接下去的  `immediate` 任务可以按时执行
* 因为 `libuv` 的设计，`libuv` 判断是否需要阻塞在 `Poll I/O` 阶段时，不会考虑 `check` 复查阶段的任务。且 `Poll I/O` 阶段是否需要阻塞 和 阻塞多久的逻辑是 [`uv_backend_timeout()`](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/src/unix/core.c#L404)函数实现的；从 `uv_backend_timeout()` 中可知，即使 `check` 阶段有任务，`uv_backend_timeout()` 也不会返回0，此时 `Poll I/O` 阶段还是可能会一直阻塞，这样会导致 `check` 阶段的任务无法按时执行了。但 Node.js 并不希望这样，它希望用户通过 `setImmediate` 产生一个任务之后可以尽快执行

* 针对这个问题，Node.js 使用的解决办法是在设置一个 `unref ` 状态的 `immediate check handle` 的前提下，再通过往 `idle` 插入一个 `immediate idle handle` 来控制 `Poll I/O` 阶段是否可以阻塞以及是否允许事件循环的退出（即通过 `idle` 阶段是否有 `immediate` 任务进行判断）。因为从 `uv_backend_timeout` 方法可以看到，`idle` 阶段的任务可以控制 `Poll I/O ` 会不会发生阻塞（ `idle` 有任务会返回 0，`Poll I/O` 不会阻塞）



### 1.3.2 libuv 侧回调函数对应的 C++ 侧函数：CheckImmediate()

事件循环在执行 `check` 复查阶段的任务时，会执行注册好的 [`CheckImmediate()`](https://github.com/nodejs/node/blob/v18.15.0/src/env.cc#L1231-L1254) 回调函数，其作用类似于 `setTimeout()` 中的 `RunTimers()`，代码如下

```c++
// node-18.15.0/src/env.cc
void Environment::CheckImmediate(uv_check_t* handle) {
  ...
  // 没有 Immediate 任务需要处理    
  if (env->immediate_info()->count() == 0 || !env->can_call_into_js())
    return;

  do { // 执行 JS 层回调 immediate_callback_function 
    MakeCallback(env->isolate(),
                 env->process_object(),
                 env->immediate_callback_function(), // 执行在初始化时就注册好的processImmediate()函数
                 0,
                 nullptr,
                 {0, 0}).ToLocalChecked();
  } while (env->immediate_info()->has_outstanding() && env->can_call_into_js());
  
  // 所有 immediate 节点都处理完了，置 idle 阶段对应节点为非激活状态，允许 Poll IO 阶段阻塞和事件循环退出
  if (env->immediate_info()->ref_count() == 0)
    env->ToggleImmediateRef(false);
}
```

在 `CheckImmediate()` 里会不断去执行已经注册好的 `processImmediate() `回调函数，直到所有 `immediate` 节点都处理完了



### 1.3.3 C++ 侧回调函数对应的 js 侧函数：processImmediate()

js 侧 [processImmediate() ](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/timers.js#L431-L494) 函数的作用：就是用于处理 `js` 侧 `immediate` 链表中的`Immediate` 实例，并执行 `setImmediate()` 中的回调函数。



**注册 `processImmediate()` 函数**

Node.js 在启动时，会注册 `processImmediate()` 函数到 `env` 的 `immediate_callback_function()` 中(类似 `setTimeout()` 中的 `processTimers()`)，源码如下

```js
// node-18.15.0/lib/internal/bootstrap/node.js
const { setupTimers } = internalBinding('timers');
  const {
    processImmediate,
    processTimers,
  } = internalTimers.getTimerCallbacks(runNextTicks);
  // Sets two per-Environment callbacks that will be run from libuv:
  // - processImmediate will be run in the callback of the per-Environment
  //   check handle.
  // - processTimers will be run in the callback of the per-Environment timer.
  setupTimers(processImmediate, processTimers); // `setupTimers`是内置模块`timers.cc`的方法
```

```c++
// node-18.15.0/src/timers.cc
void SetupTimers(const FunctionCallbackInfo<Value>& args) {
  CHECK(args[0]->IsFunction());
  CHECK(args[1]->IsFunction());
  auto env = Environment::GetCurrent(args);

  // env的immediate_callback_function()是js层的processImmediate函数
  env->set_immediate_callback_function(args[0].As<Function>());
  // env的timers_callback_function()是js层的processTimers函数
  env->set_timers_callback_function(args[1].As<Function>()); 
}
```

由此可知，`env` 的 `immediate_callback_function()` 函数就是 `js` 侧的 `processImmediate()` 函数



**`processImmediate()` 的解析**

[`processImmediate()`](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/timers.js#L431-L494) 的源码如下

```js
// node-18.15.0/lib/internal/timers.js
const immediateQueue = new ImmediateList();
...
const outstandingQueue = new ImmediateList();
...

function processImmediate() {
    const queue = outstandingQueue.head !== null ?
      outstandingQueue : immediateQueue;
    let immediate = queue.head; // 赋给新的链表

    // 防重入
    // Clear the linked list early in case new `setImmediate()`
    // calls occur while immediate callbacks are executed
    if (queue !== outstandingQueue) { // 说明当前queue是immediateQueue
      queue.head = queue.tail = null;
      immediateInfo[kHasOutstanding] = 1;
    }

    let prevImmediate;
    let ranAtLeastOneImmediate = false;
    while (immediate !== null) { // 遍历immediate队列
      if (ranAtLeastOneImmediate)
        runNextTicks();
      else
        ranAtLeastOneImmediate = true;

      // It's possible for this current Immediate to be cleared while executing
      // the next tick queue above, which means we need to use the previous
      // Immediate's _idleNext which is guaranteed to not have been cleared.
      if (immediate._destroyed) {
        outstandingQueue.head = immediate = prevImmediate._idleNext;
        continue;
      }

      // TODO(RaisinTen): Destroy and unref the Immediate after _onImmediate()
      // gets executed, just like how Timeouts work.
      immediate._destroyed = true;

      immediateInfo[kCount]--;
      if (immediate[kRefed])
        immediateInfo[kRefCount]--;
      immediate[kRefed] = null;

      prevImmediate = immediate;

      ......
      
      try {
        const argv = immediate._argv;
        if (!argv)
          immediate._onImmediate();
        else
          immediate._onImmediate(...argv); // 执行回调函数
      } finally { // 注意这里执行_onImmediate时没有catch
        immediate._onImmediate = null;

        ......

        outstandingQueue.head = immediate = immediate._idleNext;
      }
      ......
    }

    if (queue === outstandingQueue)
      outstandingQueue.head = null;
    immediateInfo[kHasOutstanding] = 0;
  }

```

主要逻辑

* 首先，先判断 `outstandingQueue` 是不是为空，若为空，则将 `queue` 赋值为 `immediateQueue`，否则赋值为 `outstandingQueue`。然后将 `queue` 赋给 `immediate`，后面的遍历就是遍历这个 `immediate` (这里另一条链表就是 `immediate`)
* 下面这个 `if(queue !== outstandingQueue){}` 就是防重入的逻辑。如果 `queue` 不等于 `outstandingQueue`，就说明它是 `immediateQueue`。如果当前遍历的是 `immediateQueue`，那么就清空这个 `immediateQueue`（将其首尾都赋空）。这样，在下面遍历时是遍历着已经拿过来的 `immediate`，在这之间新插入的 `Immediate` 是插入到已被赋空的 `immediateQueue` 链表了，两条链表毫无关系，不会再出现死循环
* 接下去 `while` 循环遍历 `immediate` 队列
* 调用 `immediate._onImmediate` 执行 `Immediate` 的回调函数

所以，`processImmediate()` 的大概流程就是：遍历这条 `ImmediateList` 链表，并逐个执行其回调函数 `_onImmediate`。



**防重入措施**

`processImmediate()` 里做了一个防重入的措施：在遍历前，将 `immediate` 链表给到另一条链表，然后清空 `immediateQueue`。因为在 `processImmediate()` 遍历过程中，如果始终用同一条链表，会导致刚从链表内拿出一个 `Immediate`，紧接着在它的回调函数中又插入一个新的 `Immediata`（`immediateQueue.append(this)`），最后它就一直在遍历这个链表，所以才需要另一条链表做防重入。类似以下代码：

```js
function set() {
  console.log('hello');
  setImmediate(set); // 在setImmediate回调中又调用了setImmediate，如果没有防重入，会一直执行这个链表
}
set();
```



**`outstandingQueue` 队列的作用**

* 首先，执行 `Immediate` 的 `_onImmediate()` 函数时，Node.js 用的是 `try-finally`，并没有 `catch`。这样会导致，如果 `setImmediate()`  中的回调函数抛错了，会触发 `uncaughtException` 事件。这时，如果用户监听了该错误事件并处理了，那么 Node.js 会继续执行，但是这个 `Immediate` 链表的遍历过程就被中断了，后面接下去再执行的话，就需要用到 `outstandingQueue`了

* `outstandingQueue` 起到了**保留现场**的作用。每次遍历执行一次  `Immediate `的 `_onImmediate()`  后的 `finally` 中，都记录一下 `outstandingQueue` 的首元素为当前执行完的 `Immediate` 的下一个元素。这样，j就算抛错了，也记录下来了现场。所以在 `processImmediate()` 函数的前面能看到，`queue` 的赋值时，`outstandingQueue` 是否为空。若不为空则说明是记录下现场后，有抛错，那么从之前的现场继续开始执行



## 1.4 例子

**例1：**

在 Node.js 下运行以下代码会发现结果是随机的，有时先 `setImmediate`，而有时则先执行 `setTimeout`。

```js
setImmediate(() => {
  console.log('setImmediate');
});

setTimeout(() => {
  console.log('setTimeout');
}, 0); // 超时时间为0，Timeout对象内部会处理为1
```

因为 `setTimeout()` 的超时时间为 `0`，在 `Timeout` 构造函数中会被自动设为 `1`。

只有在同一轮循环中，定时器事件 才早于 复查事件执行，这里 `setTimeout` 跟 `setImmediate` 不一定在同一轮循环中，所以出现了随机的执行顺序，这段代码的输出结果是取决于开始启动定时器到 `libuv` 执行定时器阶段的时间是否过去了 `1ms`



Node.js 启动时，执行完上面的代码（启动一个定时器，创建一个 `setImmediate` 任务），再进入事件循环。在执行定时器阶段，`libuv` 判断从开启定时器到现在是否已经过去了 `1ms`，是的话就先执行 `setTimeout` 回调，否则会进入 `check` 阶段，从而执行 `setImmediate` 的回调。



**例2：**

```js
setImmediate(() => {
  setImmediate(() => {
    console.log('setImmediate');
  });

  setTimeout(() => {
    console.log('setTimeout');
  }, 0);
}, 0);
```

这个例子的执行顺序也是随机的。当执行最外层 `setImmediate` 回调时，当前已经处于复查阶段，因为 `Immediate` 防重入机制让里面的 `setImmediate` 肯定不会再在当前 `Tick` （当前循环）进行，所以里面的  `setImmediate`  与  `setTimeout`  都是要在下个 `Tick` 进行了，此时又跟例1一样的情况，在下一轮 `Tick` 中，要看时间是不是超过 `1ms`  决定先执行哪个。



**例3：**

```js
setTimeout(() => {
  setImmediate(() => {
    console.log('setImmediate');
  });

  setTimeout(() => {
    console.log('setTimeout');
  }, 0);
}, 0);
```

这一定是先执行 `setImmediate` 再执行 `setTimeout` 了，因为跑完外层 `setTimeout` 后，直接就到后续阶段了，接下去肯定是先执行复查阶段，然后再是下个 Tick 才能执行后续的 `Timeout`。



**例4：**

 `setImmediate()` 与 `fs` 

```js
setImmediate(() => {
  console.log('setImmediate');
});

require('fs').readFile('temp.js', () => {
  console.log('readFile');
});
```

首先，在这个 `Tick` 中的执行顺序肯定是先 `Poll for I/O` 然后再复查 `check`。但问题在于，`Poll for I/O` 阶段，它等待文件系统事件的时间为 `0`，`0` 时间内等不到事件，那么会继续执行后续逻辑。而对一个这种可读事件来说，通常不会在 `0` 的时间内完成触发，所以第一个 `Tick` 基本上都是直接在 `Poll for I/O` 阶段假模假式等你 `0` 毫秒，然后就直奔复查阶段去了。文件在第一个 `Tick` 没读出来，自然是在下一个 `Tick` 中读出来。所以如果没有一些特殊情况，上面的代码 `setImmediate` 总会先于 `readFile` 被输出。



**例5：**

改一下例4的代码

```js
function imm() {
  setImmediate(() => {
    console.log('setImmediate');
    imm();
  });
}
imm();

require('fs').readFile('temp.js', () => {
  console.log('readFile');
  process.exit(0);
});
```

执行几次，会发现输出了几行 `setImmediate` 后，终于读取成功了，然后输出 `readFile` 并退出了。虽然我们执行等待的时候等待时间为 `0`，但是整段 js 在每个 `Tick` 执行时间还是有纳秒、微妙、毫秒级别的耗时，所以到下几个 `Tick` 时，读文件完成的事件已经到了，不用等就能直接拿到，于是执行 `readFile()` 的回调函数。



# 2 微任务 queueMicrotask()

## 2.1 微任务的理论执行时机

在 Web API 中，有一个 `queueMicrotask()` 函数，这个就是微任务，微任务在 [WHATWG 规范 ](https://html.spec.whatwg.org/multipage/timers-and-user-prompts.html#microtask-queuing)里是有定义的：

> The `queueMicrotask()` method allows authors to schedule a callback on the [microtask queue](https://html.spec.whatwg.org/multipage/webappapis.html#microtask-queue). This allows their code to run once the [JavaScript execution context stack](https://tc39.es/ecma262/#execution-context-stack) is next empty, which happens once all currently executing synchronous JavaScript has run to completion. This doesn't yield control back to the [event loop](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop), as would be the case when using, for example, `setTimeout(f, 0)`.

也就是，微任务是在**“当前 js 执行上下文堆栈”**完毕之后，要开始执行下一坨 ` js` 之前，在这个空档之间执行的任务。这与事件循环没有必然联系。在` libuv` 里面事件循环同一个阶段，也会出现这种情况。



比如：监听了很多文件系统 ` I/O` 事件，并且在某一个事件循环中同时拿到事件并触发，它们处于同个阶段，但它们的 js 执行上下文堆栈则不同。用 `setImmediate()` 举例，理论上，在复查阶段执行 `CheckImmediate()` 函数，触发这个事件时，没有任何` V8` 引擎介入的。在该函数内部，Node.js 为其创建了一个 `HandleScope` 和一个 `Context::Scope`，这才有了一个执行 `js `的上下文环境：

```c++
void Environment::CheckImmediate(uv_check_t* handle) {
  ...

  HandleScope scope(env->isolate());
  Context::Scope context_scope(env->context());
  
  ...
}
```

这才是所谓的“一次 js 上下文堆栈”。所以理论上执行完这个 `CheckImmediate()` 的 js 侧代码后，就会执行一次微任务。定时器 `setTimeout()` 最终回调执行机制同理。



**有哪些微任务？**

1.  V8 的 `Promise`
2. Node.js 中通过 `queueMicroTask()` 往微任务队列里插入自己所需要跑的微任务



## 2.2 微任务的执行时机

### 2.2.3 V8 中微任务执行时机的策略

在 V8 中，微任务的执行时机是由策略决定的，该策略是 `MicrotasksPolicy` 的枚举值，通过 `v8::Isolate::SetMicrotasksPolicy` 设置给 V8 引擎的。该枚举值有：

- `kExplicit`
- `kScoped`
- `kAuto`



### 2.2.4 Node.js 中微任务执行时机策略

Node.js 微任务执行时机用的策略是 [kExplicit](https://github.com/nodejs/node/blob/v18.15.0/src/node.h#L447)

```c++
// node-18.15.0/src/node.h
struct IsolateSettings {
  ...
  v8::MicrotasksPolicy policy = v8::MicrotasksPolicy::kExplicit; // kExplicit策略
  ...
};
```

使用该策略时，微任务只会在显式调用 `MicrotaskQueue::PerformCheckpoint()` 或`Isolate::PerformMicrotaskCheckpoint()` 时被执行。



### 2.2.5 Node.js 的5种执行时机埋点

 Node.js 的微任务是自己一个个埋点找时机进去执行的；下面看下 Node.js v18.15.0 中调用到 `PerformCheckpoint()` 的5处地方。

> 这仅代表 Node.js，不代表浏览器。每个浏览器的实现也不一样，甚至有些有可能直接用的 `kAuto` 策略



1、在` ECMAScript modules` 加载后，在 [`ModuleWrap::Evaluate()`](https://github.com/nodejs/node/blob/v18.15.0/src/module_wrap.cc#L389) 方法中会去执行一次微任务

```c++
// node-18.15.0/src/module_wrap.cc
void ModuleWrap::Evaluate(...) {
  ...
  auto run = [&]() {
    MaybeLocal<Value> result = module->Evaluate(context);
    if (!result.IsEmpty() && microtask_queue)
      microtask_queue->PerformCheckpoint(isolate); // 执行微任务
    return result;
  };
  
  ...
  result = run();
}
```



2、[vm 沙箱执行完一次之后](https://github.com/nodejs/node/blob/v18.15.0/src/node_contextify.cc#L1046-L1051)，并且 `vm` 沙箱用的非主 `Context` 执行，而是通过  `vm.createContext()` 创建新的 `Context` 执行的时候，会去执行一次微任务

> 因为 `mtask_queue` 只有在非主 `Context` 的时候才会被设值

```c++
// node-18.15.0/src/node_contextify.cc
bool ContextifyScript::EvalMachine(...) {
  ...
  auto run = [&]() {
    MaybeLocal<Value> result = script->Run(context);
    if (!result.IsEmpty() && mtask_queue)
      mtask_queue->PerformCheckpoint(env->isolate()); // 执行微任务
    return result;
  };
  ...
}
```



3、Node.js 内部一些 `callback` 函数执行之后也会执行微任务。因为在 `InternalMakeCallback()` 中，它会在里面创建一个 [`InternalCallbackScope`](https://github.com/nodejs/node/blob/v18.15.0/src/api/callback.cc#L193) 实例，该实例在结束的时候([ `InternalCallbackScope::Close()` ](https://github.com/nodejs/node/blob/v18.15.0/src/api/callback.cc#L132-L136))，会执行一次微任务

```c++
// src/api/callback.cc
MaybeLocal<Value> InternalMakeCallback(Environment* env,
                                       Local<Object> resource,
                                       Local<Object> recv,
                                       const Local<Function> callback,
                                       int argc,
                                       Local<Value> argv[],
                                       async_context asyncContext) {
    ...
      
    InternalCallbackScope scope(env, resource, asyncContext, flags);

    ...
    
    // 调用Close方法
    scope.Close();

    ...

}

void InternalCallbackScope::Close() {
  ...
  if (!tick_info->has_tick_scheduled()) {
    context->GetMicrotaskQueue()->PerformCheckpoint(isolate); // 执行微任务

    perform_stopping_check();
  }
  ...
}
```

 `InternalMakeCallback()` 通常用于唤起一些内置的回调函数，比如文件系统、网络 I/O 等等



4、 `setTimeout()`、`setInterval()`、`setImmediate()` 等时序 API，每次执行 [`runNextTicks()`](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/process/task_queues.js#L58-L65) 方法时，也会执行一次微任务

```js
// node-18.15.0/lib/internal/process/task_queues.js
function runNextTicks() {
  if (!hasTickScheduled() && !hasRejectionToWarn())
    runMicrotasks(); // 执行微任务
  if (!hasTickScheduled() && !hasRejectionToWarn())
    return;

  processTicksAndRejections();
}
```

`runNextTicks()` 里面主要调用了 `runMicrotasks()` 执行微任务，其对应的是 C++ 侧 `node_task_queue.cc `文件中的 [`RunMicrotasks()`](https://github.com/nodejs/node/blob/v18.15.0/src/node_task_queue.cc#L173-L176) 函数

```c++
// node-18.15.0/src/node_task_queue.cc
static void RunMicrotasks(const FunctionCallbackInfo<Value>& args) {
  Environment* env = Environment::GetCurrent(args);
  env->context()->GetMicrotaskQueue()->PerformCheckpoint(env->isolate()); // 执行微任务
}
```



5、 手动执行一个被遗弃的 API：[process._tickCallback()](https://nodejs.org/dist/latest-v18.x/docs/api/deprecations.html#DEP0134) ，里面会执行 `runNextTicks()`，进而执行微任务（无需关心此废弃API）



# 3 process.nextTick()

`process.nextTick()` 并不属于事件循环内的概念，它是 Node.js 中的 `Tick` 概念。像 `Timeout`、`Immediate` 这些时序相关 API 会在不同定时实例执行的间隔处模拟出一个 `Tick`，就是通过 `runNextTicks()` 来执行的；而 `process.nextTick()` 中的回调函数其中一种可能就是会在该处触发。



## 3.1 process.nextTick() 源码

`process.nextTick()` 函数是在 Node.js 启动过程中把 `nextTick()` [挂载到 process 对象](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/bootstrap/node.js#L343)上的

```js
// node-18.15.0/lib/internal/bootstrap/node.js
const { nextTick, runNextTicks } = setupTaskQueue();
process.nextTick = nextTick;
```

[nextTick()](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/process/task_queues.js#L103-L134) 的源码如下

```js
// node-18.15.0/lib/internal/process/task_queues.js
const queue = new FixedQueue();
...

function nextTick(callback) {
  validateFunction(callback, 'callback');

  if (process._exiting)
    return;

  // 处理回调的参数
  let args;
  switch (arguments.length) {
    case 1: break;
    case 2: args = [arguments[1]]; break;
    case 3: args = [arguments[1], arguments[2]]; break;
    case 4: args = [arguments[1], arguments[2], arguments[3]]; break;
    default:
      args = new Array(arguments.length - 1);
      for (let i = 1; i < arguments.length; i++)
        args[i - 1] = arguments[i];
  }

  if (queue.isEmpty())
    setHasTickScheduled(true); // 队列为空，设置tick中有回调标识
  
  ...
  const tickObject = {
    ...
    callback, // 回调函数
    args // 回调函数的参数
  };
  ...
  queue.push(tickObject); // 把tick对象放入队列
}

// 存储nextTick的队列
module.exports = class FixedQueue {
  constructor() {
    this.head = this.tail = new FixedCircularBuffer();
  }

  isEmpty() {
    return this.head.isEmpty();
  }

  push(data) {
    if (this.head.isFull()) {
      // Head is full: Creates a new queue, sets the old queue's `.next` to it,
      // and sets it as the new main queue.
      this.head = this.head.next = new FixedCircularBuffer();
    }
    this.head.push(data);
  }

  shift() {
    const tail = this.tail;
    const next = tail.shift();
    if (tail.isEmpty() && tail.next !== null) {
      // If there is another queue, it forms the new tail.
      this.tail = tail.next;
      tail.next = null;
    }
    return next;
  }
};
```

主要逻辑

* `nextTick()` 用了一个 `queue` 去存储 Node.js 的`当前 Tick` 中需要执行的 `process.nextTick()` 回调们

  > `queue` 是一个由一个或多个环形队列拼接而成的链表，可以把它当做是一个大链表

* 若该 `queue` 为空，则通过 `setHasTickScheduled(true)`，设置一个标识，即 `tickInfo[kHasTickScheduled]` 为1，这个标识表示`“Tick 中有回调”`

  > 该设置原理与 `setTimeout()` 的  `timeoutInfo[0]` 类似，都是一个可直通 `js` 侧与 ` C++`  侧的 `Int32Array`，然后此处以 `0`表示"无""，以 `1` 表示"有"，也是存在 `Environment` 的对应成员变量中

* 设置完该标识位后，构建一个 `Tick` 的回调函数相关对象，将其插入 `queue `大链表



## 3.2 process.nextTick() 的执行

### 3.2.1  processTicksAndRejections()

`process.nextTick()` 的回调是在 [`processTicksAndRejections()`](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/process/task_queues.js#L67-L99) 中执行的，其源码如下：

```js
// node-18.15.0/lib/internal/process/task_queues.js
function processTicksAndRejections() {
  let tock;
  
  do { 
    // while 循环拿队列的tick对象，拿到回调函数callback并执行
    while ((tock = queue.shift()) !== null) {
      ...
      try {
        const callback = tock.callback; // 拿到process.nextTick的回调函数
        if (tock.args === undefined) {
          callback(); // 执行回调函数
        } else {
          const args = tock.args;
          switch (args.length) {
            case 1: callback(args[0]); break;
            case 2: callback(args[0], args[1]); break;
            case 3: callback(args[0], args[1], args[2]); break;
            case 4: callback(args[0], args[1], args[2], args[3]); break;
            default: callback(...args);
          }
        }
      } finally {
        ...
      }

      ...
    }
    
    // while执行完后，执行微任务，可能又产生process.nextTick，所以需要有外层循环
    runMicrotasks();
        
  } while (!queue.isEmpty() || processPromiseRejections()); 
      
  // 当外层大循环也结束后，开始扫尾，比如将 `hasTickScheduled` 等设为 `false`
  setHasTickScheduled(false);
  setHasRejectionToWarn(false);
}
```

主要逻辑

* 每个 `Tick` 里面，都逐个从 `queue` 大链表中拿 `Tick 回调函数相关对象`，直到拿完。每拿一个，都去跑一遍它的 `callback` (即 `process.nextTick()` 中传入的函数)

* 当 `queue` 都取完且执行完后，就是执行微任务的时机了 `runMicrotasks()`
* `while (!queue.isEmpty() || processPromiseRejections())` 外层循环的作用：因为 `runMicrotasks()` 执行完后，可能 `queue` 中又插入了`process.nextTic()`，或者 `Promise` 有 `Rejection`，这个时候需要再跑一遍 `queue` 链表，然后再执行一遍微任务



在 Node-v18.15.0 中，`processTicksAndRejections()` 被调用的地方主要有2处，即前面微任务 `queueMicrotask()` 的章节中，提到两个执行时机：

1.  `setTimeout()`、`setInterval()`、`setImmediate()` 等时序 API ，每次执行 [`runNextTicks()`](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/process/task_queues.js#L58-L65) 方法时
2. Node.js 内部一些 `callback` 函数执行之后也会执行微任务。因为在 `InternalMakeCallback()`中，它会在里面创建一个 [`InternalCallbackScope`](https://github.com/nodejs/node/blob/v18.15.0/src/api/callback.cc#L193) 实例，该实例在结束的时候([`InternalCallbackScope::Close()`](https://github.com/nodejs/node/blob/v18.15.0/src/api/callback.cc#L132-L136))，会执行微任务



### 3.2.2 执行时机1：runNextTick()

第一个执行时机是 `setTimeout()`、`setInterval()`、`setImmediate()` 等时序API在执行时，会执行 `runNextTicks()` ，里面会调用 `processTicksAndRejections()`，它里面会执行 `process.nextTick()`。



**`runNextTicks()`源码:**

```js
// node-18.15.0/lib/internal/process/task_queues.js
function runNextTicks() {
  // 队列中没有nextTick回调，则执行微任务
  if (!hasTickScheduled() && !hasRejectionToWarn())
    runMicrotasks(); // 跑微任务，可能里面会产生process.nextTick()
  
  // 注意，这个条件跟上面的一样的
  if (!hasTickScheduled() && !hasRejectionToWarn())
    return;

  // 有nextTick回调，则执行processTicksAndRejections
  // 里面会执行process.nextTick的任务，里面也会跑微任务
  processTicksAndRejections();
}

// 表示有process.nextTick()
function hasTickScheduled() {
  return tickInfo[kHasTickScheduled] === 1;
}

function setHasTickScheduled(value) {
  tickInfo[kHasTickScheduled] = value ? 1 : 0;
}
```

主要逻辑

* 如果队列中没有 `nextTick `回调函数，则直接跑微任务

* 第2个 `if(!hasTickScheduled() && !hasRejectionToWarn())`，跟前面的判断条件是一样的，若符合则直接返回

  > 因为可能第1个条件在跑微任务时，里面有执行 `process.nextTick()`，这个时候跑完微任务后，`hasTickScheduled()` 可不再是 `false` 了，所以需要第2次判断

* 最后执行 `processTicksAndRejections()` ，它里面也执行 `process.nextTime()`， 也会跑微任务`runMicrotasks()`



### 3.2.3 执行时机2：每次执行完 js 回调时

第2个执行时机是 在每次执行完 js回调时，比如文件系统、网络 I/O 等等。



Node.js 在启动时，调用了 `setupTaskQueue（）`，里面会将 `processTicksAndRejections` 函数[注册为 Tick Callback](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/process/task_queues.js#L166)，该 `Callback` 被存在 `Environment` 中。

```js
// node-18.15.0/lib/internal/process/task_queues.js
module.exports = {
  setupTaskQueue() {
    ...
    // Sets the callback to be run in every tick.
    setTickCallback(processTicksAndRejections); // 将processTicksAndRejections函数注册微Tick Callback
    ...
  },
  ...
};
```

`setTickCallback()` 对应 `node_task_queue.cc` 的[`setTickCallback()`](https://github.com/nodejs/node/blob/v18.15.0/src/node_task_queue.cc#L178-L182)

```c++
// node-18.15.0/src/node_task_queue.cc
static void SetTickCallback(const FunctionCallbackInfo<Value>& args) {
  Environment* env = Environment::GetCurrent(args);
  CHECK(args[0]->IsFunction());
  env->set_tick_callback_function(args[0].As<Function>()); // 设置为env的tick_callback_function函数，即processTicksAndRejections函数
}
```



而执行 js 回调的函数是 [`MakeCallback`](https://github.com/nodejs/node/blob/v18.15.0/src/async_wrap.cc#L657-L672)，每次执行完 js 回调时，就会执行 `InternalCallbackScope` 的 `Close` 方法，[`Close()`](https://github.com/nodejs/node/blob/v18.15.0/src/api/callback.cc#L96-L164) 里会执行 `processTicksAndRejections()`

```c++
// node-18.15.0/src/async_wrap.cc
MaybeLocal<Value> AsyncWrap::MakeCallback(const Local<Function> cb,
                                          int argc,
                                          Local<Value>* argv) {
  ...
  // 调用 InternalMakeCallback
  MaybeLocal<Value> ret = InternalMakeCallback(
      env(), object(), object(), cb, argc, argv, context);

  ...
}

// node-18.15.0/src/api/callback.cc
MaybeLocal<Value> InternalMakeCallback(Environment* env,
                                       Local<Object> resource,
                                       Local<Object> recv,
                                       const Local<Function> callback,
                                       int argc,
                                       Local<Value> argv[],
                                       async_context asyncContext) {
    ...
      
    InternalCallbackScope scope(env, resource, asyncContext, flags);

    ...
    // 执行用户层 JS 回调  
    
     
    // 调用 InternalCallbackScope 的 Close方法
    scope.Close();

    ...

}

// node-18.15.0/src/api/callback.cc
void InternalCallbackScope::Close() {
  ...
    
  // 跟 runNextTicks方法类似，没有nextTick任务，则执行微任务
  if (!tick_info->has_tick_scheduled()) {
    context->GetMicrotaskQueue()->PerformCheckpoint(isolate);

    perform_stopping_check();
  }
  ...
    
  if (!tick_info->has_tick_scheduled() && !tick_info->has_rejection_to_warn()) {
    return;
  }
  
  ...
  
  Local<Function> tick_callback = env_->tick_callback_function();

  ...
    
  // 执行env的tick_callback_function函数，即执行processTicksAndRejections函数(执行process.nextTick)
  if (tick_callback->Call(context, process, 0, nullptr).IsEmpty()) {
    failed_ = true;
  }
  
  ...
}
```

`InternalCallbackScope::Close` 里的逻辑，基本上等同于上面的 `runNextTicks()` 方法：都是如果没有 Tick 任务，那就只执行微任务，否则就执行 `processTicksAndRejections()`。



## 3.3 process.nextTick() 与微任务的关系

**`process.nextTick() `与微任务的关系是：**

`process.nextTick()` 在上面两种执行时机触发后（ `runNextTicks()` 函数和 `InternalCallbackScope::Close()` 函数），可若无 `process.nextTick()` 回调，也必定会先执行微任务（`runMicrotasks()`）；若有 `process.nextTick()` 回调，则先执行 `process.nextTick()` 回调，然后再执行微任务，若期间有新的 `process.nextTick()` 回调插入，那么就继续执行 `process.nextTick()`  回调，简单来说会先执行 `process.nextTick()`，直到没有 `process.nextTick()` 为止



回到上面讲过的 `processTicksAndRejections()` 的逻辑：从中可知，`processTicksAndRejections()` 由两层 `while`循环实现，这里是有可能出现死循环的，当在 `process.nextTick()` 的回调里，又把该 `process.nextTick()` 插入微任务里，就会导致死循环。例如下面这段代码就永远执行不到 `setTimeout()` 的回调

```js
function next() {
  process.nextTick(() => {
    queueMicrotask(next); // queueMicrotask加一个微任务，即微任务里又产生process.nextTick
  });
}

next();

setTimeout(() => {
  console.log('timeout');
  process.exit(0);
}, 100);
```



根据上面死循环的代码，可以出来几个变种：

第1种：死循环

```js
function next() {
  process.nextTick(() => {
    Promise.reject().catch(next);
  });
}

next();

setTimeout(() => {
  console.log('timeout');
  process.exit(0);
}, 100);
```

第2种：一直执行nextTick，死循环

```js
function next() {
  process.nextTick(next);
}

next();

setTimeout(() => {
  console.log('timeout');
  process.exit(0);
}, 100);
```

第3种：不会死循环，输出 `timeout`

```js
function next() {
  process.nextTick(() => {
    setImmediate(next); // setImmediate不会阻塞
  });
}

next();
setTimeout(() => {
  console.log('timeout');
  process.exit(0);
}, 100);
```

第4种：死循环

```js
function next() {
  // 微任务里有nextTick
  queueMicrotask(() => {
    process.nextTick(next);
  });
}

next();
setTimeout(() => {
  console.log('timeout');
  process.exit(0);
}, 100);
```



# 4 总结

文中讲解了 Node.js 的 3 个产生异步任务的 API，分别是` setImmediate()`、`queueMicrotask()`、`process.nextTick()`，以及它们的原理和实现

* `setImmediate()`：主要讲了它和事件循环中空转事件的关系，它的启动执行时机

* `queueMicrotask()`：主要讲了Node.js 中的5钟执行时机埋点

* `process.nextTick()`：主要讲了它的执行时机 和 它与微任务的关系



# 5 参考

* [Node.js v18.15.0源码](https://github.com/nodejs/node/tree/v18.15.0)
* [死月-趣学Node.js](https://juejin.cn/book/7196627546253819916/section/7197301858296135684)
* [theanarkh-深入剖析 Node.js 底层原理](https://juejin.cn/book/7171733571638738952/section/7176118481966858297)
* [libuv api文档](http://docs.libuv.org/en/stable/api.html)

