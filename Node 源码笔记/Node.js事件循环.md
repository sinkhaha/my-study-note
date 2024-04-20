> 本文 `Node.js` 的版本 v18.15.0 、`libuv` 的版本 v1.44.2 (`libuv` 相关代码只涉及 `unix` 平台)

# 1 什么是事件循环

事件循环的本质是一个死循环，在死循环中不断处理到来的事件，比如有 文件 `I/O` 等异步 `I/O` 事件，还有定时器事件等非异步 `I/O` 事件。



`Node.js` 中的事件循环底层是由 `libuv` 实现的，`libuv` 为 `Node.js` 提供了 事件循环 与 异步 `I/O` 的能力，所以 `Node.js` 也有处理系统异步 `I/O` 事件的能力。在事件循环中，大部分时间都阻塞在等待异步 `I/O` 事件，这时不会消耗 `CPU`，一旦有异步 `I/O` 事件完成了，系统底层 `epoll` （`Linux` 是 `epoll`、`macOS` 是 `kqueue`、`Windows` 是 `IOCP`）就会得到通知，然后会去执行相应事件的回调。

> 比如 读取文件 完成了，那么 `libuv` 的事件循环接收到这个通知后，它会把结果给到等待这个读取文件事件的回调函数，回调函数最终会一路调用至 `JavaScript`，从而执行 `JavaScript` 侧的 读取文件 的回调函数



# 2 手动实现事件循环

一个最简单的事件循环的伪代码如下：

```js
while (有活跃的事件) {
  const events = 获取所有活跃的事件;
  for (const event of events) { // 遍历事件
    处理(event);
  }
}
```

更完整的例子如下：

* 增加一种任务类型及其对应的队列

* 当没有事件时阻塞，当事件到达时，需要唤醒事件循环

  > 这里使用 `await Promise` 来模拟睡眠达到等待的效果

```js
// 例子只定义了一种任务类型和队列
class EventSystem {
    constructor() {
        // 需要处理的任务队列
        this.queue = [];
      
        // 标记是否需要退出事件循环
        this.stop = 0;
      
        // 有任务时调用该函数"唤醒"
        this.wakeup = null;
    }
  
    // 没有任务时，事件循环的进入"阻塞"状态
    wait() {
        return new Promise((resolve) => {
            // 记录 resolve，可能在睡眠期间有任务到来，则需要提前唤醒
            this.wakeup = () => {
                this.wakeup = null;
                resolve();
            };
        });
    }
  
    // 停止事件循环，如果正在"阻塞"，则"唤醒它"
    setStop() {
        this.stop = 1;
        this.wakeup && this.wakeup();
    }
  
    // 生产任务
    enQueue(func) {
        this.queue.push(func);
        this.wakeup && this.wakeup();
    }

    // 处理任务队列
    handleTask() {
        if (this.queue.length === 0) {
            return;
        }
      
        // 本轮事件循环的回调中加入的任务，下一轮事件循环再处理，防止其他任务没有机会处理
        const queue = this.queue;
        this.queue = [];
      
        while(queue.length) {
            const func = queue.shift();
            func();
        }
    }

    // 事件循环的实现
    async run() {
        // 如果 stop 等于 1 则退出事件循环
        while(this.stop === 0) {
            // 处理任务，可能没有任务需要处理
            this.handleTask();
          
            // 处理任务过程中如果设置了 stop 标记则退出事件循环
            if (this.stop === 1) {
                break;
            }
          
            // 没有任务了，进入睡眠
            if (this.queue.length === 0) {
                await this.wait();
            }
        }
      
        // 退出前可能还有任务没处理，处理完再退出
        this.handleTask();
    }
}

// 测试
const eventSystem = new EventSystem();

// 启动前生成的任务
eventSystem.enQueue(() => {
    console.log('1');
});

// 模拟定时生成一个任务
setTimeout(() => {
    eventSystem.enQueue(() => {
        console.log('3');
        eventSystem.setStop();
    });
}, 1000);

// 启动事件循环
eventSystem.run();

// 启动后生成的任务
eventSystem.enQueue(() => {
    console.log('2');
});
```



# 3 Node.js 中实现事件循环

## 3.1 外层循环(do-while 循环)

`Node.js` 中[事件循环的主要代码](https://github.com/nodejs/node/blob/v18.15.0/src/api/embed_helpers.cc#L35-L58)如下

```c++
// node-18.15.0/src/api/embed_helpers.cc
do { // 外层循环
  // 停止状态则结束事件循环
  if (env->is_stopping()) break;
  uv_run(env->event_loop(), UV_RUN_DEFAULT); // 跑一次事件循环
  if (env->is_stopping()) break;

  // 执行v8平台中的一些任务
  platform->DrainTasks(isolate);

  // 事件循环是否还有活跃的事件(待处理的事件)，有的话则跳过去继续执行uv_run
  more = uv_loop_alive(env->event_loop());
  if (more && !env->is_stopping()) continue;

  // 没有process.on('beforeOnExit')事件则退出
  if (EmitProcessBeforeExit(env).IsNothing())
    break;

  {
    HandleScope handle_scope(isolate);
    // 执行 v8.startupSnapshot 的序列化回调
    if (env->RunSnapshotSerializeCallback().IsEmpty()) {
      break;
    }
  }

  // Emit `beforeExit` if the loop became alive either after emitting
  // event, or after running some callbacks.
  more = uv_loop_alive(env->event_loop());
} while (more == true && !env->is_stopping()); // 事件循环还活着，则继续循环
```

代码中主要是一个 `do-while` 循环，为了方便表述，我们直接称之为**外层循环**，其逻辑如下：

1. 首先用 `is_stopping()` 判断是否是停止状态，是的话则结束事件循环
2. 接着执行 `uv_run()` 跑一次事件循环（这是真正的事件循环的处理逻辑）
3. 再次判断是否停止 `is_stopping()` 
4. `platform->DrainTasks(isolate)`：执行 `V8` 平台中的一些任务
5. 执行完 `V8` 平台的一些任务后，可能事件循环有新的事件产生，所以又判断一下 `uv_loop_alive()` 是否还活着（是否为 0）
6. 若不为 0，则直接 `continue` 跳过，继续执行 `do-while` 循环（即又从头开始，执行 `uv_run()` 事件循环）
7. 若为 0 ，说明事件循环没有新的事件需要处理，接着做一些扫尾工作；判断是否有 `process.on('beforeExit')` 事件，若没有，则退出 `do-while` 循环，若有，则再判断是否有 `v8.startupSnapshot` 的序列化回调要执行，没有则退出 `do-while` 循环，有则执行。然后再判断一下 `uv_loop_alive()` 事件循环是否还有活跃的事件（说明在执行 `v8.startupSnapshot` 的序列化回调期间事件循环还有可能被产生新的事件）
8. 如果还有活跃的事件，说明还要执行事件循环，则继续执行 `do-while` 循环



**内层循环**：在 `do-while` 循环中，会调用 `uv_run()` 方法， `uv_run()` 就是事件循环的主要逻辑，为了方便表述，我们称之为**内存循环**

> 此时 `uv_run(env->event_loop(), UV_RUN_DEFAULT)` 表示执行事件循环；
>
>  `UV_RUN_DEFAULT` 是默认的运行模式，表示会执行事件循环直到不再有活动的 和 被引用的句柄（`handle`）或 请求（`request`）为止



**外层循环的作用：**是为了保证执行完内层循环 `uv_run()` 后，如果事件循环还会再产生新的事件，则重新执行一轮内层循环，确保事件都会被处理，只有当事件循环没事件要处理了，才真正的结束程序



## 3.2 内层循环(事件循环)

内层循环就是事件循环，它是由 `libuv` 的 `uv_run()` 实现，流程图如下：

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/20230405205556.png)







**事件循环分析**

[`uv_run()`](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/src/unix/core.c#L382-L448) 的源码如下（因为 `libuv` 是跨平台的，这里只涉及 `UNIX` 类系统）

```c
// node-18.15.0/deps/uv/src/unix/core.c
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  int timeout;
  int r;
  int ran_pending;

  // 是否有活跃事件
  r = uv__loop_alive(loop);
  if (!r)
    uv__update_time(loop);

  // libuv的uv_stop()函数会把 loop->stop_flag 设置为 1，设置为 1 则会停止事件循环
  while (r != 0 && loop->stop_flag == 0) {
    uv__update_time(loop); // 更新当前时间，每轮事件循环会缓存这个时间以便后面使用，避免过多系统调用损耗性能
    uv__run_timers(loop); // 执行定时器回调
    ran_pending = uv__run_pending(loop); // 执行 pending 回调
    uv__run_idle(loop); // 处理空转事件
    uv__run_prepare(loop); // 处理准备事件

    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop); // 计算 Poll I/O 阶段阻塞时间，0则不阻塞

    uv__io_poll(loop, timeout); // 处理Poll I/O，timeout是 epoll_wait 的等待时间

    /* Run one final update on the provider_idle_time in case uv__io_poll
     * returned because the timeout expired, but no events were received. This
     * call will be ignored if the provider_entry_time was either never set (if
     * the timeout == 0) or was already updated b/c an event was received.
     */
    uv__metrics_update_idle_time(loop);

    uv__run_check(loop); // 处理复查事件
    uv__run_closing_handles(loop); // 扫尾处理

    // 只跑一次
    if (mode == UV_RUN_ONCE) {
      /* UV_RUN_ONCE implies forward progress: at least one callback must have
       * been invoked when it returns. uv__io_poll() can return without doing
       * I/O (meaning: no callbacks) when its timeout expires - which means we
       * have pending timers that satisfy the forward progress constraint.
       *
       * UV_RUN_NOWAIT makes no guarantees about progress so it's omitted from
       * the check.
       */
      uv__update_time(loop); // 更新事件循环的时间
      uv__run_timers(loop); // 继续执行定时器
    }

    // 是否还有活跃事件，有则继续下一轮事件循环
    r = uv__loop_alive(loop);
    if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
      break;
  }

  /* The if statement lets gcc compile it to a conditional store. Avoids
   * dirtying a cache line.
   */
  // 重置状态
  if (loop->stop_flag != 0)
    loop->stop_flag = 0;

  return r;
}

// 是否有活跃事件
static int uv__loop_alive(const uv_loop_t* loop) {
  return uv__has_active_handles(loop) ||
         uv__has_active_reqs(loop) ||
         !QUEUE_EMPTY(&loop->pending_queue) ||
         loop->closing_handles != NULL;
}
```

主要逻辑为：

1. 使用 `uv__loop_alive()` 先判断有没有活跃的事件

2. 若无活跃事件，则通过 [`uv__update_time()`](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/src/unix/internal.h#L309-L313)，则更新整个循环的最后处理时间

   > `uv__update_time()` 的逻辑就是更新 `loop` 结构体内部的时间戳 `time` 字段，即最后一轮事件循环处理的时间是 `uv__hrtime(UV_CLOCK_FAST) / 1000000`，在事件循环处理中会用到这个时间字段

3. 若有活跃事件，则进入 `while` 循环开始处理各种事件，执行事件循环的各个阶段（`while` 循环里面的逻辑下面单独讲解）

   > `while` 循环的退出条件是 没有活跃事件 或 事件循环处于 `stop` 停止状态（即 `loop->stop_flag `的值为1）

4. 当 `while` 循环结束后，重置 `stop` 状态，以便下次外部再次开始事件循环

   > `libuv` 的 [`uv_stop()`](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/src/uv-common.c#L578-L580) 函数会把 `loop->stop_flag` 设置为1，所以通过调用 `uv_stop()` 可以停止事件循环

5. 最后返回最后一次获取的“是否有活跃事件”的标识`r`

   > 通常情况下，都是没有活跃事件了才会退出内层循环的 `while` 循环。不过，若是其中途被终止（调用`uv_stop()`）或 模式为 `UV_RUN_ONCE` 等，还是会出现有活跃事件的，这时外部在用 `libuv` 时就要考虑是否要重新回到新的一轮 `uv_run()` 内层循环；在 Node.js 中应用 `libuv`，外层循环就并未使用内层循环 `uv_run()` 的返回值，而是在 `V8 Platform` 跑了一遍任务之后，自己又通过 `uv_loop_alive()` 重新获取了一遍“是否有活跃事件”。



**`while` 循环主要是处理各个阶段的事件，逻辑为**

1. `uv__update_time`：更新 `loop` 的最后处理时间（这个时间的用处之一是 `setTimeout()` 会以该时间为准，判断`setTimeout()`是否已经过期）
2. [`uv__run_timers`](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/src/timer.c#L163-L180)：执行定时器 `setTimeout()`事件，大概流程就是在存放定时器的小根堆里遍历出已过期的定时器，并依次执行定时器的回调
3. `uv__run_pending`：遍历并执行 `I/O` 事件结束后（完成或失败），丢进 `pending` 队列等待后续处理的事件对应的回调（如 TCP 进行连接时连接失败产生的回调）
4. `uv__run_idle`：遍历并执行空转（`idle`）事件
5. `uv__run_prepare`：遍历并执行准备（`prepare`）事件
6. `uv_backend_timeout`：获取还没过期且是最近要过期的定时器的时间间隔，这个时间是 `Poll I/O` 阻塞等待的时间间隔
7. `uv__io_poll`：根据 `epoll`、`kqueue` 等 `I/O` 多路复用机制，去监听等待 `I/O` 事件触发，并以上一步 `uv_backend_timeout` 获取的时间间隔作为最长等待时间，若超时还没有 `I/O` 事件触发，则直接取消此次等待，因为时间到了还没有 `I/O` 事件触发，说明还有定时器要执行，且定时器触发时间到了，那 `libuv` 就要去处理下一轮定时器了，不能一直阻塞在等待 `I/O`
8. `uv__metrics_update_idle_time`：更新 `loop` 中 `Metrics` 里的 `idle_time`，上一步中会更新其 `provider_entry_time`，然后更新 `idle_time`，但若上一步是直到超时还没有事件触发，则不会更新 `idle_time`，所以这里要做一次“最终更新”，若上一步已经更新了 `idle_time`，则这一步内部逻辑不会有任何效果
9. `uv__run_check`：遍历并执行复查（`check`）事件
10. `uv__run_closing_handles`：遍历一遍“正在关闭”的句柄，并对其进行扫尾工作（如 `TCP` 类型就需要销毁其流）
11. `mode == UV_RUN_ONCE`：若当前的模式为 `UV_RUN_ONCE`（即只跑一次循环），则再次更新一遍 `loop` 最后处理时间，并执行定时器事件（因为经过刚才一系列流程后系统时间又过了一会儿，这时就要把新时间内触发的定时器都执行完）
12. `uv__loop_alive`：重新判断一遍有没有活跃的事件，因为经过上述一系列操作后，有可能一些监听被取消了
13. 最后，若循环的模式为 `UV_RUN_ONCE` 或 `UV_RUN_NOWAIT`，则退出 `while` 循环，也就是这两类模式都只跑一次循环



从代码中也可以知道，`uv_run()` 本质是在一个 `while` 循环中，串行处理各个阶段的事件回调，此实现方式有个缺点是当一个任务执行时间过长，则会影响后面任务的执行，导致事件循环延迟过高。



**uv_run() 对应流程图如下**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/%E4%BA%8B%E4%BB%B6%E5%BE%AA%E7%8E%AF%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.png)





# 4 事件循环的 7 个阶段

从上面的章节可知， `libuv` 的 `uv_run()` 事件循环共有7个阶段，每个阶段会分别处理以下事件

1. `timer`（定时器）事件
2. `pending` 态的 `I/O` 事件
3. `idle` （空转）事件
4. `prepare` （准备）事件
5. `Poll I/O` 事件
6. `check` （复查）事件
7. `close` 事件



**事件循环7个阶段的流程图如下**



![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/%E4%BA%8B%E4%BB%B6%E5%BE%AA%E7%8E%AF.png)



## 4.1 第1阶段：timer（定时器）阶段

`timer` 阶段主要处理定时器事件（`setTimeout()` 和 `setInterval()`）。



在 `Node.js` 中，定时器的执行是由 `libuv` 实现，`Node.js` 则实现了定时器的管理

* `libuv` 中维护了一个小顶堆存放定时器，堆顶节点是最快超时的定时器节点。在事件循环的定时器阶段， `libuv` 会去判断当前堆顶节点有没有过期，如果没有，说明都没有定时器过期，结束定时器处理阶段；如果堆顶节点过期了，则执行该节点定时器的回调，并且把它移出这个小顶堆，调整小顶堆后，继续遍历定时器小顶堆

* `libuv`  的定时器可以设置 `repeat` 参数，表示这个定时器是否需要重复执行。例如 `setInterval() `这种场景，可以给定时器设置 `repeat` 标记，那么这个定时器节点每次超时执行完该定时器回调后，会再次被重新插入小顶堆中，等待下次超时时再次被执行

  > 注意：`Node.js` 并没有直接用 `libuv` 的 `repeat` 参数，而是自己实现定时器的管理。`Node.js` 中`setInterval()` 实质实现跟 `setTimeout()` 一样，`setInterval()` 只是在 `setTimeout()` 到期被执行完后又重新插入 `Node.js` 自己维护的小顶堆中，等待下一次被执行。
  >
  > 关于 `Node.js` 中定时器的实现介绍可参考[另一篇文章](https://zhuanlan.zhihu.com/p/619711042)



**`libuv` 中定时器处理流程图如下：**



![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/%E5%AE%9A%E6%97%B6%E5%99%A8%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.png)





###  使用 libuv 中的定时器

**使用 `libuv` 中定时器的例子，如下**

> 完整代码可参考[这里](https://github.com/sinkhaha/libuv-demo/tree/main/timer)

```c
#include <stdio.h>
#include <uv.h> // 引入libuv头文件

void once_cb(uv_timer_t *handle) {
    printf("timer callback\n");
}

int main() {
    uv_timer_t once;
    // 初始化 uv_timer_t 结构体
    uv_timer_init(uv_default_loop(), &once);

    // 启动定时器，超时后执行 once_cb回调函数，2000为超时时间，0 表示只执行一次回调
    uv_timer_start(&once, once_cb, 2000, 0);

    // 执行事件循环
    uv_run(uv_default_loop(), UV_RUN_DEFAULT);

    return 0;        
}

```

1. `uv_timer_init()`：初始化 `uv_timer_t handle`
2. `uv_timer_start()`：启动定时器
3. `uv_run()`：执行事件循环



**分析**

1、 `uv_timer_init()` 函数：和事件循环其他阶段的 `init` 函数一样，用于初始化 `handle`

```c
// 初始化uv_timer_t结构体
int uv_timer_init(uv_loop_t* loop, uv_timer_t* handle) {
    uv__handle_init(loop, (uv_handle_t*)handle, UV_TIMER);
    handle->timer_cb = NULL;
    handle->repeat = 0;
    return 0;
}
```



2、`uv_timer_start()` 函数：用于激活启动一个定时器

> 激活启动定时器后，在事件循环的 `timer` 定时器阶段就会判断有没有定时器超时，有则执行回调

```c
// 启动一个定时器
int uv_timer_start(uv_timer_t* handle,
                   uv_timer_cb cb, // 超时之后的回调函数
                   uint64_t timeout, // 超时时间
                   uint64_t repeat) {
  uint64_t clamped_timeout;

  if (uv__is_closing(handle) || cb == NULL)
    return UV_EINVAL;

  // 在启动定时器前，把之前的先停掉
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

  // 把handle节点插入小顶堆，里面会根据该节点的超时时间动态调整小顶堆
  heap_insert(timer_heap(handle->loop),
              (struct heap_node*) &handle->heap_node,
              timer_less_than);
  // 启动定时器(激活该handle)
  uv__handle_start(handle);

  return 0;
}
```

注意：`Node.js` 并不是每次调 `setTimeout()` 的时候都往 `libuv` 的小顶堆插入一个节点，因为这样会引起 `JS` 侧和 `C++` 、`C` 侧频繁通信，损耗性能。因此在 Node.js 中，只有一个关于 `uv_timer_t handle`，`Node.js` 在 `JS` 侧自己维护了 `Map` 和 优先队列，每次计算出最快到期节点的时间，然后修改 `libuv uv_timer_s handle` 的超时时间

> 关于 Node.js 中定时器的原理可参考[另一篇文章](https://zhuanlan.zhihu.com/p/619711042)



3、 `uv__run_timers()` 函数：该函数即为事件循环的定时器处理阶段，处理定时器事件

```c
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

    // 移除定时器节点
    uv_timer_stop(handle);
    // 如果设置了 repeat 则重新插入小顶堆，等待下次超时时执行
    uv_timer_again(handle);
    // 执行定时器的回调
    handle->timer_cb(handle);
  }
}
```

主要逻辑就是遍历小顶堆，找出当前超时的节点，执行超时节点的回调



4、`uv_timer_stop()` 函数：在 `uv__run_timers()` 函数中被执行，其作用是停止一个定时器

```c
// 停止一个定时器
int uv_timer_stop(uv_timer_t* handle) {
  if (!uv__is_active(handle))
    return 0;

  // 从小顶堆中移除该定时器节点
  heap_remove(timer_heap(handle->loop),
              (struct heap_node*) &handle->heap_node,
              timer_less_than);
  
  // 清除激活状态和 handle 的 active 数减一
  uv__handle_stop(handle);

  return 0;
}
```

其主要逻辑就是把 定时器句柄（ `handle` ）从小顶堆中删除



5、`uv_timer_again()` 函数：也是在 `uv__run_timers()` 函数中被执行，作用是可以支持 `setInterval()` 这种重复执行定时器的场景

```c
int uv_timer_again(uv_timer_t* handle) {
  if (handle->timer_cb == NULL)
    return UV_EINVAL;

  // 如果设置了 repeat ，说明定时器需要被重复触发
  if (handle->repeat) {
    // 先把旧的定时器节点从小顶堆中移除，然后再重新开启一个定时器
    uv_timer_stop(handle);
    uv_timer_start(handle, handle->timer_cb, handle->repeat, handle->repeat);
  }

  return 0;
}
```



## 4.2 第2阶段：pending 阶段

`pending` 阶段主要处理 `pending` 态的 `I/O` 事件，即上一轮循环中 `Poll I/O` 阶段产生的一些回调。

在大多数情况下，所有的 `Poll I/O` 回调函数都会在 `Poll I/O` 阶段调用；但有些情况例外，它们需要将 `Poll I/O` 回调函数延迟到下一轮事件循环的 `pending` 阶段调用（比如 `tcp` 连接失败时产生的回调）



**uv__run_pending()函数**

[`uv__run_pending()`](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/src/unix/core.c#L798-L812) 函数主要处理 `pending` 阶段的事件，源码如下

```c
// node-18.15.0/deps/uv/src/unix/core.c
static void uv__run_pending(uv_loop_t* loop) {
  QUEUE* q;
  QUEUE pq;
  uv__io_t* w;

  // 把 pending_queue 队列的节点移到 pq，清空 pending_queue，达到一个防重入的效果
  QUEUE_MOVE(&loop->pending_queue, &pq);

  // 遍历 pq 队列
  while (!QUEUE_EMPTY(&pq)) {
    // 取出当前第一个需要处理的节点
    q = QUEUE_HEAD(&pq);
    // 把当前需要处理的节点移出队列
    QUEUE_REMOVE(q);
    // 重置一下 prev 和 next 指针
    QUEUE_INIT(q);
    w = QUEUE_DATA(q, uv__io_t, pending_queue);
    // 执行回调，注意：回调的入参是 loop、IO观察者、可写事件
    w->cb(loop, w, POLLOUT);
  }
}
```

主要逻辑就是遍历 `pending_queue` 队列，然后执行每个节点的回调。



**防重入措施：**如 `QUEUE_MOVE(&loop->pending_queue, &pq)`，就是把 `pending_queue` 队列的节点移到 `pq` 队列中（相当于清空了 `pending_queue` 队列），接下去遍历 `pq` 队列即可；因为如果直接遍历 `pending_queue` 队列，在执行回调时如果一直往 `pending_queue` 队列里添加节点，会导致 `while` 循环无法退出。此时利用防重入，新插入 `pending_queue` 的节点会在下一轮事件循环才被处理。



**tcp 连接错误回调延迟到 pending 阶段处理的例子**

当 `TCP` 连接时，连接失败产生的回调 会在下一轮事件循环的 `pending` 阶段执行。例如[发生 ECONNREFUSED 错误时](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/src/unix/tcp.c#L249)，需要延迟报错，此时会将该错误的回调处理放入 `pending_queue` 队列，一直到下一轮事件循环的 `pending` 阶段才进行处理。源码如下

```c++
// node-18.15.0/deps/uv/src/unix/tcp.c
int uv__tcp_connect(...) {
   ...
    
  do {
    errno = 0;
     // 非阻塞式发起连接
    r = connect(uv__stream_fd(handle), addr, addrlen);
  } while (r == -1 && errno == EINTR);

  
   /* We not only check the return value, but also check the errno != 0.
   * Because in rare cases connect() will return -1 but the errno
   * is 0 (for example, on Android 4.3, OnePlus phone A0001_12_150227)
   * and actually the tcp three-way handshake is completed.
   */
  // 连接失败或还没成功
  if (r == -1 && errno != 0) {
    if (errno == EINPROGRESS) // 连接中的错误码
      ; /* not an error */
    else if (errno == ECONNREFUSED
#if defined(__OpenBSD__)
      || errno == EINVAL
#endif
      )
    /* If we get ECONNREFUSED (Solaris) or EINVAL (OpenBSD) wait until the
     * next tick to report the error. Solaris and OpenBSD wants to report
     * immediately -- other unixes want to wait.
     */
      handle->delayed_error = UV__ERR(ECONNREFUSED); // ECONNREFUSED错误需要延迟报错
    else
      return UV__ERR(errno);
  }
  
...

  if (handle->delayed_error)
    // uv__io_feed()就是放入pending_queue，即产生 pending 任务
    uv__io_feed(handle->loop, &handle->io_watcher);
  
}

// 把一个 IO 观察者放入pending_queue队列
void uv__io_feed(uv_loop_t* loop, uv__io_t* w) {
  if (QUEUE_EMPTY(&w->pending_queue))
    QUEUE_INSERT_TAIL(&loop->pending_queue, &w->pending_queue);
}
```

主要逻辑

* `uv__tcp_connect()` 调用 `connect` 函数以非阻塞方式发起 `TCP` 连接
* 当 `connect` 返回时，可能处于连接中或者失败
* 当失败的错误码 `errno` 是 `ECONNREFUSED` 时，`libuv` 会执行 `uv__io_feed()` 函数，`uv__io_feed()` 函数会把一个 `IO` 观察者插入到 `pending` 队列



## 4.3 第3阶段：idle（空转）阶段

`idle` 阶段主要用于处理 `idle` 空转事件。



`Node.js v18.15.0` 中有一个地方用到了空转事件，那就是 `setImmediate()`。因为 `libuv` 中，当存在空转事件时，事件循环的 `Poll I/O` 阶段不会阻塞等待 `IO`，所以 `Node.js` 在通过 `libuv` 激活`setImmediate()` 时，会在 `idle` 空转队列插入一个空转事件，标记有 `immediate` 任务要处理不能阻塞 `IO`，需要 `immediate`（立即）执行

> `idle`、`prepare`、`check` 在 `libuv` 事件循环中属于比较简单的阶段，它们的实现是一样的，只是执行时机不一样，这 3 个阶段此本文只讲解 `prepare` 阶段的流程，具体解析见下节。
>
> 注意 `libuv v1.44.2` 版本，它们的定义是在 `libuv/src/unix/loop-watcher.c` 文件中。
>
> `Node.js` 中 `setImmediate()` 的实现可参考[另一篇文章](https://zhuanlan.zhihu.com/p/619712752)



**为什么有空转事件时就不会阻塞Poll I/O**

因为在 `libuv` 中， `Poll I/O` 的阻塞时间由 [`uv_backend_timeout()`](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/src/unix/core.c#L357-L374) 的返回值决定，当返回 0 时，说明不会阻塞，否则返回下一次定时器到期的时间间隔作为阻塞时间。当空转事件 `&loop->idle_handles` 不为空时，`uv_backend_timeout()` 会返回 0，为空则返回下一次定时器到期的时间间隔，源码如下

```c
// node-18.15.0/deps/uv/src/unix/core.c
int uv_backend_timeout(const uv_loop_t* loop) {
  if (QUEUE_EMPTY(&loop->watcher_queue))
    return uv__backend_timeout(loop);
  /* Need to call uv_run to update the backend fd state. */
  return 0;
}

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
```



## 4.4 第4阶段：prepare 准备阶段

`prepare` 准备阶段主要用于处理 `prepare` 准备事件。`prepare` 准备事件在 `idle` 空转事件之后执行，只有 `prepare` 时，会阻塞在 `Poll I/O`。

> `idle`、`prepare`、`check` 在  `libuv` 事件循环中属于比较简单的阶段，它们的实现是一样的，但执行时机不一样。
>
> 有一点不同的是，当只有 `idle` 任务时，事件循环不会阻塞在 `Poll I/O` 阶段，而当只有 `prepare` 任务 或 `check` 任务时，事件循环会阻塞在  `Poll I/O`  阶段。



### 使用 libuv 中的 prepare

通过下面的例子，看下 `prepare` 阶段的任务是如何创建和处理的

> 完整代码可参考[这里](https://github.com/sinkhaha/libuv-demo/tree/main/prepare)

```c
#include <stdio.h>
#include <uv.h> // 引入libuv头文件

void prep_cb(uv_prepare_t *handle) {
    printf("prepare callback\n");
}

int main() {
    uv_prepare_t prep;

    // 初始化 prepare handle
    uv_prepare_init(uv_default_loop(), &prep);

    // 启动prepare，prep_cb 为事件循环都会执行的回调函数 
    uv_prepare_start(&prep, prep_cb);

    // 执行事件循环，运行模式为UV_RUN_DEFAULT，在 prepare阶段 会执行 prep_cb 回调
    uv_run(uv_default_loop(), UV_RUN_DEFAULT);

    return 0;
}
```

1. `uv_prepare_init()`：初始化 `uv_prepare_t handle`
2. `uv_prepare_start()`：启动 `prepare` 阶段
3. `uv_run()`：执行事件循环

> 注意：这个例子执行后不会退出事件循环，会一直阻塞在 `Poll I/O` 阶段。虽然 `prepare` 事件在执行后，又会重新插入 `prepare` 队列，但是此例子执行后，只会打印一次 `“prepare callback”`，这是因为此时只有 `prepare` 任务，事件循环会阻塞在 `Poll I/O` 阶段，所以不会继续执行下去。
>
> 如果在该例子的基础上再添加一个定时器，使得不阻塞在 `Poll I/O` 阶段，会发现会一直执行 `prep_cb` 回调，代码如下
>
> ```c
> #include <stdio.h>
> #include <uv.h> // 引入libuv头文件
> 
> void prep_cb(uv_prepare_t *handle) {
>     printf("prepare callback\n");
> }
> 
> void once_cb(uv_timer_t *handle) {
>     printf("timer callback\n");
> }
> 
> int main() {
>     uv_timer_t once;
>     // 初始化 uv_timer_t 结构体
>     uv_timer_init(uv_default_loop(), &once);
>     // 启动定时器，超时后执行 once_cb回调函数，2000为超时时间，1 表示重复执行
>     uv_timer_start(&once, once_cb, 2000, 1);
> 
> 
>     uv_prepare_t prep;
>     // 初始化 prepare handle
>     uv_prepare_init(uv_default_loop(), &prep);
>     // 启动prepare，prep_cb 为事件循环都会执行的回调函数 
>     uv_prepare_start(&prep, prep_cb);
>     // 执行事件循环，运行模式为UV_RUN_DEFAULT，在 prepare阶段 会执行 prep_cb 回滴
>     uv_run(uv_default_loop(), UV_RUN_DEFAULT);
>    
>     return 0;
> }
> ```



**分析**

对于一个 `handle` 类型，有四个通用的操作函数，分别是 `init`、`start`、`stop` 和 `close`。

* `uv_handle类型_init()`：做一些 `handle` 的初始化操作，如 `uv_prepare_init()`
* `uv_handle类型_start()`：通常在 `uv_handle类型_init` 后调用，表示激活这个 `handle`，如 `uv_prepare_start()`
* `uv_handle类型_stop()`：它会修改 `handle` 的状态和修改 `active handle` 的数量，但是不会把 `handle` 移出事件循环的 `handle` 队列；通常意味着这个 `handle` 处于暂停状态，后续还可以调用 `start` 重新激活。比如可以使用 `uv_timer_stop()` 停止一个定时器，然后再执行 `uv_timer_start()` 重启这个定时器
* `uv_handle类型_close()`：表示这个 `handle` 已经被关闭了，不再重新使用，同时也会被移出事件循环的 `handle` 队列，如`uv_handle_close()`



1、`uv_prepare_init()` 函数：主要是做一些初始化操作

```c
// libuv 1.44.2 中该函数是使用宏的形式定义在 libuv/src/unix/loop-watcher.c 文件中
#define UV_LOOP_WATCHER_DEFINE(name, type)                                    \
  int uv_##name##_init(uv_loop_t* loop, uv_##name##_t* handle) {              \
    uv__handle_init(loop, (uv_handle_t*)handle, UV_##type);                   \
    handle->name##_cb = NULL;                                                 \
    return 0;                                                                 \
  }                                                                           \
  ...
 }      
   
// 展开如下
int uv_prepare_init(uv_loop_t* loop, uv_prepare_t* handle) {
    uv__handle_init(loop, (uv_handle_t*)handle, UV_PREPARE);
    handle->prepare_cb = NULL;
    return 0;
}
```



2、`uv_prepare_start()` 函数：激活 `prepare handle`

```c
// libuv 1.44.2 中该函数是使用宏的形式定义在 libuv/src/unix/loop-watcher.c 文件中
#define UV_LOOP_WATCHER_DEFINE(name, type)                                    \
                                                                              \
  int uv_##name##_start(uv_##name##_t* handle, uv_##name##_cb cb) {           \
    if (uv__is_active(handle)) return 0;                                      \
    if (cb == NULL) return UV_EINVAL;                                         \
    QUEUE_INSERT_HEAD(&handle->loop->name##_handles, &handle->queue);         \
    handle->name##_cb = cb;                                                   \
    uv__handle_start(handle);                                                 \
    return 0;                                                                 \
  }   

  ....
}

// 展开如下
int uv_prepare_start(uv_prepare_t* handle, uv_prepare_cb cb) {
    // 如果已经执行过 start 函数则直接返回
    if (uv__is_active(handle)) return 0;
    if (cb == NULL) return UV_EINVAL;
    // 把 handle 插入事件循环中的 prepare_handles 队列（prepare_handles里保存了prepare阶段的任务）
    QUEUE_INSERT_HEAD(&handle->loop->prepare_handles, &handle->queue); 
    handle->prepare_cb = cb; // 设置回调函数
    uv__handle_start(handle);
    return 0;
}
```



3、` uv__run_prepare()` 函数：该函数即为事件循环的 `prepare` 处理阶段

```c
// libuv 1.44.2 中该函数是使用宏的形式定义在 libuv/src/unix/loop-watcher.c 文件中
#define UV_LOOP_WATCHER_DEFINE(name, type)                                    \
 void uv__run_##name(uv_loop_t* loop) {                                      \
    uv_##name##_t* h;                                                         \
    QUEUE queue;                                                              \
    QUEUE* q;                                                                 \
    QUEUE_MOVE(&loop->name##_handles, &queue);                                \
    while (!QUEUE_EMPTY(&queue)) {                                            \
      q = QUEUE_HEAD(&queue);                                                 \
      h = QUEUE_DATA(q, uv_##name##_t, queue);                                \
      QUEUE_REMOVE(q);                                                        \
      QUEUE_INSERT_TAIL(&loop->name##_handles, q);                            \
      h->name##_cb(h);                                                        \
    }                                                                         \
  }  
...
}

// 展开如下
void uv__run_prepare(uv_loop_t* loop) {
    uv_prepare_t* h;
    QUEUE queue;
    QUEUE* q;
  
    // 把prepare_handles队列的节点移动到queue中， 防重入措施（作用跟上面pending阶段的处理类似）
    QUEUE_MOVE(&loop->prepare_handles, &queue);
  
    // 遍历队列，执行每个节点里面的函数
    while (!QUEUE_EMPTY(&queue)) {
        // 取下当前待处理的节点（队列的头节点）
        q = QUEUE_HEAD(&queue);
      
        // 取得该节点对应的整个结构体的基地址，即通过结构体成员取得结构体首地址
        h = QUEUE_DATA(q, uv_prepare_t, queue);
      
        // 把该节点移出当前队列
        QUEUE_REMOVE(q);
      
        // 重新插入原来的prepare_handles队列
        QUEUE_INSERT_TAIL(&loop->prepare_handles, q);
      
        // 执行回调函数
        h->prepare_cb(h);
    }
}
```

主要逻辑就是逐个执行 `prepare_handles` 队列的节点。

**注意： `prepare` 节点被移出队列后，又执行 `QUEUE_INSERT_TAIL` 被重新插入到`prepare_handles` 队列了。所以 `prepare` 阶段的任务是每次事件循环都会被执行的**

> `idle` 跟 `check` 这两个阶段的处理逻辑跟 `prepare` 类似，所以它们的任务也是每次事件循环都会被执行



4、`uv_prepare_stop()` 函数： 停止 `prepare handle`

因为此例子的 `libuv` 的运行模式是 `UV_RUN_DEFAULT` ，且有 `active` 状态的 `handle（prepare 节点）`，所以事件循环是不会退出的，它会一直执行回调。那怎样可以退出事件循环呢？

只要执行 `uv_prepare_stop()` 即可退出事件循环，因为 `uv_prepare_stop()` 函数，它会停止 `uv_prepare_t handle` ，并移除 `prepare` 队列；但是该函数不会把 `handle` 移出事件循环的 `handle` 队列，如果想把 `handle` 移出事件循环的 `handle` 队列，需要调用 `uv_close()`，`uv_close()` 里除了调用 `uv_prepare_stop()`，还会把 `handle` 移出 `handle` 队列，源码如下

```c
// libuv 1.44.2 中该函数是使用宏的形式定义在 libuv/src/unix/loop-watcher.c 文件中
#define UV_LOOP_WATCHER_DEFINE(name, type)                                    \
  int uv_##name##_stop(uv_##name##_t* handle) {                               \
    if (!uv__is_active(handle)) return 0;                                     \
    QUEUE_REMOVE(&handle->queue);                                             \
    uv__handle_stop(handle);                                                  \
    return 0;                                                                 \
  }       
}


// 展开如下
int uv_prepare_stop(uv_prepare_t* handle) {
    if (!uv__is_active(handle)) return 0;
    // 把 handle 从 prepare 队列中移除，但是还挂载到 handle_queue 中
    QUEUE_REMOVE(&handle->queue);
    // 清除 active 标记位并且减去事件循环中 handle 的 active 数
    uv__handle_stop(handle);
    return 0;
}
```



## 4.5 第5阶段：Poll I/O 阶段

`Poll I/O` 是 `libuv` 最重要的一个阶段，网络 `IO`、线程池完成任务、信号处理等回调都是在这个阶段处理的。这个阶段本质上是对不同操作系统事件驱动模块的封装（比如 `linux` 的 `epoll` 、`MacOS` 的 `kqueue`、`Windows` 的 `IOCP` ），等到有事件触发时，再去执行相应的回调函数。



### 4.5.1 I/O 观察者

先了解一下 `libuv` 中， `Poll I/O` 阶段最重要的数据结构：`I/O` 观察者（ [`uv__io_s`](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/include/uv/unix.h#L94-L102)），它本质上是封装了文件描述符(fd)、感兴趣的事件、回调的结构体。结构体定义如下：

```c
// node-18.15.0/deps/uv/include/uv/unix.h
struct uv__io_s {
    // 事件触发后的回调
    uv__io_cb cb;
  
    void* pending_queue[2];
    void* watcher_queue[2]; // IO 观察者队列
  
    // 保存当前感兴趣的事件，还没有同步的操作系统。每次设置时首先保存事件在这个字段，然后 Poll IO 阶段再操作事件驱动模块更新到操作系统
    unsigned int pevents; 
  
    // 保存更新到操作系统的事件，每次 Poll IO 阶段更新 pevents 的值到操作系统后就把 pevents 同步到 events
    unsigned int events;
  
    // 标记对哪个文件描述符的事件感兴趣
    int fd;
  
    UV_IO_PRIVATE_PLATFORM_FIELDS
};
```

`libuv` 会维护一个 `IO` 观察者队列 `watcher_queue` ，根据 `IO` 观察者描述的信息，在 `Poll I/O` 阶段往底层的事件驱动模块注册相应的信息，当注册的事件触发时，`IO` 观察者的回调就会被执行。



#### **初始化** **IO** 观察者

[`uv__io_init()`](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/src/unix/core.c#L861-L875) 函数的作用是初始化 `IO` 观察者

```c
// node-18.15.0/deps/uv/src/unix/core.c
void uv__io_init(uv__io_t* w, uv__io_cb cb, int fd) {
  ...
    
  // 初始化队列 
  QUEUE_INIT(&w->pending_queue);
  QUEUE_INIT(&w->watcher_queue); // IO 观察者队列
  w->cb = cb;  // 初始化回调
  w->fd = fd;  // 初始化需要监听的fd
  w->events = 0;  // 表示当前设置到操作系统的事件
  w->pevents = 0;  // 表示当前感兴趣的事件

  ...
}

```

主要逻辑如下

* 初始化队列
* 初始化回调
* 初始化要监听的 `fd`
* 初始化当前设置到操作系统的事件 `events`
* 初始化当前感兴趣的事件 `pevents`

注意：`pevents` 字段中感兴趣的事件并不会实时同步到操作系统，而是需要在 `Poll I/O` 阶段才进行同步



#### **注册事件**

[`uv__io_start()`](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/src/unix/core.c#L878-L903) 函数的作用是注册事件

```c
// node-18.15.0/deps/uv/src/unix/core.c
void uv__io_start(uv_loop_t* loop, uv__io_t* w, unsigned int events) {  
    ...
      
    // 设置当前感兴趣的事件（但还没有同步到操作系统，而是在 Poll I/O 阶段才进行同步 ）
    w->pevents |= events;  
  
    // 扩容 loop->watchers
    maybe_resize(loop, w->fd + 1); 
  
  #if !defined(__sun)
  /* The event ports backend needs to rearm all file descriptors on each and
   * every tick of the event loop but the other backends allow us to
   * short-circuit here if the event mask is unchanged.
   */
  // 事件没有变化则直接返回 
  if (w->events == w->pevents)
    return;
#endif
  
    // IO 观察者如果还没插入队列则插入 IO 观察者队列，等待 Poll I/O 阶段的处理  
    if (QUEUE_EMPTY(&w->watcher_queue))  
        QUEUE_INSERT_TAIL(&loop->watcher_queue, &w->watcher_queue);  
  
    // 保存映射关系，事件触发时通过 fd 获取对应的 IO 观察者，见 Poll IO 阶段的处理逻辑
    if (loop->watchers[w->fd] == NULL) {  
        loop->watchers[w->fd] = w;  
        loop->nfds++;  
    }  
}  
```

主要逻辑如下

* 设置当前感兴趣的事件 `pevents`（但是还没有同步到操作系统，而是在 `Poll I/O` 阶段才进行同步）
* 把一个 `IO` 观察者插入到事件循环的观察者队列 `&loop->watcher_queue` 中，然后 `libuv` 在 `Poll I/O` 阶段会处理这个队列的数据，比如注册到操作系统中
* 在事件循环的 `watchers` 数组中保存一个映射关系，当从事件驱动模块返回时，`libuv` 会根据拿到的 `fd` 从 `watchers ` 中找到对应的 `IO` 观察者，从而执行回调



#### **注销事件**

[`uv__io_stop()`](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/src/unix/core.c#L906-L935) 函数可以修改 `IO` 观察者感兴趣的事件，注销事件

```c
// node-18.15.0/deps/uv/src/unix/core.c
void uv__io_stop(uv_loop_t* loop, uv__io_t* w, unsigned int events) {
  ...
    
  // 清除不感兴趣的事件
  w->pevents &= ~events;
  
  // 如果当前没有感兴趣的事件，则移出 IO 观察者队列，把 loop->watchers[w->fd] 置 NULL
  if (w->pevents == 0) {
    // 移出 IO 观察者队列
    QUEUE_REMOVE(&w->watcher_queue);
    QUEUE_INIT(&w->watcher_queue);
    w->events = 0;
    
    // loop->watchers[w->fd] 置 NULL，但是不会修改操作系统的数据，所以之前感兴趣的事件还是可能触发，
    // 当在Poll IO 阶段时，需要通过 loop->watchers[w->fd] 为 NULL 进行过滤
    if (w == loop->watchers[w->fd]) {
      loop->watchers[w->fd] = NULL;
      loop->nfds--;
    }
  }
  // 如果当前还有感兴趣的事件，并且还没有在 IO 观察者队列，则插入，等待 Poll IO阶段修改操作系统数据。
  else if (QUEUE_EMPTY(&w->watcher_queue))
    QUEUE_INSERT_TAIL(&loop->watcher_queue, &w->watcher_queue);
}
```

主要逻辑如下

1. 修改当前感兴趣的事件 `pevents`

1. 如果当前没有感兴趣的事件（`w->pevents == 0`），则需要移出 `IO` 观察者队列，并删除 `fd` 和 `IO` 观察者的映射关系，但是不会实时地从事件驱动模块中注销这个 `fd` 对应的事件

1. 如果当前还有感兴趣的事件 并且 `IO` 观察者还没有插入 `IO` 观察者队列，把一个 `IO` 观察者插入到事件循环的观察者队列 `&loop->watcher_queue` 中，否则不需要操作，因为修改的事件会在 `Poll I/O` 阶段同步到操作系统，只需要保证 `IO` 观察者在队列里就行


注意：当调用 `uv__io_stop()` 注销事件时，注销的事件可能已经触发，比如在  `回调 1` 里注销了 `回调 2` 的事件，所以在 `Poll I/O` 阶段时需要根据 `pevents` 进行判断，过滤已经被注销的事件，不去执行相应的回调。



#### **关闭** **IO** **观察者**

[`uv__io_close()`](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/src/unix/core.c#L937-L944) 函数的作用是关闭 `IO` 观察者

```c
// node-18.15.0/deps/uv/src/unix/core.c
void uv__io_close(uv_loop_t* loop, uv__io_t* w) {
  // 注销事件
  uv__io_stop(loop, w, POLLIN | POLLOUT | UV__POLLRDHUP | UV__POLLPRI);
  
  // 移出 pending 队列，如果在的话
  QUEUE_REMOVE(&w->pending_queue);
  
  if (w->fd != -1)
    uv__platform_invalidate_fd(loop, w->fd);
}
```

主要逻辑如下

* 首先调用了 `uv__io_stop()` 注销所有事件
* 移出 `pending_queue` 队列
* 最后调用了 `uv__platform_invalidate_fd()` 函数



**uv__platform_invalidate_fd()函数**

[`uv__platform_invalidate_fd()`](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/src/unix/epoll.c#L49-L79)函数的源码如下

```c
// node-18.15.0/deps/uv/src/unix/epoll.c
void uv__platform_invalidate_fd(uv_loop_t* loop, int fd) {
    struct epoll_event* events;
    struct epoll_event dummy;
    uintptr_t i;
    uintptr_t nfds;
  
    // events 和 nfds 为本轮 Poll IO 阶段返回的，代表触发的事件和个数
    events = (struct epoll_event*) loop->watchers[loop->nwatchers];
    nfds = (uintptr_t) loop->watchers[loop->nwatchers + 1];
  
    // 修改对应的结构体的 fd 为 -1
    for (i = 0; i < nfds; i++)
      if (events[i].data.fd == fd)
        events[i].data.fd = -1;

  /* Remove the file descriptor from the epoll.
   * This avoids a problem where the same file description remains open
   * in another process, causing repeated junk epoll events.
   *
   * We pass in a dummy epoll_event, to work around a bug in old kernels.
   */
  if (loop->backend_fd >= 0) {
    /* Work around a bug in kernels 3.10 to 3.19 where passing a struct that
     * has the EPOLLWAKEUP flag set generates spurious audit syslog warnings.
     */
    memset(&dummy, 0, sizeof(dummy));
    
     // 从事件驱动模块注销 fd
    epoll_ctl(loop->backend_fd, EPOLL_CTL_DEL, fd, &dummy);
  }
}
```

主要逻辑如下

* 修改 `Poll I/O` 阶段返回的事件结构体的 `fd` 为 `-1`
* 接着从事件驱动模块注销这个 `fd` 感兴趣的所有事件

注意：因为删除操作系统中感兴趣的事件只能保证后面不会触发，但是这个事件可能已经在本轮 `Poll I/O` 阶段中触发，并等待处理，比如回调 `1` 里关闭了回调 `2` 的 `IO` 观察者，所以 `Poll I/O` 阶段需要根据 `fd` 是否为 `-1` 进行过滤。



**`uv__io_stop()` 跟 `uv__io_close()` 的区别**

* `uv__io_stop()`不会实时地注销事件驱动模块中这个 `fd` 对应的事件，因为 `uv__io_stop` 的语义是注销事件，后续还可以通过 `uv__io_start` 重新注册事件；所以实现上先不注销事件驱动模块中的 `fd` 事件，可以减少一次系统调用，如果后续没有调用 `uv__io_start`，而又有事件触发时，`libuv` 才会真正注销事件驱动中该 `fd` 对应的事件
* `uv__io_close()` 会实时地注销事件驱动模块中这个 `fd` 对应的事件；因为 `uv__io_close()` 的语义是这个 `IO` 观察者不会再被使用了，并且通常调用 `uv__io_close()` 后 `libuv` 会马上关闭 `fd`，如果这时不实时地调用事件驱动模块注销该 `fd` 的事件，那就没有机会注销了。(当我们调用事件驱动模块注销一个 `fd` 的事件时，如果这个 `fd` 已经被关闭，操作系统会报错)



### 4.5.2 Poll I/O具体处理

上面了解了 `IO` 观察者，下面开始分析 `Poll I/O` 阶段。



#### **uv__io_poll() 函数**

`Poll I/O` 具体处理逻辑在 [`uv__io_poll()`](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/src/unix/epoll.c#L103-L421) 函数，源码如下

> 应用订阅的事件会首先保存到 `IO` 观察者中，然后在 `Poll I/O` 阶段被处理，而不是在订阅时就实时处理，这样可以尽可能避免过多系统调用

```c
// node-18.15.0/deps/uv/src/unix/epoll.c
void uv__io_poll(uv_loop_t* loop, int timeout) {
  
    ...
  
    uv__io_t* w;
  
    ...
      
    // 遍历 IO 观察者队列
    while (!QUEUE_EMPTY(&loop->watcher_queue)) {
        // 取出当前头节点
        q = QUEUE_HEAD(&loop->watcher_queue);
  
        // 移出队列
        QUEUE_REMOVE(q);
  
        // 重置节点的前后指针
        QUEUE_INIT(q);
  
        // 通过结构体成功获取结构体首地址
        w = QUEUE_DATA(q, uv__io_t, watcher_queue);
  
        // 设置当前感兴趣的事件
        e.events = w->pevents;
  
        // 记录 fd，事件触发后再通过 fd 从 loop->watchs 字段里找到对应的 IO 观察者
        e.data.fd = w->fd;
  
        // w->events 为 0 ，则新增，否则修改
        if (w->events == 0)
            op = EPOLL_CTL_ADD;
        else
            op = EPOLL_CTL_MOD;
  
        // 修改 epoll 的数据
        epoll_ctl(loop->backend_fd, op, w->fd, &e);
      
        // 记录当前最新的状态
        w->events = w->pevents;
    }
  
    ...
      
    // 阻塞等待事件 或 超时，events 保存就绪的事件 
    // epoll_wait 返回值 nfds 表示有多少个 fd 的事件触发了
    nfds = epoll_wait(loop->backend_fd,
                        events,
                        ARRAY_SIZE(events),
                        timeout);
    ...
      
      
    // epoll_wait 可能会引起主线程阻塞，所以wait 返回后需要更新当前的时间，否则在使用的时候时间差会比较大。因为libuv 会在每轮时间循环开始的时候缓存当前时间这个值。其他地方直接使用，而不是每次都去获取
    SAVE_ERRNO(uv__update_time(loop));

   
    // 遍历有事件触发的 fd
    for (i = 0; i < nfds; i++) {
       // 哪个 fd 触发了什么事情
      pe = events + i;
      fd = pe->data.fd;

      /* Skip invalidated events, see uv__platform_invalidate_fd */
      if (fd == -1)
        continue;

      assert(fd >= 0);
      
      // 根据 fd 获取 IO 观察者
      assert((unsigned) fd < loop->nwatchers);

      w = loop->watchers[fd];

      if (w == NULL) {
        /* File descriptor that we've stopped watching, disarm it.
         *
         * Ignore all errors because we may be racing with another thread
         * when the file descriptor is closed.
         */
        epoll_ctl(loop->backend_fd, EPOLL_CTL_DEL, fd, pe);
        continue;
      }

      /* Give users only events they're interested in. Prevents spurious
       * callbacks when previous callback invocation in this loop has stopped
       * the current watcher. Also, filters out events that users has not
       * requested us to watch.
       */
      pe->events &= w->pevents | POLLERR | POLLHUP;

      /* Work around an epoll quirk where it sometimes reports just the
       * EPOLLERR or EPOLLHUP event.  In order to force the event loop to
       * move forward, we merge in the read/write events that the watcher
       * is interested in; uv__read() and uv__write() will then deal with
       * the error or hangup in the usual fashion.
       *
       * Note to self: happens when epoll reports EPOLLIN|EPOLLHUP, the user
       * reads the available data, calls uv_read_stop(), then sometime later
       * calls uv_read_start() again.  By then, libuv has forgotten about the
       * hangup and the kernel won't report EPOLLIN again because there's
       * nothing left to read.  If anything, libuv is to blame here.  The
       * current hack is just a quick bandaid; to properly fix it, libuv
       * needs to remember the error/hangup event.  We should get that for
       * free when we switch over to edge-triggered I/O.
       */
      if (pe->events == POLLERR || pe->events == POLLHUP)
        pe->events |=
          w->pevents & (POLLIN | POLLOUT | UV__POLLRDHUP | UV__POLLPRI);

      if (pe->events != 0) {
        /* Run signal watchers last.  This also affects child process watchers
         * because those are implemented in terms of signal watchers.
         */
        if (w == &loop->signal_io_watcher) {
          have_signals = 1;
        } else {
          uv__metrics_update_idle_time(loop);
          // 执行回调
          w->cb(loop, w, pe->events);
        }

        nevents++;
      }
    }

}
```

* 首先 `while` 循环遍历 `IO` 观察者队列，处理 IO 观察者注册的事件，修改 `epoll` 的数据（即注册 `fd` 感兴趣的事件），`epoll` 需要针对每个 `IO` 观察者调用一次系统调用 `epoll_ctl()`

* 调用 `epoll_wait()` 阻塞等待事件或超时，返回事件和个数，从事件结构体找到关联的 `IO` 观察者，然后执行对应回调即可

  

#### **等待事件触发或超时**

```c
// 阻塞等待事件或超时，events 保存就绪的事件，nfds 保存就绪的事件个数
int nfds = epoll_wait(loop->backend_fd, events, ARRAY_SIZE(events),  timeout);
```

`epoll_wait()` 用于阻塞等待事件的触发，这就是 `阻塞/唤醒` 机制的实现。`epoll_wait()` 除了在事件触发时返回，还支持超时，也就是如果超过一定的时间还没有事件触发，则返回。阻塞时间的计算规则如下：

1. `0` 代表不阻塞

1. `大于 0` 代表最多阻塞一段时间

1. `小于 0` 代表一直阻塞，直到有事件发生



这个超时时间由`libuv` 的 `uv_backend_timeout()` 实现，`libuv` 的各个阶段中，只有 `prepare` 和 `check` 阶段的任务不会影响 `Poll I/O` 阶段的超时时间的计算，源码如下

```c
int uv_backend_timeout(const uv_loop_t* loop) {
  if (QUEUE_EMPTY(&loop->watcher_queue))
    return uv__backend_timeout(loop);
  /* Need to call uv_run to update the backend fd state. */
  return 0;
}

static int uv__backend_timeout(const uv_loop_t* loop) {
  if (loop->stop_flag == 0 &&
      /* uv__loop_alive(loop) && */
      (uv__has_active_handles(loop) || uv__has_active_reqs(loop)) && // 有活跃的事件要处理
      QUEUE_EMPTY(&loop->pending_queue) && // pending阶段没有任务
      QUEUE_EMPTY(&loop->idle_handles) && // idle阶段没有任务
      loop->closing_handles == NULL) // close阶段没有任务
    return uv__next_timeout(loop); // 返回下一个最早过期的定时器的时间间隔，即最早超时的定时器节点，以此作为事件循环阻塞的最长时间，因为要保证定时器按时执行，不能一直阻塞在等待IO阶段
  
  // 以下即为不阻塞的，根据以上条件可知，不阻塞的情况如下
  // 1. stop_flag不为0时，不阻塞
  // 2. 有活跃的事件要处理，不阻塞
  // 3. pending阶段没有任务，不阻塞
  // 4. idle 阶段有任务，不阻塞，尽快返回处理 idle 任务
  // 5. close 阶段有任务，不阻塞
  return 0;
}

// uv__next_timeout() 函数用于计算阻塞时间，主要逻辑时判断是否有定时器，有则返回最快到期节点的时间
int uv__next_timeout(const uv_loop_t* loop) {
  const struct heap_node* heap_node;
  const uv_timer_t* handle;
  uint64_t diff;
  heap_node = heap_min(timer_heap(loop));
  // 没有定时器则事件循环一直阻塞
  if (heap_node == NULL)
    return -1;
  handle = container_of(heap_node, uv_timer_t, heap_node);
  // 已经超时的节点会被 timer 阶段执行，这里不太可能出现
  if (handle->timeout <= loop->time)
    return 0;
  // 计算最快超时的节点还需要多久超时，以此作为事件循环的最长阻塞时间
  diff = handle->timeout - loop->time;
  if (diff > INT_MAX)
    diff = INT_MAX;
  return (int) diff;
}
```



#### **处理触发的事件**

从上面的分析中可知， `epoll_wait` 返回时，可能是超时也可能是有事件触发，具体需要根据 `epoll_wait` 的返回值判断，`epoll_wait` 返回值表示有多少个 `fd` 的事件触发了，通过 `epoll_wait()` 返回的事件和个数，从事件结构体找到关联的 `IO` 观察者，然后执行对应回调即可

```c
// epoll_wait 可能会引起主线程阻塞，所以wait 返回后需要更新当前的时间，否则在使用的时候时间差会比较大。因为libuv 会在每轮时间循环开始的时候缓存当前时间这个值。其他地方直接使用，而不是每次都去获取。 
SAVE_ERRNO(uv__update_time(loop));


// 遍历有事件触发的 fd
for (i = 0; i < nfds; i++) {
    // 哪个 fd 触发了什么事情
    pe = events + i;
    fd = pe->data.fd;
  
    // 根据 fd 获取 IO 观察者
    w = loop->watchers[fd];
  
    // 执行回调
    if (pe->events != 0) {
        w->cb(loop, w, pe->events);
    }
}
```



#### **删除事件的处理**

如果之前订阅的事件触发了，但目前又不感兴趣已经注销了，要怎么过滤这种过期的事件

> 这个过期事件可能来源于 `libuv` 还没有同步最新的数据到操作系统；也可能来源于前面的回调删除了后面回调的事件或 `IO` 观察者，比如在 `回调 1` 里注销了 `回调 2` 的事件，那么就不需要执行 `回调 2` 了。

处理方式如下：

```c
for (i = 0; i < nfds; i++) {
    pe = events + i;
    fd = pe->data.fd;
  
    // fd 无效则不需要处理了（调用了 uv__io_close）
    if (fd == -1)
        continue;
  
    // IO 观察者已经被删除则不需要执行回调了（调用了 uv__io_stop），并且删除操作系统的数据
    w = loop->watchers[fd];
    if (w == NULL) {
        epoll_ctl(loop->backend_fd, EPOLL_CTL_DEL, fd, pe);
        continue;
    }
  
    // 和当前感兴趣的事件 w->pevents 进行 & 操作，保证剩下的事件 pe->events 已经触发且感兴趣的
    pe->events &= w->pevents | POLLERR | POLLHUP;
    if (pe->events != 0) {
        w->cb(loop, w, pe->events);
    }
}
```

从代码中可知，`libuv` 会根据 `uv__io_stop` 和 `uv__io_close` 设置的各种标记进行过滤，避免处理过期的事件。



**`Poll I/O` 阶段的整体流程图**



![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/Poll%20IO%E9%98%B6%E6%AE%B5.png)



## 4.6 第6阶段：check（复查）阶段

`check` 阶段主要处理 复查事件， `setImmediate()` 方法的回调就是在该阶段执行。只有 `check` 时，会阻塞在 `Poll I/O` 阶段



## 4.7 第7阶段：close（收尾）阶段

`close` 阶段主要处理一些收尾事件； `uv_close()` 函数支持传入一个回调，它会往 `close` 队列插入要处理的事件，这个回调就会在 `close` 阶段被执行（比如关闭 `tcp` 连接）



**`uv__run_closing_handles()`函数**

该函数就是 `close` 阶段的主要处理，[源码](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/src/unix/core.c#L324-L336)如下

```c
// node-18.15.0/deps/uv/src/unix/core.c
static void uv__run_closing_handles(uv_loop_t* loop) {
    uv_handle_t* p;
    uv_handle_t* q;
    p = loop->closing_handles;
    loop->closing_handles = NULL;
    while (p) {
        q = p->next_closing;
        uv__finish_close(p); // 执行 close 阶段的回调
        p = q;
    }
}

// 执行 close 阶段的回调
static void uv__finish_close(uv_handle_t* handle) {
    // ...
    handle->flags |= UV_HANDLE_CLOSED;
    // ...
  
    uv__handle_unref(handle);
  
    // 移出 handle 队列
    QUEUE_REMOVE(&handle->handle_queue);
  
    // 执行回调
    if (handle->close_cb) {
        handle->close_cb(handle);
    }
}
```



**close()函数**

主要是往 `close` 队列插入要处理的事件，[源码](https://github.com/nodejs/node/blob/v18.15.0/deps/uv/src/unix/core.c#L115-L194)如下

```c
// node-18.15.0/deps/uv/src/unix/core.c
void uv_close(uv_handle_t* handle, uv_close_cb close_cb) {
    assert(!uv__is_closing(handle));

    // 设置 handle 的回调和状态
    handle->flags |= UV_HANDLE_CLOSING;
    handle->close_cb = close_cb;
  
    // 根据 handle 类型执行不同的 close 函数，通常是 stop 这个 handle
    switch (handle->type) {
        case UV_PREPARE:
            uv__prepare_close((uv_prepare_t*)handle); // 这里是 prepare 阶段 的 close 函数
                break;
        case UV_CHECK:
            uv__check_close((uv_check_t*)handle); // check 阶段的 close函数
            break;
        ...
        // 其他句柄类型  
          
        default:
            assert(0);
    }
  
    // 以头插法往 close 队列插入一个任务
    uv__make_close_pending(handle);
}

// 以头插法往 close 队列插入一个任务
void uv__make_close_pending(uv_handle_t* handle) {
    handle->next_closing = handle->loop->closing_handles;
    handle->loop->closing_handles = handle;
}
```



# 5 总结

本文主要讲解了事件循环，主要有以下重点

* `Node.js` 中事件循环的实现

* 底层 `libuv` 事件循环的 7 个阶段的讲解 ，以及各个阶段分别处理的事件

  

# 6 参考

* [libuv v1.44.2源码](https://github.com/libuv/libuv/tree/v1.44.2)
* [libuv Design overview](http://docs.libuv.org/en/v1.x/design.html)
* [Node.js v18.15.0源码](https://link.zhihu.com/?target=https%3A//github.com/nodejs/node/tree/v18.15.0)
* [深入剖析 Node.js 底层原理-Liuv事件循环](https://juejin.cn/book/7171733571638738952/section/7174421241225281566?enter_from=course_center&utm_source=course_center)
* [趣学Node.js-事件循环与异步IO](https://juejin.cn/book/7196627546253819916/section/7196992628036993028)
* [[Nodejs原理] 核心库Libuv入门（Hello World篇）](https://cloud.tencent.com/developer/article/1484459)
* [libuv例子](https://github.com/sinkhaha/libuv-demo/tree/main/timer)
