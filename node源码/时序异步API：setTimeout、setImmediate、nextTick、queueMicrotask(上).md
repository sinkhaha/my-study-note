> 本文 Node.js 的版本 v18.15.0 、libuv 的版本 v1.44.2 (libuv 相关只涉及 unix 平台)

# 1 定时器 setTimeout()

## 1.1 setTimeout() 的实现

`setTimeout()` 是与 `I/O` 无关的异步 API，`setTimeout()` 并不是 `ECMAScript` 规范 的 API，而是 [Web API](https://html.spec.whatwg.org/multipage/timers-and-user-prompts.html#dom-settimeout-dev)，V8 等 js 引擎不会实现 `setTimeout()`。不同的运行时，`setTimeout()` 的实现机制是不一样的。



Node.js 中 `setTimeout()` 的实现，主要由两部分组成， js 侧提供的定时器的调度管理 和 ` libuv` 侧提供的定时器的实际执行（基于 `libuv` 的 `uv_timer_t(uv_timer_s)` 句柄，`uv_timer_t` 是 `uv_timer_s` 的别名）

> js 侧是定时器的调度管理层，`libuv` 侧是实际的定时器执行层，而 C++ 侧则是粘合层(负责 js跟 `libuv` 之间的交互)



`setTimeout()`的执行时机：是在 [Node.js 事件循环](https://github.com/nodejs/node/blob/v18.15.0/src/api/embed_helpers.cc#L37)（基于 `libuv` 的事件循环）的 `定时器阶段` 执行（[ libuv 的`uv__run_timers` 方法](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/src/unix/core.c#L393)）

> 注意： `setInterval()` 的执行机制跟 `setTimeout()` 完全相同，`setInterval()` 跟 `setTimeout()` 的区别只是 `setInterval()`  在 `Timeout` 实例对象中多了个一个“是否循环”字段控制是否需要重复执行而已



## 1.2 setTimeout() 源码

Node.js 中 `setTimeout()` 的实现并不完全按规范来实现

> 比如，在规范中 `setTimeout()` 的返回值 `id` 是一个整数，而 Node.js 实际上返回的是一个 [`Timeout类`](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/timers.js#L166) 的实例对象，`Timeout` 类主要用于控制定时器并触发回调函数。



Node.js 中 [`setTimeout()`](https://github.com/nodejs/node/blob/v18.15.0/lib/timers.js#L140-L168) 的源码如下：

```js
// node-18.15.0/lib/timer.js
function setTimeout(callback, after, arg1, arg2, arg3) {
  validateCallback(callback);

  // 1、先处理回调函数中参数个数
  let i, args;
  switch (arguments.length) {
    // fast cases
    case 1:
    case 2:
      break;
    case 3:
      args = [arg1];
      break;
    case 4:
      args = [arg1, arg2];
      break;
    default:
      args = [arg1, arg2, arg3];
      for (i = 5; i < arguments.length; i++) {
        // Extend array dynamically, makes .apply run much faster in v6.0.0
        args[i - 2] = arguments[i];
      }
      break;
  }

  // 2、实例化Timeout 
  // 注意：第4个参数isRepeat表示是否重复执行，这里是跟setInterval()的区别，setInterval()对应值为true
  const timeout = new Timeout(callback, after, args, false, true);
  
  // 3、把Timeout实例插入Map和优先队列中
  insert(timeout, timeout._idleTimeout);

  return timeout; // 返回 Timeout实例
}
```

主要逻辑如下：

* 处理回调函数中的参数：`args` 是回调函数的参数，因为一般不会超过3个参数，所以这里显示声明了3个 `(arg1, arg2, arg3)` ；如果参数超过3个，则动态扩展 `args` 数组

* 生成 `Timeout` 实例

* `Insert()` 把 `Timeout` 实例 插入 `Map` 和优先队列中

* 返回 `Timeout` 实例



### 1.2.1 Timeout 管理类

[`Timeout`](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/timers.js#L166-L242) 类主要用于控制定时器并触发回调函数，存储了定时器的一些元数据，如起始时间、过期间隔、过期回调函数等，代码如下

```js
// node-18.15.0/lib/internal/timers.js
class Timeout {
  constructor(callback, after, args, isRepeat, isRefed) {
    
    // setTimeout超时时间没传或者为0时，则默认为1
    after *= 1; // Coalesce to number or NaN
    if (!(after >= 1 && after <= TIMEOUT_MAX)) {
      if (after > TIMEOUT_MAX) {
        process.emitWarning(`${after} does not fit into` +
                            ' a 32-bit signed integer.' +
                            '\nTimeout duration was set to 1.',
                            'TimeoutOverflowWarning');
      }
      after = 1; // Schedule on next tick, follows browser behavior
    }

    this._idleTimeout = after; // 超时时间
    this._idlePrev = this;
    this._idleNext = this;
    this._idleStart = null; // 定时器开始时间
    // This must be set to null first to avoid function tracking
    // on the hidden class, revisit in V8 versions after 6.2
    this._onTimeout = null;
    this._onTimeout = callback; // 超时的回调函数，具体在listOnTimeout方法中执行
    this._timerArgs = args;
    this._repeat = isRepeat ? after : null; // 是否重复触发，用于setInterval方法
    this._destroyed = false;

    if (isRefed)
      incRefCount(); // 激活libuv的定时器，说明有定时器需要处理  

    this[kRefed] = isRefed; // 标记状态

    ...
  }
}
```

主要逻辑如下：

1. 最开始是判断过期间隔的合法性，如果时间不合法，则强行将过期时间改为 1

2. 将各种元数据信息放到成员变量中（如超时时间、定时器开始时间、过期回调函数等）

   *  `_onTimeout` 变量就是 `setTimeout()` 时候传进来的回调函数，具体在 [`listOnTimeout()`](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/timers.js#L568-L571) 方法中被调用

   * `isRepeat` 表示 `Timeout` 是否重复执行； `setTimeout()` 将该值是 `false`，即不重复；而 `setInterval()` 该值是 `true`，即重复

3. `incRefCount()` ：用于激活 `libuv` 的定时器



### 1.2.2 Map 与 优先队列

`setTimeout()` 里的 [`insert()`](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/timers.js#L360-L380) 的作用：就是把新生成的 `Timeout` 实例插入 js 侧自己维护的 `Map` 和优先队列中



**`Map` 和优先队列**

1. [Map](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/timers.js#L157)：键名是超时时间（即 `setTimeout()` 的第2个参数），键值为一条 [`TimersList` 链表](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/timers.js#L244-L264)。链表的元素是 `Timeout` 定时器实例，所以在这条链表中，所有 `Timeout` 实例的 `timeout` 值都是相同的，链表中还额外存了一个这条链表最近的超时时间 `expiry`

   > 因为同一条链表所有元素的 `timeout` 超时时间相同，且 `Timeout` 都是按时序插入到链表，所以在较后面时间插入的一定比前面时间插入的晚超时，这其实是一个有序链表，顺序是按超时时间点从早到晚

2. [优先队列](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/priority_queue.js#L13)：[队列的元素是 `TimersList` 链表](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/timers.js#L371)，这是一个小顶堆，以这条链表的`最近超时时间 expiry`为权重，所以堆顶是最近的超时时间的链表

   

`Map` 和优先队列的结构如下图所示：

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/%E5%AE%9A%E6%97%B6%E5%99%A8.png)

从图中可以看出：

* 有过期时间分别为 10ms、50ms、100ms 的定时器。10ms 的有 3个定时器，50ms 的有2个定时器，100ms 的有2个定时器

* 10ms 定时器的最近过期时间为 1680700421783
* 50ms 定时器的最近过期时间为 1680700541108
* 100ms 定时器的最近过期时间为 1680700548601

因为优先队列中，以 `最近超时时间 expiry` 为权重，所以优先队列的顺序就是 10ms 的链表、50ms 链表、100ms 链表



**`insert()` 源码解析**

```js
// node-18.15.0/lib/internal/timers.js
function insert(item, msecs, start = getLibuvNow()) {
  // Truncate so that accuracy of sub-millisecond timers is not assumed.
  msecs = MathTrunc(msecs); // 去掉小数，保留整数部分
  item._idleStart = start; // 当前定时器的开始时间

  // Use an existing list if there is one, otherwise we need to make a new one.
  let list = timerListMap[msecs];
  if (list === undefined) {
    debug('no %d list was found in insert, creating a new one', msecs);
    const expiry = start + msecs; // 计算该链表的最近超时时间
    timerListMap[msecs] = list = new TimersList(expiry, msecs); // 链表存入map，key是超时时间
    timerListQueue.insert(list); // 链表插入优先队列

    if (nextExpiry > expiry) { // nextExpiry 存的是所有Timeout实例中，最近要过期的时间
      scheduleTimer(msecs); // 启动定时器，调用了libuv的uv_timer_start方法
      nextExpiry = expiry;
    }
  }

  L.append(list, item);
}
```

主要逻辑如下

1. `MathTrunc`：截断超时时间，去掉小数，只保留整数部分

2. `_idleStart`：记录当前定时器的开始时间

3. 判断是否需要实例化链表并存入 `Map`：从 `Map` 获取当前超时时间对应的链表，链表如果不存在，则计算过期时间点 `expiry` 并实例化一个新链表，这个过期时间就是这整条链表的 `最近超时时间` 了，然后新链表存入 `Map` 中，同时插入优先队列；最后判断是否需要启动定时器，通过 `scheduleTimer()` 启动定时器

4. 最后把当前定时器插入到链表的最后面



## 1.3 定时器的启动和执行

### 1.3.1 scheduleTimer() 启动定时器

Node.js 中，针对 `setTimeout()` 的单个定时器被存在 [`Environment` 类](https://github.com/nodejs/node/blob/v18.15.0/src/env.h#L527) 中。`Environment` 是针对 Node.js 中每个 V8 的 `Isolate` 都存在一份的环境相关类，`setTimeout()` 也是只需要一份；`Environment` 的 `timer_handle()` 会返回一个定时器句柄对象，底层定时器的执行就是基于此句柄。

```c++
// node-18.15.0/src/env.h
class Environment : public ... {
  ...
  inline uv_timer_t* timer_handle() { return &timer_handle_; } // 返回一个定时器句柄对象
  uv_timer_t timer_handle_;
};
```

在前面 `setTimeout()` 的 `insert()` 源码中，把 `Timeout` 实例插入链表时，有[如下代码](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/timers.js#L373-L376)：

```js
if (nextExpiry > expiry) { // 如果有一个最近要过期的时间 比 全部的Timeout的过期时间都要小
  scheduleTimer(msecs); // 触发定时器的启动
  nextExpiry = expiry; // 替换掉所有定时器最近要过期的时间点
}
```

* 这里 `scheduleTimer()` 的作用：就是开始定时器的调度，实际最终调用的是 `libuv` 的 `uv_timer_start()`。

* 全局变量 `nextExpiry` ，存的是所有链表的所有 `Timeout` 定时器实例中，最近要过期的那个时间点。



`scheduleTimer()` 方法如下，可知是由内置模块 `timers.cc` 提供的

```js
// node-18.15.0/lib/internal/timers.js
const {
  scheduleTimer, // 内置模块timers里的ScheduleTimer方法
} = internalBinding('timers');
```

> `internalBinding` 是加载 `timers` 内置模块，此处对应 `src/timers.cc` 文件（Node.js 注册内置模块讲解可参考[这里](https://zhuanlan.zhihu.com/p/580904505)）

`timers.cc` 的 [`ScheduleTimer()`](https://github.com/nodejs/node/blob/v18.15.0/src/timers.cc#L32) 如下，可知调用了 `env` 的 `ScheduleTimer()`

```c++
// node-18.15.0/src/timers.cc
void ScheduleTimer(const FunctionCallbackInfo<Value>& args) {
  auto env = Environment::GetCurrent(args);
  // 调用Environment的ScheduleTimer方法
  env->ScheduleTimer(args[0]->IntegerValue(env->context()).FromJust());
}
```

`env.cc` 里的 [`ScheduleTimer()`](https://github.com/nodejs/node/blob/v18.15.0/src/env.cc#L1155)如下，可知调用了 `libuv` 的 `uv_timer_start()`

```c++
// node-18.15.0/src/env.cc
void Environment::ScheduleTimer(int64_t duration_ms) {
  if (started_cleanup_) return;
  // libuv的uv_timer_start方法，这里注入了RunTimers方法作为 libuv定时器执行完成后会调用的回调
  uv_timer_start(timer_handle(), RunTimers, duration_ms, 0);
}
```

`libuv` 的 [`uv_timer_start()`](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/src/timer.c#L67) 如下，如果 `libuv` 的定时器未激活过则会激活，然后将定时器的触发时间改成传进来的最新时间并激活

```c
// node-18.15.0/deps/uv/src/timer.c
int uv_timer_start(uv_timer_t* handle,
                   uv_timer_cb cb,
                   uint64_t timeout,
                   uint64_t repeat) {
  ...
  // 启动定时器
  uv__handle_start(handle);
  ...
}
```



### 1.3.2 libuv 侧：就一个定时器

在 js 侧，调用一次 `setTimeout()` 就生成一个 `Timeout` 实例，而且 `Timeout` 实例被 `Map` 和优先队列管理起来；但实际上，这只是 `js` 侧的类，并不实际参与调度，只是在真正 `libuv` 定时器调度时，被引用一下而已。真正在 ` libuv` 层，定时器就一个。



`libuv` 中，一个定时器是以一个定时器句柄 `uv_timer_t(也叫 uv_timer_s)` 结构体的形式进行操作的，也是用小顶堆管理定时器。



注意：Node.js 并不是每次调  `setTimeout() ` 的时候都往小顶堆插入一个节点，因为这样会引起 js 侧和 C++ 、C 侧频繁通信，损耗性能。因此在 Node.js 中，只有一个关于 ` uv_timer_t(uv_timer_s)` 的 `handle`，Node.js 在 js 侧维护了 `Map` 和优先队列，每次计算出最快到期节点的时间，然后修改 `libuv uv_timer_s handle` 的超时时间。



`libuv ` 中 [`uv_timer_start() `](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/src/timer.c#L67-L95) 的作用：就是是在 `libuv` 事件循环中启动一个定时器。代码如下

```c
// node-18.15.0/deps/uv/src/timer.c
int uv_timer_start(uv_timer_t* handle,
                   uv_timer_cb cb, // 超时之后的回调函数
                   uint64_t timeout, // 超时时间
                   uint64_t repeat) {
  uint64_t clamped_timeout;

  if (uv__is_closing(handle) || cb == NULL)
    return UV_EINVAL;

  // 在启动定时器前，把之前的先停到
  if (uv__is_active(handle))
    uv_timer_stop(handle);

  // 计算超时时间 事件循环的当前时间+下一个超时时间
  clamped_timeout = handle->loop->time + timeout;
  if (clamped_timeout < timeout)
    clamped_timeout = (uint64_t) -1;

  // 初始化参数
  handle->timer_cb = cb; // 回调函数，在Node.js中其实就是 Environment的RunTimers方法
  handle->timeout = clamped_timeout; // 超时时间
  handle->repeat = repeat; // 是否重复运行
  /* start_id is the second index to be compared in timer_less_than() */
  handle->start_id = handle->loop->timer_counter++; // 赋予一个唯一的自增id

  // 插入小顶堆，里面会根据该节点的超时时间动态调整小顶堆
  heap_insert(timer_heap(handle->loop),
              (struct heap_node*) &handle->heap_node,
              timer_less_than);
  // 启动定时器
  uv__handle_start(handle);

  return 0;
}
```

方法的 `repeat` 参数代表“是否循环”，即触发一次之后，是否等 `timeout` 时间到了会再次触发。

> 注意：该参数看起来可以用于区分 `setTimeout()` 和 `setInterval()`。不过 Node.js 中的 `setInterval() `并没用使用 `repeat` 参数，而是执行逻辑跟 `setTimeout()` 一样，在执行完一次后，又重新把 `Timeout` 实例插入链表中继续等待下一次执行。



定时器启动后，在 `libuv` 事件循环的定时器阶段（[调用了 `uv__run_timers()` 方法](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/src/unix/core.c#L393)）中会判断有没有定时器超时，有则执行传入的 `cb回调函数`，在Node.js 中此回调函数实际是 `Environment`的`RunTimers()` 方法。代码如下

```c
// node-18.15.0/deps/uv/src/timer.c
// 找出小顶堆中超时的定时器节点，并执行里面的回调
void uv__run_timers(uv_loop_t* loop) {
  struct heap_node* heap_node;
  uv_timer_t* handle;

  for (;;) {
    // 找出小顶堆的超时节点，即堆顶节点
    heap_node = heap_min(timer_heap(loop));
    if (heap_node == NULL)
      break;

    handle = container_of(heap_node, uv_timer_t, heap_node);
    // 当前堆顶节点时间大于当前时间，说明后面的节点也没有超时
    if (handle->timeout > loop->time)
      break;
    
    // 移除计时器节点
    uv_timer_stop(handle);
    // 如果设置了 repeat 则重新插入小顶堆，等待下次超时时执行
    uv_timer_again(handle);
    // 执行定时器的回调，把handle传到C++层Environment的RunTimers方法
    handle->timer_cb(handle);
  }
}
```



### 1.3.3 libuv 侧回调函数对应的C++侧函数：RunTimers

一旦定时器被触发（ `Environment` 类的 `timer_handle_` 定时器），执行的始终是 `uv_timer_start()` 方法传进去的 `RunTimers()`  回调函数。



 [`RunTimers()`](https://github.com/nodejs/node/blob/v18.15.0/src/env.cc#L1170-L1228) 的作用：是经过一系列逻辑后，调用 js 侧的定时回调函数，在那个函数中才会去找对应的 `Timeout` 并触发对应的真实回调函数，即该 `RunTimers()` 最终达到的是一个 `Dispatcher` 的效果。代码如下：

```c++
// node-18.15.0/src/env.cc
void Environment::RunTimers(uv_timer_t* handle) {
  // 因为RunTimers是静态方法，所以无法通过this来表示Environment实例，此处用了 Environment::from_timer_handle() 静态方法来通过对应 uv_timer_t 获取其所属的 Environment实例
  Environment* env = Environment::from_timer_handle(handle); 
  ...

  Local<Object> process = env->process_object(); // 获取process对象
  ...

  // 获取一个事先注册到 Environment 中的 js 侧的定时器回调函数
  // 这里是env里的timers_callback_function函数，在Node.js中是processTimers()
  Local<Function> cb = env->timers_callback_function(); 
  
  MaybeLocal<Value> ret;
  Local<Value> arg = env->GetNow();
  do {
    TryCatchScope try_catch(env);
    try_catch.SetVerbose(true);
    
    // 执行回调函数，对应js层的processTimers函数，该函数的返回值是下一次要执行的定时器的过期时间；可能是正数，也可能是负数
    ret = cb->Call(env->context(), process, 1, &arg); 
  } while (ret.IsEmpty() && ...);

  ...

  //  从js层拿到的下一次超时时间，下面会调用ScheduleTimer重启定时器
  int64_t expiry_ms =
      ret.ToLocalChecked()->IntegerValue(env->context()).FromJust();

  uv_handle_t* h = reinterpret_cast<uv_handle_t*>(handle);
  if (expiry_ms != 0) { // 不等于0说明还有定时器要执行，只是还没到过期时间
    // 计算下一次真正要触发的时间，这里只用expiry_ms的绝对值
    int64_t duration_ms =
        llabs(expiry_ms) - (uv_now(env->event_loop()) - env->timer_base());

    // 重启定时器
    env->ScheduleTimer(duration_ms > 0 ? duration_ms : 1);

    // 这里根据正负判断是ref还是unref，用于判断是否需要对 libuv 进行 Reference 或者 Unreference 操作，以让 Node.js 决定需不需要“及时退出”
    if (expiry_ms > 0)
      uv_ref(h);
    else
      uv_unref(h);
  } else { // 说明没有定时器了
    uv_unref(h);****
  }
}
```

主要逻辑如下：

* 参数规定是当前触发当前定时的 `uv_timer_t` 句柄。因为该函数是可以以简单函数指针的形式传给 `uv_timer_start()`，所以它是一个 `static` 静态方法。所以在内部无法通过 `this` 来表示 `Environment` 实例，此处用了 `Environment::from_timer_handle()` 静态方法来通过对应 `uv_timer_t` 获取其所属的 `Environment` 实例

* 该方法的主要逻辑就是拿到一个  `cb` 函数( `env->timers_callback_function()`），该回调函数实际是 js 侧的 `processTimers `函数），然后执行该回调函数，传入的参数为当前时间，并且 `this` 对象为 `process`

* 回调函数的返回值是由 `js` 侧的 `processTimers` 函数计算出来的下一次定时器触发时间。若触发时间为 `0`，则说明之后没有 `Timeout` 定时器了，就不需要重新调度；若非 `0`，则通过一定逻辑计算出真正下一次触发所需的时刻，并通过调用 `env->ScheduleTimer()` 函数重启定时器

  > 这里回调函数的返回值可能有正负(具体原因在下面的 `timeout.ref` 和 `timeout.unref` 讲解)



### 1.3.4 C++ 侧回调函数对应的 js 侧函数：processTimers

从上面我们知道在 ` Environment::RunTimers` 方法里会取得回调函数并执行，此时回调函数是 `env` 的`timers_callback_function()` 函数，下面来看下这个回调函数是什么？



在Node.js的启动过程中，会[注册 `processTimers()` 函数](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/bootstrap/node.js#L350-L361)为 `env` 的  `timers_callback_function()` 。源码如下

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
  setupTimers(processImmediate, processTimers);
```

`setupTimers` 是内置模块 `timers.cc` 的方法，如下可以看出 `setupTimers` 注册了两个个函数，分别是 `processImmediate()` 和 `processTimers()`

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

所以 `env` 的 `timers_callback_function()` 函数就是 `js` 层的 `processTimers()` 函数



**processTimers()的逻辑**

Node.js 中 [`processTimers` 的源码](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/timers.js#L497-L515)如下

```js
// node-18.15.0/lib/internal/timers.js
// 该函数返回值是下一次过期时间，可能是正数，也可能是负数，也可能是0
function processTimers(now) {
  nextExpiry = Infinity;

  let list;
  let ranAtLeastOneList = false;
  
  // 循环执行到没有定时器了，或者没有即将要执行的定时器为止
  while ((list = timerListQueue.peek()) != null) {
    if (list.expiry > now) { // 链表过期时间大于当前时间，表示没有过期的定时器要执行
      nextExpiry = list.expiry;
      return timeoutInfo[0] > 0 ? nextExpiry : -nextExpiry; // 注意正负值的含义
    }
    
    // 接下去执行链表过期时间小于等于当前时间的逻辑
    
    if (ranAtLeastOneList)
      runNextTicks();
    else
      ranAtLeastOneList = true;
    
    listOnTimeout(list, now); // 执行定时器
  }
  
  return 0; // 等于0说明没有定时器要执行了
}
```

主要逻辑如下：

1. 整体逻辑就是，不断从 `timerListQueue` 优先队列中获取队首元素，首元素也就是最近要过期的那条链表
2. 判断一下链表的过期时间是否大于当前时间 `now`。如果大于当前时间，说明这个 `Timeout`还不能执行，于是将 `nextExpiry` 用该链表的过期时间替换。接着返回下一次过期时间，注意这里先判断了 `timeoutInfo[0]` 的大小，并以此返回正负的值。实际上在 C++ 侧的 `RunTimers()` 中用的是该值的绝对值最为下一次定时器要触发的时间；正负只是最为一个判断标识，用于判断是否需要对 `libuv` 进行 `ref` 或者 `unref` 操作，以让 Node.js 决定需不需要“及时退出”。总之，这里的逻辑就是如果还有定时器，但是还没到执行时间，则返回 `nextExpiry` 以让 `RunTimers()` 开启下一轮的 ` libuv` 的定时器
3. 如果链表过期时间小于等于当前时间，则说明在当前状态下，该 `Timeout` 是需要被触发的。由于时间的不精确性，如果时间循环卡了一下，导致一下子过了好几毫秒，而在这之前有好几条链表都会过期，所以就需要在一次 `processTimers` 里面持续执行 `Timeout` 直到获取的 `Timeout` 未过期。所以这里一整套逻辑都是被一个 `while` 所包围
4. 在执行 `Timeout` 之前，先判断一下当前的 `while` 里面是不是已经执行过至少一个 `Timeout` 了。若未执行过，则直接执行；若已经执行过，则在 Node.js 的语义中已经过了 Tick 了（当前这轮的事件循环），接下去的 `Timeout` 应该在下一个 Tick 执行，所以在这里执行一下 `runNextTicks()` 来让 Node.js 同步模拟出一个 Tick 结束的样子。这个 `runNextTicks()` 里面主要做的事情就是去处理微任务、`Promise` 的 `rejection` 等。毕竟在 Node.js 的语义中，一个 Tick 结束要做的事情有好多
5. 到下一个 `Tick` 之后，就可以触发 js 侧的 `Timeout` 了，触发的逻辑是在 `listOnTimeout()` 中
6. 接下去就开始下一条循环，从链表中再获取下一条 `Timeout` 重复上面的操作。如果链表空了，则退出，退出之后在外层循环实际上就是 Node.js 继续从优先队列中获取再继续判断了



简单总结 `processTimers` 的流程就是：从优先队列拿最早的链表，里面元素一个个都执行了，一直做到元素未超时，再把链表放回去重排优先队列；然后再从优先队列拿最早的链表，里面元素一个个都执行了，一直做到元素未超时，再把链表放回去重排优先队列，一直做到优先队列里面第一条链表也未超时为止。



由于 `processTimers` 的实现，在极端情况下会出现一些问题，如下代码：

```js
'use strict';

setTimeout(() => {
  console.log(1);
}, 10);

setTimeout(() => {
  console.log(2);
}, 15);

let now = Date.now();
while (Date.now() - now < 100) {
  // 100ms的循环
}

setTimeout(() => {
  console.log(3);
}, 10);

now = Date.now();
while (Date.now() - now < 100) {
  // 100ms的循环
}
```

按理说，按实际 `Timeout` 触发时间的迟早进行排序触发，3 个 `Timeout` 触发时机分别为 10、15、10，所以顺序应该一次是 1、2、3。但实际这种情况下，触发顺序不再是 1、2、3，而是 1、3、2。因为第 1 是 10ms 超时，肯定是先执行，由于第 3 和 第 1 一样都是10ms 超时，它们在同一个链表中，所以当第1执行后，接下去就是执行3，最后才执行 2。



### 1.3.5 js 侧定时器的执行：listOnTimeout

[`listOnTimeout()` ](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/timers.js#L517-L603) 的作用：就是触发 js 侧定时器的执行。代码如下

```js
// node-18.15.0/lib/internal/timers.js
// list就是某个超时时间的链表; now就是当前 Tick 的时间
function listOnTimeout(list, now) { 
    const msecs = list.msecs;

    let ranAtLeastOneTimer = false;
    let timer;
  
    // L.peek(list)从一个 list 中获取首个元素
    while ((timer = L.peek(list)) != null) {
      const diff = now - timer._idleStart;

      // 还没过期
      // Check if this loop iteration is too early for the next timer.
      // This happens if there are more timers scheduled for later in the list.
      if (diff < msecs) {
        list.expiry = MathMax(timer._idleStart + msecs, now + 1);
        list.id = timerListId++;
        timerListQueue.percolateDown(1); // 重排优先队列
        debug('%d list wait because diff is %d', msecs, diff);
        return;
      }

      // 下面执行过期的逻辑
      
      if (ranAtLeastOneTimer)
        runNextTicks();
      else
        ranAtLeastOneTimer = true;

      // The actual logic for when a timeout happens.
      L.remove(timer); // 把定时器从链表移除

      ...

      let start;
      if (timer._repeat)
        start = getLibuvNow();

      try {
        const args = timer._timerArgs;
        if (args === undefined)
          timer._onTimeout(); // 执行超时回调，回调没有参数
        else
          // 执行超时回调，_onTimeout就是js层定时器要执行的回调函数，此时回调有参数
          ReflectApply(timer._onTimeout, timer, args); 
      } finally {
        if (timer._repeat && timer._idleTimeout !== -1) {
          timer._idleTimeout = timer._repeat;
          insert(timer, timer._idleTimeout, start); // 要重复执行，重新放入链表，setInterval场景
        } else if (...) {
          ...
            if (timer[kRefed])
                timeoutInfo[0]--; // 执行完后，引用数减少1
          ... 
        }
      }
    }
```

主要逻辑如下

* 首先是一个 `while` 通过 `L.peek()` 不断从 `TimersList` 中获取首个元素，首个元素就是最早过期的元素

  > 因为链表所有元素的 `timeout` 超时时间相同，任何一个 `Timeout` 都是按时序插入到，所以在较后面时间插入的一定是前面时间插入的晚超时，这其实是一个有序列表，按超时时间点从早到晚

* 在 `while` 每次循环中，先判断一下拿到的 `Timeout` 实例是否应被触发，即是否过期。如果没有过期，则进入 `if(diff < msecs)` 分支。将当前 `Timeout` 实例对应的过期时间作为当前链表整体的过期时间，并重排优先队列；`timerListQueue.percolateDown(1)` 的意思是：对优先队列第一个元素进行下滤操作。因为这时它的 `expiry` 被修改了，不一定是最早过期的链表了，需要下滤以得到新的最早链表。下滤过后，退出该函数，回到之前的 `processTimers()`，进入下一个循环，即再拿出新的最早过期链表，并判断有没有过期，然后做后续逻辑
* 若是当前 `Timeout` 过期了，即该定时可以被执行，仍然先模拟一次 `Tick`，即走一遍 `runNextTicks()` 的逻辑，然后从链表中将当前 `Timeout` 移除
* 做了一系列额外逻辑后（如操作 `Timeout` 的 `ref` 值等），就是通过 `try-finally` 去执行 `Timeout` 实例的 `_onTimeout()` 方法了， `_onTimeout()` 就是某个 `Timeout` 触发后真正要执行的超时回调函数，就是去执行用户侧传入 `setTimeout()` 的第一个参数 `callback`
* 执行完 `Timeout` 后，先判断该 `Timeout` 是否需要重复执行（即 `setInterval()` 的情况）。若需要重复执行，则将 `Timeout` 改期并通过 `insert()` 重新插入到Map和优先队列中



**`processTimers()` 和 `listOnTimeout` 的流程如下图所示：**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/setTimeout.png)





## 1.4 Timeout 的 ref() 和 unref()

### 1.4.1 含义

从 Node.js 中 `Timeout` 的 [ref()](https://nodejs.org/docs/v18.15.0/api/timers.html#timeoutref) 和 [unref()](https://nodejs.org/docs/v18.15.0/api/timers.html#timeoutunref) 的文档可知：

1. 如果 `Timeout` 有被 “引用”，则在没有其他让事件循环 “长存” 的条件下（如文件 I/O 等待、网络事件等待等），Node.js 的执行生命周期会被 `Timeout` 撑着
2. 如果 `Timeout` 没有被 “引用”，若没有其他 “长存” 条件，Node.js 会执行完当前 `Tick` 后，马上退出，并不会去执行 `Timeout` 


在 libuv 中，就有 `uv_ref()` 和 `uv_unref()` 的概念。如果 `libuv` 中，不存在任何被 `ref` 的句柄，就会退出事件循环，所以 `uv_ref()` 是为了撑起其生命周期用的。



例如下面这段代码，会先输出 `start`，然后在 100 ms 之后，输出 `done`，然后 Node.js 退出。如果把最后的 `timer.unref()` 加上，则 Node.js 会在输出  `start` 后，创建 `Timeout`，然后直接退出，并不会执行 `setTimeout`

```js
console.log('start');

const timer = setTimeout(() => {
  console.log('done');
}, 100);

// timer.unref()
```



### 1.4.2 timeoutInfo[0]

**timeoutInfo[0]的设计**

Node.js 中，用了一种非常规的方式，来做那个唯一的 C++ 侧定时器的 `ref` 与 `unref`。

* 首先，在 Node.js 的 `Environment` 中，有一个 `timeout_info_` 的 `Int32Array` 的 `js TypedArray`。说是数组，实际上它就一个元素，`timeout_info_[0]`。如果我们直接用一个 `Number` 类型，那么在获取它、更改它的时候，要在 js 与 C++ 侧反复横跳，这是有开销的。而根据 V8 的特性，C++ 侧是可以直接访问 `Int32Array` 指定下标的内容，获取的是原生 4 字节的整型内存，而 js 侧也可以直接操作该数组。这么一来，双方对其第 `0` 个元素读取和操作都可直接进行，而不用切换上下文，少了些开销

* 所以，虽然它是个 `Int32Array`，但 Node.js 却 Trick 地将其当作横跳 C++ 侧与 js 侧的一个 `Number` 值。简而言之，你就简单粗暴地当 `timeoutInfo[0]` 是一个 `Number` 值用就好了



**timeoutInfo[0]的作用**

这个 `timeoutInfo[0]` 值的作用：就是记录所有 `Timeout` 实例的 `ref` 与 `unref` 累加的值。当该值为 `0` 时，说明不再有 js 侧的 `Timeout` 需要被 `ref`，那么 `Environment` 中那个唯一的定时器可被 `uv_unref()`；否则就说明还有至少一个 `Timeout` 被 `ref`，生命周期需继续撑着。



在 `Timeout` 的构造函数中，最后一个参数是 `isRefed`。`setTimeout()` 与 `setInterval()` 中传的都是 `true`。如果 `isRefed` 为 `true`，会通过 `incRefCount()` 增加 `ref` 的值。

```js
class Timeout {
  ......
  
  // 
  if (isRefed)
      incRefCount();
  ......
}

function incRefCount() {
  if (timeoutInfo[0]++ === 0)
    toggleTimerRef(true); // c++层的ToggleTimerRef方法
}

function decRefCount() {
  if (--timeoutInfo[0] === 0)
    toggleTimerRef(false);
}
```

`incRefCount()` 里的逻辑就是将 `timeoutInfo[0]` 加一。如果它的值为 `0`，说明之前不存在一个被 `ref` 的 `Timeout`，所以需要调用 `toggleTimerRef(true)` 去 C++ 侧将唯一的定时器给 `ref`。



在 `listOnTimeout()` 函数中，执行 `timer._onTimeout()` 之后，如果它有被 `ref`，则执行完后，[会将 timeoutInfo[0] 减一](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/timers.js#L579-L580)。

```js
function listOnTimeout(list, now) {
  ...
  try {
    
  } finally {
    ...
    
    // 在定时器执行完后，如果有被引用，则timeoutInfo[0]减少1
    if (timer[kRefed])
        timeoutInfo[0]--;
    ...
  }
}
```



### 1.4.3 ref 和 unref

对于一个 `Timeout` 的 `ref` 和 `unref` ，实际上就是[调用 `ref()` 和 `unref()` 这两个函数](https://github.com/nodejs/node/blob/v18.15.0/lib/internal/timers.js#L221-L237)

```js
// node-18.15.0/lib/internal/timers.js
class Timeout {
  ......
  unref() {
    if (this[kRefed]) {
      this[kRefed] = false;
      if (!this._destroyed)
        decRefCount();
    }
    return this;
  }

  ref() {
    if (!this[kRefed]) {
      this[kRefed] = true;
      if (!this._destroyed)
        incRefCount();
    }
    return this;
  }
......
}

function incRefCount() {
  if (timeoutInfo[0]++ === 0)
    toggleTimerRef(true); // c++层的ToggleTimerRef方法
}

function decRefCount() {
  if (--timeoutInfo[0] === 0)
    toggleTimerRef(false); // 去掉ref
}
```

由代码可知，调用的是 C++ 侧的 [`toggleTimerRef()` 函数](https://github.com/nodejs/node/blob/v18.15.0/src/env.cc#L1160-L1168)，它实际又是调用了 `libuv` 的 `uv_ref()` 和 `uv_unref()` 

```c++
// node-18.15.0/src/env.cc
void Environment::ToggleTimerRef(bool ref) {
  if (started_cleanup_) return;

  if (ref) {
    uv_ref(reinterpret_cast<uv_handle_t*>(timer_handle())); // libuv的uv__handle_ref
  } else {
    uv_unref(reinterpret_cast<uv_handle_t*>(timer_handle())); // libuv的uv__handle_unref
  }
}

// 会调libuv层的uv__handle_ref
void uv_ref(uv_handle_t* handle) {
  uv__handle_ref(handle);
}
void uv_unref(uv_handle_t* handle) {
  uv__handle_unref(handle);
}
```

`libuv` 的 `uv__handle_ref` 和 `uv__handle_unref`

```c
// node-18.15.0/deps/uv/src/uv-common.h

// uv__handle_ref 标记 handle 为 REF 状态，如果 handle 是 ACTIVE 状态，则 active handle 数加一

// 末尾的反斜杠是宏续行符，用于一行写不下时换行
#define uv__handle_ref(h)                                                     \
  do {                                                                        \
    if (((h)->flags & UV_HANDLE_REF) != 0) break;                             \
    (h)->flags |= UV_HANDLE_REF;                                              \
    if (((h)->flags & UV_HANDLE_CLOSING) != 0) break;                         \
    if (((h)->flags & UV_HANDLE_ACTIVE) != 0) uv__active_handle_add(h);       \
  }                                                                           \
  while (0)

// uv__handle_unref 用于去掉 handle 的 REF 状态，如果 handle 是 ACTIVE 状态，则 active handle数减一
#define uv__handle_unref(h)                                                   \
  do {                                                                        \
    if (((h)->flags & UV_HANDLE_REF) == 0) break;                             \
    (h)->flags &= ~UV_HANDLE_REF;                                             \
    if (((h)->flags & UV_HANDLE_CLOSING) != 0) break;                         \
    if (((h)->flags & UV_HANDLE_ACTIVE) != 0) uv__active_handle_rm(h);        \
  }                                                                           \
  while (0)
```



### 1.4.4 processTimers 的正负返回值

上面讲  `processTimers()` 函数时，我们知道它的返回值 `nextExpiry` 有正负。实际上在 C++ 侧的 `RunTimers()` 中始终用的是该值的绝对值，而正负只是一个判断标识，用于判断是否需要对 `libuv` 进行 `ref` 或者 `unref` 操作。

```js
// processTimers函数
if (list.expiry > now) {
  nextExpiry = list.expiry;
  return timeoutInfo[0] > 0 ? nextExpiry : -nextExpiry; // 这里的正负表示是否还有setTimeout的引用计数
}
```

在 `processTimers` 中，当链表跑完了，剩下的都是没过期的 `Timeout`，那么需要告知 C++ 侧下一次的过期时间，以便其重设定时器。由于跑完了一堆的 `Timeout`，跑完的 `Timeout` 自然将 `timeoutInfo[0]` 的值减掉了自身这一个计数，这个时候要看剩下的 `timeoutInfo[0]` 的值是否还存在至少一个被 `ref` 的 `Timeout`。若存在，则返回正的值；否则返回负的值。



**C++ 侧怎么用这个 `nextExpiry` 返回值**

如下 `RunTimers()` 中的代码

```c++
// 在 node-18.15.0/src/env.cc 的 RunTimers() 中
if (expiry_ms != 0) {
   // 通过 llabs() 对这个返回值取绝对值
    int64_t duration_ms =
        llabs(expiry_ms) - (uv_now(env->event_loop()) - env->timer_base());

    env->ScheduleTimer(duration_ms > 0 ? duration_ms : 1);

    // 正数，则调用uv_ref
    if (expiry_ms > 0)
      uv_ref(h);
    else
      uv_unref(h);
  } else {
    uv_unref(h);
  }
```

主要逻辑如下

* 通过 `llabs()` 对这个返回值取绝对值。也就是在使用过期值时，总用其绝对值（即真实过期时间）

* 正负符号则用于判断是否需要 `uv_ref()` 或 `uv_unref()`
* 如果返回值是 `0`，则意味着没有更多 `Timeout` 了，那么直接 `uv_unref()`



# 2 定时器 setInterval()

## 2.1 setInterval() 源码

Node.js 中 [`setInterval()`](https://github.com/nodejs/node/blob/v18.15.0/lib/timers.js#L210-L238) 的源码如下

```js
// node-18.15.0/lib/timers.js
function setInterval(callback, repeat, arg1, arg2, arg3) {
  validateFunction(callback, 'callback');

  let i, args;
  switch (arguments.length) {
    // fast cases
    case 1:
    case 2:
      break;
    case 3:
      args = [arg1];
      break;
    case 4:
      args = [arg1, arg2];
      break;
    default:
      args = [arg1, arg2, arg3];
      for (i = 5; i < arguments.length; i++) {
        // Extend array dynamically, makes .apply run much faster in v6.0.0
        args[i - 2] = arguments[i];
      }
      break;
  }

  const timeout = new Timeout(callback, repeat, args, true, true);
  insert(timeout, timeout._idleTimeout);

  return timeout;
}
```

可以看到，实际上  `setInterval()  ` 的执行机制跟 `setTimeout()  ` 完全相同，`setInterval() ` 跟 `setTimeout() `的区别只是在实例化 `Timeout` 时，第4个参数 `isRepeat` 的值不一样，`setTimeout() ` 是 `false ` 表示不需要重复执行，`setInterval()` 是  `true ` 表示需要重复执行。



# 3 总结

文中讲解了 Node.js 的 2 个产生异步任务的 API，分别是` setTimeout()`、`setInterval()`，以及它们的原理和实现， `setInterval() ` 的执行机制跟 `setTimeout()` 完全相同。主要讲解了`setTimeout()`内部的 `Map` 和 优先队列结构 以及 从 `libuv` 侧执行定时器开始，一直触发到 js 侧回调函数的执行的整个流程。



# 4 参考

* [Node.js v18.15.0源码](https://github.com/nodejs/node/tree/v18.15.0)
* [死月-趣学Node.js](https://juejin.cn/book/7196627546253819916/section/7197301858296135684)
* [theanarkh-深入剖析 Node.js 底层原理](https://juejin.cn/book/7171733571638738952/section/7176118481966858297)
* [libuv api文档](http://docs.libuv.org/en/stable/api.html)

