# 1 前言

上一章讲了 `Egg.js` 中[多进程的模型](https://zhuanlan.zhihu.com/p/699815241)，今天主要讲下 `Egg.js` 各个进程间的通信，如有错误之处，欢迎指正。



**此文所涉及源码库的版本**

* [egg-cluster](https://link.zhihu.com/?target=https%3A//github.com/eggjs/egg-cluster) : 2.2.1
* [egg](https://link.zhihu.com/?target=https%3A//github.com/eggjs/egg)： 3.23.0
* [sendmessage](https://github.com/node-modules/sendmessage)：2.0.0



# 2 Egg 进程间的通信

`Egg` 是一个多进程架构，所以各个进程间难免需要互相通信，它们之间的通信主要通过 `IPC` 通道实现

* `IPC` 通道主要用于 `Node.js` 父子进程间的通信。如 `Master` 和 `Agent` 的通信，`Master` 和 `App Worker` 的通信，`Master` 和 `Parent` 的通信（如果是通过 `egg-scripts` 启动的 `egg` 项目，那 `Parent` 进程就是 `egg-scripts` 进程）
* `Agent` 和 `App Worker` 进程之间不是父子关系，`App Worker` 和 `App Worker` 进程之间也不是父子进程，但是它们之间也可以通信，因为它们有公共的父进程 `Master`，可以通过 `IPC` 通道先把消息发给 `Master` 进程，然后经过 `Master` 进程中转



**各个进程通信的总体流程图如下**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/rust-chengxusheji/%E9%80%9A%E4%BF%A1%E6%B5%81%E7%A8%8B%E5%9B%BE.drawio.png)



## 2.1 Messenger 消息发规则基类

### 2.1.1 Messenger 类的作用

> `Messenger` 由 `egg-cluster` 包提供

`Messenger` 类封装了 `Egg` 中多进程通信的转发规则。`Messenger` 提供了各个进程消息转发的方法支持 `Parent`、`Master`、`Agent`、`App` 互相通信。`egg-cluster` 的 `Master` 主进程里就维护了一个 `messenger` 对象



**messenger 的进程通信关系如下图**

```JS
             ┌────────┐
             │ parent │
            /└────────┘\
           /     |      \
          /  ┌────────┐  \
         /   │ master │   \
        /    └────────┘    \
       /     /         \    \
     ┌───────┐         ┌───────┐
     │ agent │ ------- │  app  │
     └───────┘         └───────┘
     
```

分析如下

* `parent` 进程： 即 `egg-bin/egg-scripts` 等“启动 `egg-cluster` 中 `Master` 进程”的进程

* `master` 进程： 即 `egg-cluster` 的 `Master` 主进程

  > `master` 继承了 `events` 模块，所以 `master` 自己可以通过事件机制来进行消息的通知

* `agent` 进程： 即 `master` 主进程通过 `child_process.fork` 出来的 `Agent` 子进程 

* `app` 进程： 即  `master` 主进程通过 `cfork` 包（`cluster.fork`）出来的 `App Worker` 子进程

由图可知，各个进程间都是可以互相通信的。因为 `agent` 和各个 `app worker`之间 、` app worker` 和 `app worker ` 之间、`parent` 和 `agent/app` 之间没有直接的进程关系，所以无法直接进行通信，但是它们有公共的进程 `Master`，所以可以间接经过 `Master` 转发进行通信



### 2.1.2 Messenger 的分析

#### **2.1.2.1 Node.js 的 IPC 通信**

在分析 `Messenger` 的通信规则前，我们先来看下 `Node.js` 中进程通信方式： `IPC`



**什么是 IPC**

`IPC` 的全称是 `Inter-Process Communication`，即进程间通信。`Node.js` 中实现 `IPC` 通道的是管道（`pipe`）技术。但此管道非彼管道，在 `Node.js` 中管道是个抽象层面的称呼，具体细节实现由 `libuv` 提供，在 `Windows` 下由命名管道（`named pipe`）实现，`*nix` 系统则采用 `Unix Domain Socket` 实现。



进程间通信的目的是为了让不同的进程能够`互相访问资源`并进行协调工作。



**Node.js 怎么使用 IPC 通信**

通过 `IPC` 通道，`Node.js` 父子进程之间才能通过 `message` 事件和 `send()` 传递消息。

> 在 `Node.js` 中，`IPC` 通道被抽象为 `Stream` 对象，在调用 `send()` 时发送数据（类似于 `write()`），接收到的消息会通过 `message` 事件（类似于 `data`）触发给应用层。

在父进程中：

1. 发消息给子进程：`子进程.send()`
2. 接消息来自子进程：`子进程.on('message')`

在子进程中:

1. 发消息给父进程：`process.send()`
2. 接消息来自父进程：`process.on('message')`



例子：`parent.js` 父进程文件

```js
// parent.js 父进程文件
const path = require('path');
const childProcess = require('child_process');

const cp = childProcess.fork(path.join(__dirname, 'child.js'));

// 给子进程发消息
cp.send({ msg: 'ping' });

// 接收子进程的消息
cp.on('message', (msg) => {
    console.log('parent接收到child消息', msg);
});
```

`child.js` 子进程文件

```js
// child.js 子进程文件
// 接收父进程的消息
process.on('message', (msg) => {
    console.log('child接收到parent的消息', msg);
    
    // 给父进程发消息
    process.send({ msg: 'pong' });
});

// 打印日志如下
// child接收到parent的消息 { msg: 'ping' }
// parent接收到child消息 { msg: 'pong' }
```



#### 2.1.2.2 Messenger 的核心逻辑

> 源码位于 `egg-cluster/lib/utils/messenger.js`

1. 构造方法中使用 `process` 监听 `message` 事件，用于接收来自父进程的消息， `Master` 的父进程是 `Parent` ，所以这里 `Master` 会接收来自 `Parent` 的消息
2. 监听 `disconnect`，即 `Master` 监听与 `Parent`  的 `IPC` 通信通道是否关闭
3. 提供 `send()` 方法统一处理消息的转发，发送规则如下

* **发送给 `Master` 的消息**：发送给 `Master` 直接用 `events` 模块的事件机制，因为 `Agent/App Worker` 等所有发给 `Master` 进程的消息最终都会在 `Master` 进程上被接收，接着再转发给 `Master` 进程时，直接用事件机制即可
* **发送给 `Parent` 的消息**：使用 `process.send()` 通过 `Master` 发送给 `Parent` 父进程，因为只有 `Master` 和 `Parent` 有父子进程关系，它们之间才能直接通信
* **发送给 `App` 的消息**：因为有多个 `App` 进程，所以发送给 `App` 时可以指定进程 `id` 表示发给哪个进程，使用 `worker.send()`，表示 `Master` 发消息给 `App Worker` 子进程
* **发送给 `Agent` 的消息**：使用 `agent.send(data)`，表示 `Master` 发消息给 `Agent` 子进程



**Messenger 源码如下**

```js
// egg-cluster/lib/master.js
class Master extends EventEmitter {
     constructor(options) {
          // Master 进程拥有messenger对象
          this.messenger = new Messenger(this, this.workerManager);
     }
}

// egg-cluster/lib/utils/messenger.js
class Messenger {
    // 1. master接收parent的消息
    constructor(master) {
        this.master = master;
        // 如果node进程是使用ipc通道衍生的，可以用process.send方法发消息给父进程，如果不是通过ipc通道衍生的，则process.send是undefined
       // 如用egg-bin的dev命令启动
       this.hasParent = !!workerThreads.parentPort || !!process.send;
      
        // master接收到parent的消息
        // process是当前进程，即parent，例如egg-bin/egg-scripts
        process.on('message', msg => {
            msg.from = 'parent';
            this.send(msg);
        });
      
        // master监听与parent通信通道的关闭(如parent进程挂了)
        process.once('disconnect', () => {
            this.hasParent = false;
        });
    }

    // 2. 发送消息的统一转发处理方法
    send(data) {
        // 默认master 
        if (!data.from) {
            data.from = 'master';
        }

        // 根据进程id转发 
        // recognise receiverPid is to who
        if (data.receiverPid) {
            if (data.receiverPid === String(process.pid)) {
                data.to = 'master';
            } else if (data.receiverPid === String(this.master.agentWorker.pid)) {
                data.to = 'agent';
            } else {
                data.to = 'app';
            }
        }

        // 默认的转发规则
        // default from -> to rules
        if (!data.to) {
            if (data.from === 'agent') data.to = 'app';
            if (data.from === 'app') data.to = 'agent';
            if (data.from === 'parent') data.to = 'master';
        }

        // app或agent 到 master 使用EventEmitter事件通信 
        // app -> master
        // agent -> master
        if (data.to === 'master') {
            debug('%s -> master, data: %j', data.from, data);
            // app/agent to master
            this.sendToMaster(data);
            return;
        }

        // master或app或agent 到parent 使用 ipc通信(基于process.send)
        // master -> parent
        // app -> parent
        // agent -> parent
        if (data.to === 'parent') {
            debug('%s -> parent, data: %j', data.from, data);
            this.sendToParent(data);
            return;
        }

        // 发消息给 app
        // parent -> master -> app
        // agent -> master -> app
        if (data.to === 'app') {
            debug('%s -> %s, data: %j', data.from, data.to, data);
            this.sendToAppWorker(data);
            return;
        }

        // 发消息给 agent
        // parent -> master -> agent
        // app -> master -> agent，可能不指定 to
        if (data.to === 'agent') {
            debug('%s -> %s, data: %j', data.from, data.to, data);
            this.sendToAgentWorker(data);
            return;
        }
    }
  
}
```



## 2.2 App 和 Agent 的消息通信类

因为用户对于底层 `Master` 和 `App/Agent` 的之间通信是无感的，我们在实际业务开发中，也不需要关心它们之间是怎么通信的，我们主要关心的是面向用户的 `App` 进程和 `Agent` 进程之间的通信。`Egg` 针对 `Agent` 和 `App` 之间的通信，提供了一系列友好的 `API`，使我们可以很方便的使用这些 `API` 进行通信。



### 2.2.1 Messenger 类的作用

> `Messenger` 由 `egg` 包提供，注意这里的 `Messenger` 跟上面讲的 `Messenger` 基础消息发送类不一样，这里的源码位于 `egg/lib/core/messenger/ipc.js`

`Messenger` 类是 `Agent` 和 `App` 之间通信的通道，它封装了 `Agent` 和 `App` 之间通信的 `API` 。`Egg` 在 `App` 和 `Agent` 实例拥有 `messenger` 对象，实际它们之间的通信通过 `Master` 进程进行中转。消息发送示意图如下

```js
广播消息：Agent => 所有 Worker
                  +--------+            +-------+
                  | Master |<-----------| Agent |
                  +--------+            +-------+
                 /    |     \
                /     |      \
               /      |       \
              /       |        \
             v        v         v
  +----------+   +----------+   +----------+
  | Worker 1 |   | Worker 2 |   | Worker 3 |
  +----------+   +----------+   +----------+

指定接收方：一个 Worker => 另一个 Worker
                  +--------+            +-------+
                  | Master |------------| Agent |
                  +--------+            +-------+
                 ^    |
                 |    | 发送至 Worker 2
                 |    |
                 |    v
  +----------+   +----------+   +----------+
  | Worker 1 |   | Worker 2 |   | Worker 3 |
  +----------+   +----------+   +----------+
```



### 2.2.2 Messenger 的 API 

**发送消息的 API**

> 以下 `API` 在 `App` 和 `Agent` 进程对象上都可以调用

1、`broadcast(action, data)`: 广播，向所有的 `agent / app` 进程发送消息（包括自己）

2、`sendToApp(action, data)`: 发送至所有的 `app` 进程

- `app` 上调用：即发送至自己与其他 `app`
- `agent` 上调用：则发送至所有 `app` 进程

3、`sendToAgent(action, data)`: 发送消息至 `agent` 进程

- `app` 上调用：即发送至 `agent`
- `agent` 上调用：即发送至自己

4、`sendRandom(action, data)`:

- `agent` 上调用：即随机向某 app 进程发送消息（由 `master` 控制）
- `app` 上调用：则发消息给agent进程（此时作用等价于调用 `sentToAgent` 方法）

5、`sendTo(pid, action, data)`: 向指定进程发送消息



使用如下

```js
// app.js
module.exports = (app) => {
  // 注意：只有在 egg-ready 事件后才能发送消息
  app.messenger.once('egg-ready', () => {
    // 发 agent-event 事件消息给 agent 进程
    app.messenger.sendToAgent('agent-event', { foo: 'bar' });
    // 发 app-event 事件消息给 app 进程
    app.messenger.sendToApp('app-event', { foo: 'bar' });
  });
};
```



**接收方式**

接收时，使用 `events` 模块的事件机制监听 `messenger` 上的相应事件即可收到其他进程发送的消息，例如

```js
// app 监听名为 “app-event” 的事件
app.messenger.on("app-event", (data) => {
  // 处理数据
});
app.messenger.once("app-event", (data) => {
  // 处理数据
});
```



**注意：**只有在接到 `egg-ready` 消息后方才可以发送消息。当 `Master` 确认所有 `Agent` 和 `Worker` 启动成功并就绪后，才会通过 `messenger` 发送 `egg-ready` 通知每个 `Agent` 和 `Worker`，表示一切就绪，此时才可使用发送消息 `API`

* 如果 `app` 还没全部启动成功，即还没收到 `egg-ready` 事件，那 `app` 上调用发消息 `API`，可能会因为个别`app` 进程还没启动而导致接收不到信息
* 如果 `app` 还没全部启动成功，即还没收到 `egg-ready` 事件，那 `agent` 上调用发消息 `API`，则会报错



`agent` 上通信方法挂载时机的源码如下

 ```js
// egg/lib/agent.js
class Agent extends EggApplication {
  // 包装了下 agent进程的各个进程发送方法，只有在接收到 egg-ready 事件后，才挂上对应到方法，否则调用方法会报错
  _wrapMessenger() {
    for (const methodName of [
      'broadcast',
      'sendTo',
      'sendToApp',
      'sendToAgent',
      'sendRandom',
    ]) {
      wrapMethod(methodName, this.messenger, this.coreLogger);
    }

    function wrapMethod(methodName, messenger, logger) {
      const originMethod = messenger[methodName];
      messenger[methodName] = function() {
        const stack = new Error().stack.split('\n').slice(1).join('\n');
        logger.warn(
          "agent can't call %s before server started\n%s",
          methodName,
          stack
        );
        originMethod.apply(this, arguments);
      };
      // agent进程接收到应用启动消息egg-ready后才把对应的api方法挂载上去
      messenger.prependOnceListener('egg-ready', () => {
        messenger[methodName] = originMethod;
      });
    }
  }
}
 ```



### 2.2.3 Messenger 类分析

**核心逻辑**

> egg/lib/core/messenger/ipc.js

`agent` 和 `app` 进程都拥有一个`messenger` 对象，可以解决 `agent` 和 `app` 之间的互相通信，其核心逻辑为

* 接收 `app/worker` 启动后的进程 `id`，用于 `sendRandom` 方法对随机进程发送消息

* 使用 `process.on('message')` 接收来自父进程的消息，然后调用 `_onMessage()` 方法处理，里面使用了 `events` 事件机制把消息发到自己的进程（`app` 或 `agent`）

  > 因为此时 `agent` 和 `app` 的父进程是 `master`，所以是 `agent` 或 `app` 接收来自 `master` 进程的消息

* 接着对外提供了一些 `agent` 和 `app` 进程间通信的方法，例如 `broadcast`、`sendToApp`、`sendToAgent`、`sendRandom`、`sendTo`等方法，这些方法最终调用的是 `sendmessage` 模块进行发送消息



**源码分析**

```js
// egg/lib/egg.js/EggApplication
// EggApplication 中实例化了 messenger 对象，而 App 和 Agent 都继承自EggApplication类
class EggApplication extends EggCore {
  constructor(options = {}) {
      // 实例化 App 和 Agent 中的 messenger 对象
      this.messenger = Messenger.create(this);
  }
}

// egg/lib/core/messenger/ipc.js
class Messenger extends EventEmitter {
    constructor() {
        super();
        this.pid = String(process.pid);
        // pids of agent or app maneged by master
        // - retrieve app worker pids when it's an agent worker
        // - retrieve agent worker pids when it's an app worker
        this.opids = [];
      
        // 接收app/worker启动的pid，用于sendRandom方法对随机进程发送消息
        this.on('egg-pids', pids => {
            this.opids = pids;
        });
        
        this._onMessage = this._onMessage.bind(this);
        
        // 接收来自父进程的消息，因为现在父进程是Master，所以是(agent/app)接收来自master进程的消息
        process.on('message', this._onMessage);
      
        // 多线程的处理，这里不涉及不讲
        if (!workerThreads.isMainThread) {
          workerThreads.parentPort.on('message', this._onMessage);
        }
     }
  
     _onMessage(message) {
         if (message && is.string(message.action)) {
             this.emit(message.action, message.data);
         }
     }
  
    // 广播，发消息给所有app进程 和 agent 进程
     broadcast(action, data) {
        this.send(action, data, 'app');
       this.send(action, data, 'agent');
       return this;
     }
  
    // 发消息给指定 app 进程
     sendTo(pid, action, data) {
       sendmessage(process, {
        action,
        data,
        receiverPid: String(pid),
      });
      return this;
     }
  
    // 随机发消息给一个 app 进程
     sendRandom(action, data) {
       if (!this.opids.length) return this;
       const pid = random(this.opids);
       this.sendTo(String(pid), action, data);
       return this;
     }
     
     // 发消息给 app
     sendToApp(action, data) {
       this.send(action, data, 'app');
       return this;
     }
  
    // 发消息给 agent
     sendToAgent(action, data) {
        this.send(action, data, 'agent');
        return this;
     }
    
    // sendmessage 封装了下进程间的通信
    send(action, data, to) {
      // 使用 sendmessage模块
       sendmessage(process, {
         action,
         data,
         to,
       });
       return this;
     }
}
```

### 2.2.4 sendmessage 模块

主要用于子进程向父进程发消息，比如在 `agent` 进程调用，就是向 `master` 进程发消息，源码如下

```js
const { isMainThread, parentPort } = require('worker_threads');

let IS_NODE_DEV_RUNNER = /node\-dev$/.test(process.env._ || '');
if (!IS_NODE_DEV_RUNNER && process.env.IS_NODE_DEV_RUNNER) {
  IS_NODE_DEV_RUNNER = true;
}

module.exports = function send(child, message) {
  if (
    isMainThread // not in worker thread
    && typeof child.postMessage !== 'function' // child is not worker
    && typeof child.send !== 'function'
  ) {
    // not a child process
    return setImmediate(child.emit.bind(child, 'message', message));
  }

  if (IS_NODE_DEV_RUNNER || process.env.SENDMESSAGE_ONE_PROCESS) {
    // run with node-dev, only one process
    // https://github.com/node-modules/sendmessage/issues/1
    return setImmediate(child.emit.bind(child, 'message', message));
  }

  // 线程的一些处理
  // child is worker
  if (typeof child.postMessage === 'function') {
    return child.postMessage(message);
  }
  // in worker thread
  if (!isMainThread) {
    return parentPort.postMessage(message);
  }

  // 区分app和agent用两种不同的方式fork
  // cluster.fork(): child.process is process
  // childprocess.fork(): child is process
  const connected = child.process ? child.process.connected : child.connected;

  // 发消息给child进程
  if (connected) {
    return child.send(message);
  }

  // 记录日志
  // just log warnning message
  const pid = child.process ? child.process.pid : child.pid;
  const err = new Error('channel closed');
  console.warn('[%s][sendmessage] WARN pid#%s channel closed, nothing send\nstack: %s',
    Date(), pid, err.stack);
};
```



## 2.3 例子：Agent 和 App 通信流程

**`agent` 发消息给 `app` 是通过 `master` 中转，详细流程如下：**

* `agent` 发消息给 `app`， `agent` 先把发消息给 `master` 父进程
* `master` 监听到 `agent` 进程的消息后发给 `app` 进程 
* `app` 进程接收到消息后触发 `events` 事件机制，最后 `app` 进程使用 `events` 监听对应事件即可接收到消息



如下图：蓝色部分为屏蔽通信细节的流程，灰色部分有消息通信细节的流程

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/rust-chengxusheji/agent%E5%8F%91%E6%B6%88%E6%81%AF%E7%BB%99app.png)





下面以 `agent` 进程向 `app` 进程发消息为例子讲解下消息是怎么流转的：

1、先在项目的 `agent.js` 调用 `app.messenger.sendToApp(action, data) `方法发消息给全部 `app`，然后在源码加点日志，查看消息打印的日志

```js
// 项目根目录下的agent.js发消息给全部app
class AgentBootHook {
    constructor(agent) {
        this.agent = agent;
      
        this.agent.messenger.once('egg-ready', () => {
            // 测试agent发test-event给所有app
            this.agent.messenger.sendToApp('test-event', { name: 'haha' });
        });
    }
    
}
module.exports = AgentBootHook;

```

2、`agent` 调用的是 `egg` 包 `messenger` 的  `ipc.js` 的 `sendToApp`，源码如下

```js
// egg/lib/core/messenger/ipc.js
sendToApp(action, data) {
    this.send(action, data, 'app');
    return this;
}

send(action, data, to) {
    // 注意process为当前node进程，此时为agent进程
    sendmessage(process, { 
        action,
        data,
        to,
    });
    return this;
  }
```

3、`sendToApp` 实际通过 `sendmessage` 包的 ` send` 方法，即 `agent` 发消息给父进程 `master`，源码如下

```js
// sendmessage/index.js
module.exports = function send(child, message) {
  // 非子进程(agent/app)直接触发emit即可
  if (typeof child.send !== 'function') {
    // not a child process
    return setImmediate(child.emit.bind(child, 'message', message));
  }

  if (IS_NODE_DEV_RUNNER || process.env.SENDMESSAGE_ONE_PROCESS) {
    // run with node-dev, only one process
    // https://github.com/node-modules/sendmessage/issues/1
    return setImmediate(child.emit.bind(child, 'message', message));
  }

  // 区分app和agent用两种不同的方式fork
  // cluster.fork(): child.process is process
  // childprocess.fork(): child is process
  var connected = child.process ? child.process.connected : child.connected;

  if (connected) {
    // 增加调试日志
    // 1. message包 child=97987 { action: 'test-event', data: { name: 'haha' }, to: 'app' }
    if (message.action == 'test-event') {
       console.log(`1. message包`, child.process ? `child.process=${child.process.pid}` : `child=${child.pid}`, message);
    }
    
    // 注意：此时的child为process实例，即子进程发消息给父进程，即agent发消息给master
    return child.send(message);
  }
  ......
};
```

4、`master` 监听到发给 `agent` 进程的消息，然后转发消息给 `app` 进程，源码如下

```js
// egg-cluster/lib/master.js
// master监听到agent进程的消息，转发给app进程
agentWorker.on('message', msg => {
    if (typeof msg === 'string') {
        msg = {
            action: msg,
            data: msg,
        };
    }
      
    // 自己增加一些调试日志
    // 2. agentWorker.onMessage= { action: 'test-event', data: { name: 'haha' }, to: 'app' }
    if (msg.action == 'test-event') {
        console.log(`2. agentWorker.onMessage=`, msg);
    }
   
    msg.from = 'agent';
    this.messenger.send(msg);
});

```

5、通过 `master` 转发给 `app` 的消息，实际调用的是 `worker.send`，即 `master` 发消息给 `app` 进程，源码如下

```js
//  egg-cluster/lib/utils/messenger.js
class Messenger {
   send(data) { 
     if (data.to === 'app') {
       // 自己增加一些调试日志
        // 3. sendToAppWorker = {
        // action: 'test-event',
        // data: { name: 'haha' },
        // to: 'app',
        // from: 'agent'
        // }
        if (data.action == 'test-event') {
            console.log(`3. sendToAppWorker =`, data);
        }
      debug('%s -> %s, data: %j', data.from, data.to, data);
      this.sendToAppWorker(data);
      return;
    }
   }
  
  // 发消息给 app 进程（通过ipc）
  /**
   * send message to app worker
   * @param {Object} data message body
   */
  sendToAppWorker(data) {
    const workerManager = this.workerManager;
    for (const id of workerManager.listWorkerIds()) {
      const worker = workerManager.getWorker(id);
      if (worker.state === 'disconnected') {
        continue;
      }
      // check receiverPid
      if (data.receiverPid && data.receiverPid !== String(worker.workerId)) {
        continue;
      }
      // 发消息给 app 进程
      worker.send(data);
    }
  }
}
```

6、`app` 监听 `master` 发给 `app` 进程的消息，即 `egg` 包中 `ipc.js` 的 `process.on('message', this._onMessage)`，最后在 `app` 进程上使用 `events` 触发事件，这样 `app` 进程使用 `events` 模块监听对应的事件就能接收到消息了，源码如下

```js
// egg/lib/core/messenger/ipc.js
class Messenger extends EventEmitter {
    constructor() {
        super();
        this.pid = String(process.pid);
        this.opids = [];
        this.on('egg-pids', pids => {
            this.opids = pids;
        });
        this._onMessage = this._onMessage.bind(this);
        // 监听当前进程app的消息，即发给app进程的消息
        process.on('message', this._onMessage);
    }
  
    _onMessage(message) {
        if (message && is.string(message.action)) {
            // 增加调试日志
            // 4. _onMessage = test-event { name: 'haha' }
            if (message.action == 'test-event') {
                console.log('4. _onMessage =', message.action, message.data);
            }
      
            // 最终触发事件，在app中监听该事件即可
            this.emit(message.action, message.data);
        }
    }
  
}
```

运行后，打印的日志如下

```bash
1. message包 child=97987 { action: 'test-event', data: { name: 'haha' }, to: 'app' }

2. agentWorker.onMessage= { action: 'test-event', data: { name: 'haha' }, to: 'app' }

3. sendToAppWorker = {
  action: 'test-event',
  data: { name: 'haha' },
  to: 'app',
  from: 'agent'
}

4. _onMessage = test-event { name: 'haha' }
```



# 3 Egg 中各个事件的触发与接收

`Egg` 中有很多事件分散在各个地方，下面整理一份 `Egg` 中各个事件的触发的接收

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/egg/egg%E5%90%84%E8%BF%9B%E7%A8%8B%E7%9B%91%E5%90%AC%E7%9A%84%E6%B6%88%E6%81%AF.png)



# 4 参考

* [Egg文档-多进程模型和进程间通讯](https://eggjs.org/zh-cn/core/cluster-and-ipc.html)

  

