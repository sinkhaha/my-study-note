# 1 前言

在平时工作中，也用 `Egg.js` 好几年了，之前看过一些源码，今天分享一下 `Egg.js` 多进程模型的实现，让大家可以逐步深入理解多进程模型，不只停留在框架的使用层面，`Egg.js` 曾经也是风靡一时，虽然这两年热度有所下降，但是其内部的实现思想还是可以借鉴的，如有错误之处，欢迎大家指正。



**此文所涉及源码库的版本**

* [egg-cluster](https://github.com/eggjs/egg-cluster) : 2.2.1
* [egg](https://github.com/eggjs/egg)： 3.23.0
* [egg-core](https://github.com/eggjs/egg-core): 5.4.1
* [cfork](https://github.com/node-modules/cfork): 1.10.0
* [graceful-process](https://github.com/node-modules/graceful-process): 1.2.0
* [graceful](https://github.com/node-modules/graceful): 1.1.0



# 2 Egg 的整体认识

学习 `Egg.js` 多进程前，先来了解下 `Egg.js` 的全貌。`Egg.js` 框架其实由很多个项目构成，比较核心的有 `egg`、`egg-cluster`、`egg-core`、`egg-scripts`、`egg-bin` 等项目，还有一些 `plugin` 插件，简单介绍如下

* `egg` 项目：它是 `egg` 最上层的应用层，基于 `egg-core` 和 `egg-cluster` 等项目对外提供 `egg` 的功能

* `egg-core` 项目：它是基于 `koa` 的 `egg` 的底层框架，主要实现了 `egg` 的加载器（`Loader`）和 生命周期（`lifecycle`）

* `egg-cluster` 项目：它实现了 `egg` 的多进程，管理内部多进程，负责它们之间的通信

* [egg-bin](https://github.com/eggjs/egg-bin) 项目：它是一些在开发环境常用的工具，常用于本地启动

* [egg-scripts](https://github.com/eggjs/egg-scripts) 项目：它是 `egg` 的进程管理工具，守护我们的 `egg` 项目，用于线上环境，类似 `pm2`

  

**`egg` 项目如下图**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/egg/Egg%E6%95%B4%E4%BD%93%E8%AE%A4%E8%AF%86.png)



# 3 Egg 的多进程模型

[egg-cluster](https://github.com/eggjs/egg-cluster) 项目提供了 `egg` 的多进程模型，多进程模型如下图

```
                +--------+          +-------+
                | Master |<-------->| Agent |
                +--------+          +-------+
                ^   ^    ^
               /    |     \
             /      |       \
           /        |         \
         v          v          v
+----------+   +----------+   +----------+
| Worker 1 |   | Worker 2 |   | Worker 3 |
+----------+   +----------+   +----------+
```

由图中可知，有 `Master`、`Agent`、`Worker` 这 3 种进程角色。`Worker` 和 `Master`，`Agent` 和 `Master` 之间都是双箭头，说明它们可以互相通信，当然 `Agent` 和 `Worker` 之间也可以通信，因为它们有一个公共的父进程 `Master`，所以可以通过 `Master` 进程进行中转，后面会另开一章来讲解通信，这里只关注它们的关系和作用即可



## 3.1 Worker 的作用

图示中画了 3 个 `Worker` 进程，说明 `Worker` 是可以存在多个的，实际上每个 `Worker` 都是我们自己的业务进程，它可用于对外提供 `http` 服务，也可以把它称为 `app` 进程

> 启动的 `app` 进程数量默认等于 `CPU` 的核数，也可以由我们自己指定 `app` 进程数量



## 3.2 Agent 的作用

* 它是后面根据使用场景衍生出来的一个进程，它的数量固定只有 1 个

* 它不对外提供 `http` 服务，一般用于专门处理每个 `Worker` 的公共功能，因为 `Worker` 在设计上就是多进程的，有些功能放在 `Worker` 上可能会导致重复执行，所以需要放到 `Agent` 这样的单进程来处理




## 3.3 Master 的作用

`Master` 进程是一个守护进程，充当守护 `Agent` 和各个 `App Worker` 的角色

- 负责 `Agent` 进程 的启动、退出、重启

- 负责各个 `Worker` 进程的启动、退出、重启

  > 在开发模式下还负责重载 `Worker`，即 `reload-worker`  事件

- 负责中转 `Agent` 和各个 `Worker` 之间的通信

- 负责中转各个 `Worker` 之间的通信



## 3.4 Egg 多进程的优缺点

**优点**

* 内置多进程可以有效的利用多核 `CPU`

* `Agent` 可以处理 `Worker` 不适合处理的业务

* 实现的成熟度很高，用户可以很方便且无感知的使用多进程

  

**缺点**

* 随着容器化的广泛使用，使用容器就可以很方便的开启多进程集群，在某些场景下，`Egg.js` 内置了多进程反而不够灵活

  > 框架都是逐步演进的，内置多进程在当时也是一种伟大的进步，只不过随着时代的发展，有些场景变得不那么适用

* `Egg.js` 的多进程也不是很适合长连接场景，多个 `app` 进程会导致创建太多的长连接

  > 针对长连接场景官方已经有提供了解决方案-[多进程研发模式增强](https://www.eggjs.org/zh-CN/advanced/cluster-client)

* 一个 `Egg.js` 项目，最少会有 3 个进程（1 个 `Master`、1 个 `Agent` 和 1 个 `App Worker`），一些比较简单的业务，其实只要一个 `App Worker` 就可以，也用不到 `Agent` 进程，此时就会比较浪费内存等资源



## 3.5 小结

| 类型     | 进程数量              | 作用                         | 稳定性 | 是否运行业务代码 |
| -------- | --------------------- | ---------------------------- | ------ | ---------------- |
| `Master` | 1                     | 进程管理，进程间消息转发     | 非常高 | 否               |
| `Agent`  | 1                     | 后台运行工作（长连接客户端） | 高     | 少量             |
| `Worker` | 通常设置为 `CPU` 核数 | 执行业务代码                 | 一般   | 是               |

一般业务代码越多的进程，不确定性就越多，稳定性就差，比较容易出问题。因为 `Agent` 只有 1 个进程，所以不建议运行大量业务代码，如果 `Worker` 可以处理的业务还是尽量放在 `Worker` 处理



#  4 进程的启动流程

先看下框架的启动，各个进程是启动顺序，如下图

```
+---------+           +---------+          +---------+
|  Master |           |  Agent  |          |  Worker |
+---------+           +----+----+          +----+----+
     |      fork agent     |                    |
     +-------------------->|                    |
     |      agent ready    |                    |
     |<--------------------+                    |
     |                     |     fork worker    |
     +----------------------------------------->|
     |     worker ready    |                    |
     |<-----------------------------------------+
     |      Egg ready      |                    |
     +-------------------->|                    |
     |      Egg ready      |                    |
     +----------------------------------------->|
```

分析如下

1. `Master` 进程先启动（如通过 `egg-bin` 或 `egg-scripts fork` 出 `Master` 进程）
2. 接着 `Master` 会先启动 `Agent` 进程
3. `Agent` 进程启动成功后，会通过 `IPC` 通信通知 `Master` 自己启动成功了
4. 接着 `Master` 会根据 `CPU` 的核数启动相同数目的 `App Worker` 进程
5. 每个 `App Worker` 进程启动成功后，也会通过 `IPC` 通道通知 `Master` 自己启动成功
6. 一直到所有的 `App Worker` 进程启动成功后，此时整个应用启动成功，`Master` 会通知 `Agent` 和各个 `App Worker` 进程整个应用已经启动成功



## 4.1 Master 的启动

> `Maste`r 的代码主要位于 `egg-cluster/lib/master.js` 文件



**Master 的核心逻辑**

1. `egg-bin` 或 `egg-scripts` 会先 `fork` 出 `Master` 进程

2. `Master` 会初始化进程管理对象 `workerManager`，它是一个 `Manager` 类的实例，维护着进程对象

3. 初始化进程通信对象 `messenger`，它是一个 `Messenger` 类的实例，定义进程通信的规则

4. 可以根据 `startMode` 可选参数选择使用多进程模式启动还是多线程模式启动，接着实例化进程实例

   > 这里只讲多进程，多线程模式是 `hyj1991` 大佬后面才支持的，详情可见此 [PR](https://github.com/eggjs/egg-cluster/pull/101)

5. 用 [get-ready](https://github.com/node-modules/get-ready) 库注册一些启动后要执行的回调方法

6. 监听 `App/Agent` 的启动和退出事件

7. 监听进程的一些信号，如 `SIGTERM` 等

8. 接着先启动 `Agent` 进程，再启动所有 `App` 进程



**源码分析**

先调用启动 `Master` 进程的方法（如 `egg-scripts` 包是用 `child_process` 模块的 `spawn` 启动 `Master` 进程）

```js
// egg-cluster/index.js
// 启动 master 进程入口
exports.startCluster = function(options, callback) {
    new Master(options).ready(callback);
};
```

`Master` 核心源码如下 

```js
// egg-cluster/lib/master.js
// master 继承了 EventEmitter 事件模块
class Master extends EventEmitter {
    constructor(options) {
        // 1. 初始化进程管理对象 workerManager，这个对象主要做的事情如下
        //   -> 管理 worker 和 agent 进程对象（用 Map 存储 worker 进程实例）
        //   -> 提供获取所有 worker的pid 的方法
        //   -> 提供获取worker和agent的数量方法
        //   -> 健康检查 startCheck 方法，正式环境会开启，每 10s 检查一次agent和worker 进程
        //      是否都活着，不是则增加异常计数，异常大于等于3次则触发 exception 事件
        this.workerManager = new Manager();

        // 2. 初始化各进程之间通信的包装类对象
        //   -> 提供统一的方法让 parent、master、agent、app 之间进行通信
        //      parent是父进程，比如使用egg-scripts启动master，则此时egg-scripts进程即parent进程
        this.messenger = new Messenger(this);

        // 3. 判断是启动多进程还是多线程模式，默认启动多进程模式
        // set app & agent worker impl
        if (this.options.startMode === 'worker_threads') {
            this.startByWorkerThreads();
        } else {
            this.startByProcess(); // 实例化多进程对象
        }
      
        // 4. 注册方法，在所有 app 启动完成后会执行(即onAppStart方法里的this.ready(true)触发执行）
        //    这里的ready是 get-ready 库提供
        this.ready(() => {
            const action = 'egg-ready'; 
           // 这里会使用进程通信messenger对象发送egg-ready 事件通知parent、app、agent进程应用启动成功
            
            // 正式环境开启健康检查，定时检查agent和worker的状态
            if (this.isProduction) {
                this.workerManager.startCheck();
            }
        });
    }

    // 5. 监听 app/agent 的启动和退出事件
    this.on('agent-exit', this.onAgentExit.bind(this));
    this.on('agent-start', this.onAgentStart.bind(this));
    this.on('app-exit', this.onAppExit.bind(this));
    this.on('app-start', this.onAppStart.bind(this));
    // reload-worker 事件 在egg-development项目中的agent.js触发
    this.on('reload-worker', this.onReload.bind(this));
    // 监听agent启动事件，然后启动worker进程（在agent进程启动后会发出该事件）
    this.once('agent-start', this.forkAppWorkers.bind(this));

    // 监听各种信号
    // kill(2) Ctrl-C
    process.once('SIGINT', this.onSignal.bind(this, 'SIGINT'));
    // kill(3) Ctrl-\
    process.once('SIGQUIT', this.onSignal.bind(this, 'SIGQUIT'));
    // kill(15) default  ，如kill master进程id会触发
    process.once('SIGTERM', this.onSignal.bind(this, 'SIGTERM'));
    process.once('exit', this.onExit.bind(this));

    // 检测端口有没有被占用，然后启动agent进程 
    this.detectPorts()
        .then(() => {
            // 6. 启动agent
            this.forkAgentWorker();
        });
        
    }
}
```



## 4.2 Agent 的启动

**核心逻辑**

* `Master` 启动 `Agent` 进程

  > 实际是使用了 `child_process` 模块的 `fork()` 来启动 `Agent` 进程，默认传入了 `egg-cluster/lib/agent_worker.js` 文件

* 接着监听进程的 `message`、 `error`、`exit` 等事件

* `Agent` 进程默认会实例化 `Agent` 对象，这个是 `egg` 包中暴露的对象，实例化时执行加载器的一些逻辑

  > `Agent` 对象是 `egg` 包默认的 `Agent` 对象，即  `egg/lib/agent.js` 的 `Agent` 类，这里的 `Agent` 类我们也可以自己实现，只要实现最基础的 `EggCore` 类即可

* `Agent` 进程启动成功会发送 `agent-start` 事件给 `Master` 进程

* 最后通过 `gracefulExit` 添加进程的优雅退出



**源码分析**

`Master` 进程准备完毕后开始启动 `Agent` 进程，源码如下

```js
// egg-cluster/lib/master.js
this.detectPorts()
    .then(() => {
        //  启动 agent
        this.forkAgentWorker();
    });

forkAgentWorker() {
    this.agentWorker.on('agent_forked', agent => this.workerManager.setAgent(agent));
    this.agentWorker.fork();
}


// egg-cluster/lib/utils/mode/impl/process/agent.js
fork() {
    ......
    
    // 使用child_process.fork启动，跟app使用cfork包有区别
    // egg-cluster/lib/agent_worker.js为启动文件
    const worker = this.#worker = childprocess.fork(this.getAgentWorkerFile(), args, opt);
    const agentWorker = this.instance = new AgentWorker(worker);
    this.emit('agent_forked', agentWorker);
    agentWorker.status = 'starting';
    agentWorker.id = ++this.#id;
  
   // agent进程监听发给它的消息
   // 如 agent 进程调用 sendToApp 跟 app 进程通信时，会经过这里转发给 app 进程
   // forwarding agent' message to messenger
    worker.on('message', msg => {
      if (typeof msg === 'string') {
        msg = {
          action: msg,
          data: msg,
        };
      }
      msg.from = 'agent';
      this.messenger.send(msg);
    });
    // 监听error事件
    worker.on('error', err => {
      err.name = 'AgentWorkerError';
      err.id = worker.id;
      err.pid = agentWorker.workerId;
      this.logger.error(err);
    });
      
    // 监听agent退出事件，发消息通知master  
    // agent exit message
    worker.once('exit', (code, signal) => {
      this.messenger.send({
        action: 'agent-exit',
        data: {
          code,
          signal,
        },
        to: 'master',
        from: 'agent',
      });
    });

    return this;
  }


// agent进程的启动文件
getAgentWorkerFile() {
    return path.join(__dirname, '../../../agent_worker.js');
}
```

`Agent` 进程启动后会触发 `agent-start` 事件，源码如下

```js
// 位于 `egg-cluster/lib/agent_worker.js` 中
// 实例化agent 
// framework为egg包，即为egg中导出的Agent对象，exports.Agent = require('./lib/agent');
const Agent = require(options.framework).Agent;

let agent;
try {
  // 实例化 Agent
  agent = new Agent(options);
} catch (err) {
  consoleLogger.error(err);
  throw err;
}

// 注册方法，在agent启动后该注册的方法会被调用
agent.ready(err => {
  // don't send started message to master when start error
  if (err) return;

  agent.removeListener('error', startErrorHandler);
  // 向 agent 进程发送 agent-start 事件，agent接收到消息后会转发给master
  AgentWorker.send({ action: 'agent-start', to: 'master' });
});

// 监听error，主动退出
agent.once('error', startErrorHandler);

// 优雅退出
gracefulExit({
    logger: consoleLogger,
    label: 'agent_worker',
    beforeExit: () => agent.close(),
});
```

`Master` 接收到 `Agent` 启动成功的 `agent-start` 事件，接着会开始启动 `App Worker` 进程，源码如下

```js
// egg-cluster/lib/master.js
this.on('agent-start', this.onAgentStart.bind(this));

// 启动过app 进程
this.once('agent-start', this.forkAppWorkers.bind(this));

onAgentStart() {
    this.agentWorker.status = 'started';
    
   // 标识所有app是否启动，当agent和app都启动后，如果agent挂掉了重新fork了一个，此时isAllAppWorkerStarted为true，执行该逻辑
    // Send egg-ready when agent is started after launched
    if (this.isAllAppWorkerStarted) {
        this.messenger.send({
            action: 'egg-ready',
            to: 'agent',
            data: this.options,
        });
    }

    // 把启动的agent的pid通知app
    this.messenger.send({
        action: 'egg-pids',
        to: 'app',
        data: [this.agentWorker.pid],
    });
    
    // should send current worker pids when agent restart
    if (this.isStarted) {
        this.messenger.send({
            action: 'egg-pids',
            to: 'agent',
            data: this.workerManager.getListeningWorkerIds(),
        });
    }

    // 通知app，agent启动完成
    this.messenger.send({
        action: 'agent-start',
        to: 'app',
    });
}
```



## 4.3 App Worker 的启动

**核心逻辑**

* `Master` 监听到 `agent` 进程启动成功的事件 `agent-start` 后，会使用 `cfork` 包来启动 `App Worker`

  >  默认传入了 `egg-cluster/lib/app_worker.js` 文件来启动 `app`，`cfork` 包装了 `Node.js` 的 `cluster` 模块，使得启动进程更容易，还提供了进程重启次数的限制功能
  >
  
* `App` 进程会先实例化 `Application` 对象，实例化时执行加载器的一些逻辑

  > `Application` 是 `egg` 包的 `Application` 对象，即  `egg/lib/application.js` 的 `Application` 类，这里的 `Application` 类我们也可以自己实现，只要实现最基础的 `EggCore `类即可

* 接着启动 `http` 服务

* 最后通过 `gracefulExit` 添加进程的优雅退出



**源码分析**

`Master`  在 `Agent` 启动成功后后启动 `App Worker`，源码如下

```js
// egg-cluster/lib/master.js
// 监听到agent启动事件后，启动app 
this.once('agent-start', this.forkAppWorkers.bind(this));

forkAppWorkers() {
    this.appWorker.on('worker_forked', worker => this.workerManager.setWorker(worker));
    this.appWorker.fork();
}

// egg-cluster/lib/utils/mode/impl/process/app.js
class AppUtils extends BaseAppUtils {
  fork() {
    //////
    // 使用cfork包启动，底层是cluster包，跟agent进程使用child_process.fork启动有点区别
    cfork({
      exec: this.getAppWorkerFile(), // egg-cluster/lib/app_worker.js
      args,
      silent: false,
      count: this.options.workers,
      // don't refork in local env
      refork: this.isProduction,
      windowsHide: process.platform === 'win32',
    });

    // 当新的 app 进程被fork时触发
    cluster.on('fork', worker => {
      const appWorker = new AppWorker(worker);
      this.emit('worker_forked', appWorker);
      appWorker.disableRefork = true;
      
       // 监听发给app的消息
      worker.on('message', msg => {
        if (typeof msg === 'string') {
          msg = {
            action: msg,
            data: msg,
          };
        }
        msg.from = 'app';
        this.messenger.send(msg);
      });
      
      .....
    });
    
    // disconnect 进程失联事件，此处只是打印日志，不做具体处理，实际处理在cfork包中
    cluster.on('disconnect', worker => {
      const appWorker = new AppWorker(worker);
      this.logger.info('[master] app_worker#%s:%s disconnect, suicide: %s, state: %s, current workers: %j',
        appWorker.id, appWorker.workerId, appWorker.exitedAfterDisconnect, appWorker.state, Object.keys(cluster.workers));
    });
    
    // 监听app进程退出事件，然后通知master，例如当app worker出现OOM被系统杀死时触发
    cluster.on('exit', (worker, code, signal) => {
      const appWorker = new AppWorker(worker);
        //  app worker退出后发送 app-exit 通知master 
      this.messenger.send({
        action: 'app-exit',
        data: {
          workerId: appWorker.workerId,
          code,
          signal,
        },
        to: 'master',
        from: 'app',
      });
    });
    
   // 监听listening，app worker调用listen()后触发，通知master
    cluster.on('listening', (worker, address) => {
      const appWorker = new AppWorker(worker);
      
      // app worker启动后通知master
      //   -> this.on('app-start', this.onAppStart.bind(this));
      //     -> 发送egg-pids给agent进程
      //     -> 当所有app woker启动后，发送egg-ready给app
      //     -> this.ready(true)触发执行master实例化时ready注册的方法 
      this.messenger.send({
        action: 'app-start',
        data: {
          workerId: appWorker.workerId,
          address,
        },
        to: 'master',
        from: 'app',
      });
    });

    return this;
  }
}

// worker进程的启动文件
getAppWorkerFile() {
    return path.join(__dirname, 'app_worker.js');
}
```

`App` 启动成功后会向 `Master` 发送 `app-start` 事件，源码如下

```js
// egg-cluster/lib/app_worker.js
// 实例化app
// 默认framework为egg包，即为egg中导出的Application对象， egg/lib/application.js
const Application = require(options.framework).Application;
const app = new Application(options); 

// 注册方法，app启动后执行
app.ready(startServer);

// app启动超时事件
app.once('startTimeout', startTimeoutHandler);

// 测试环境worker默认是1，正式环境会取process.env.EGG_WORKERS，默认取os.cpus().length
// 每个worker启动app服务(EggApplication的实例)用于处理请求
server = require('http').createServer(app.callback());

// 优雅退出
gracefulExit({
    logger: consoleLogger,
    label: 'app_worker',
    beforeExit: () => app.close(),
});

// egg-cluster/lib/utils/mode/impl/process/app.js
cluster.on('listening', (worker, address) => {
     // 这里app启动后就会发送app-start消息给master
      const appWorker = new AppWorker(worker);
      this.messenger.send({
        action: 'app-start',
        data: {
          workerId: appWorker.workerId,
          address,
        },
        to: 'master',
        from: 'app',
      });
    });
```

`Master` 接收到 `App` 启动成功的 `app-start` 事件，源码如下

```js
// egg-cluster/lib/master.js
// 监听woker启动成功后做处理
this.on('app-start', this.onAppStart.bind(this));
  
onAppStart(data) {
    if (this.isAllAppWorkerStarted) {
        // 向所有worker发送egg-ready事件
        this.messenger.send({
            action: 'egg-ready',
            to: 'app',
            data: this.options,
        });
    }
    
    if (this.options.sticky) {
        this.startMasterSocketServer(err => {
            if (err) return this.ready(err);
            this.ready(true);
        });
    } else {
        // 触发master上注册的回调函数
        this.ready(true);
    }
}
```



##  4.4 Agent 和 Worker 启动的区别

1. 启动 `Agent` 使用的是 `child_process` 模块的 `fork` 方法，因为它不需要对外提供 `http` 服务，所以用 `child_process` 即可
2. 启动各个 `App Worker` 使用的是 `cfork` 包，它里面使用的是 `cluster` 模块的 `fork` 方法，因为 `App Worker` 是对外提供 `http` 服务的，显然用 `cluster` 模块会更合适，且 `cluster` 模块可以允许对同一端口进行监听而不会产生端口冲突，`cluster` 还默认使用 `round-robin` 轮询策略进行负载均衡，它会把收到的 `http` 请求合理地分配给各个 `Worker` 进行处理



## 4.5 cfork 库分析

`cfork` 库的作用

1. 更方便的使用 `cluster` 去启动一个进程
2. 允许自动重启进程，且有重启频率的控制
3. 自动监听 `uncaughtException` 未捕获异常并记录日志



**源码分析**

监听 `disconnect` 事件：如果 `ipc` 通道关闭，会触发该事件，这里会执行重启进程的逻辑，源码如下

```js
cluster.on('disconnect', function (worker) {
    disconnectCount++;
    var isDead = worker.isDead && worker.isDead();
    ...
    // 如果进程已经死了，则只记录日志即可
    if (isDead) {
      // worker has terminated before disconnect
      log('[%s] [cfork:master:%s] don\'t fork, because worker:%s exit event emit before disconnect',
        utility.logDate(), process.pid, worker.process.pid);
      return;
    }
    
    // 设置是否不需要重启标识，默认会重启的，如果不需要重启得手动设置为false
    if (worker.disableRefork) {
      // worker has terminated by master, like egg-cluster master will set disableRefork to true
      log('[%s] [cfork:master:%s] don\'t fork, because worker:%s will be kill soon',
        utility.logDate(), process.pid, worker.process.pid);
      return;
    }

 
    disconnects[worker.process.pid] = utility.logDate();
  // 判断是否满足重启条件
    if (allow()) {
      newWorker = forkWorker(worker._clusterSettings, worker._clusterWorkerEnv);
      newWorker._clusterSettings = worker._clusterSettings;
      newWorker._clusterWorkerEnv = worker._clusterWorkerEnv;
      log('[%s] [cfork:master:%s] new worker:%s fork (state: %s)',
        utility.logDate(), process.pid, newWorker.process.pid, newWorker.state);
    } else {
      log('[%s] [cfork:master:%s] don\'t fork new work (refork: %s)',
        utility.logDate(), process.pid, refork);
    }
  });

```

监听 `exit` 事件：如果进程退出，会触发该事件，里面也会执行重启进程的逻辑，源码如下

```js
cluster.on('exit', function (worker, code, signal) {
    var log = console[worker.disableRefork ? 'info' : 'error'];
    var isExpected = !!disconnects[worker.process.pid];
    var isDead = worker.isDead && worker.isDead();
    // 进程已经失联死了，就不重启了
    if (isExpected) {
      delete disconnects[worker.process.pid];
      // worker disconnect first, exit expected
      return;
    }
  
    if (worker.disableRefork) {
      // worker is killed by master
      return;
    }

    unexpectedCount++;
    // 判断是否满足重启条件
    if (allow()) {
      newWorker = forkWorker(worker._clusterSettings, worker._clusterWorkerEnv);
      newWorker._clusterSettings = worker._clusterSettings;
      newWorker._clusterWorkerEnv = worker._clusterWorkerEnv;
      log('[%s] [cfork:master:%s] new worker:%s fork (state: %s)',
        utility.logDate(), process.pid, newWorker.process.pid, newWorker.state);
    } else {
      log('[%s] [cfork:master:%s] don\'t fork new work (refork: %s)',
        utility.logDate(), process.pid, refork);
    }
    cluster.emit('unexpectedExit', worker, code, signal);
  });

```

`allow()` 函数用于判断是否满足进程重启的条件

* 默认重启次数小于 60 或 最后一次的重启跟最初重启的事件间隔大于1分钟，就允许重启
* 如果不允许重启，会触发 `reachReforkLimit` 事件

```js
/**
   * allow refork
   */
  function allow() {
    if (!refork) {
      return false;
    }

    var times = reforks.push(Date.now());

    if (times > limit) {
      reforks.shift();
    }

    var span = reforks[reforks.length - 1] - reforks[0];
    var canFork = reforks.length < limit || span > duration;

    if (!canFork) {
      cluster.emit('reachReforkLimit');
    }

    return canFork;
  }
```



# 5 进程的退出流程

## 5.1 App Worker 的退出

假如 `App` 出现 `OOM`、系统异常时，当前 `app` 进程会直接退出

> 如果是出现`未捕获异常`还有机会让进程继续执行，下面优雅退出章节会讲解此退出方式的处理



**核心逻辑**

* 我们知道 `App Worker` 退出后，`App Worker` 会自动重启。`App Worker` 是通过 `cfork` 包（本质是 `cluster` 模块）启动的进程，里面会监听 `app worker` 进程 `exit` 退出事件和 `disconnect` 事件，然后判断是否需要重启

* `App worker` 退出后，会发送 `app-exit` 事件通知 `Master` 进程

* `Master` 接收到 `app-exit` 事件后会做一些清理工作，比如移除 `worker` 的监听器，从 `manager` 进程管理对象中删除进程对象等

* 如果在所有 `App worker` 全部启动完成的情况下，有一个 `App Worker` 挂了，此时会发送 `app-worker-died` 事件通知父进程有 `App Worker` 进程挂了

  > 如果正式环境是用 `egg-scripts` 启动的，那此时的父进程就是 `egg-scripts` 进程，由代码可知 `egg-scripts`  并没有处理该事件，只是 `egg` 内部暴露了这么一个事件而已

* 如果是在 `App Worker` 还没全部启动完成的情况下，有一个 `App Worker` 挂了，说明 `App Worker` 有可能是在服务启动过程中挂掉的，那 `Master` 进程直接退出



**源码分析**

`cfork` 里监听 `worker` 进程 `exit` 退出事件，然后向 `Master` 进程发送 `app-exit` 事件，源码如下

```js
// egg-cluster/lib/utils/mode/impl/process/app.js
// worker进程退出事件
cluster.on('exit', (worker, code, signal) => {
      const appWorker = new AppWorker(worker);
      // 向 master 发送app-exit 事件
      this.messenger.send({
        action: 'app-exit',
        data: {
          workerId: appWorker.workerId,
          code,
          signal,
        },
        to: 'master',
        from: 'app',
      });
    });
```

`Master` 接收到 `app-exit` 事件后，执行 `onAppExit` 退出函数，这里主要是执行一些清理工作，源码如下

```js
// egg-cluster/lib/master.js
this.on('app-exit', this.onAppExit.bind(this));

/**
   * App Worker exit handler
   * @param {Object} data
   *  - {String} workerId - worker id
   *  - {Number} code - exit code
   *  - {String} signal - received signal
   */
  onAppExit(data) {
    if (this.closed) return;

    const worker = this.workerManager.getWorker(data.workerId);

    // 开发模式下
    if (!worker.isDevReload) {
      const signal = data.signal;
      const message = util.format(
        '[master] app_worker#%s:%s died (code: %s, signal: %s, suicide: %s, state: %s), current workers: %j',
        worker.id, worker.workerId, worker.exitCode, signal,
        worker.exitedAfterDisconnect, worker.state,
        this.workerManager.getWorkers()
      );
      if (this.options.isDebug && signal === 'SIGKILL') {
        // exit if died during debug
        this.logger.error(message);
        this.logger.error('[master] worker kill by debugger, exiting...');
        setTimeout(() => this.close(), 10);
      } else {
        const err = new Error(message);
        err.name = 'AppWorkerDiedError';
        this.logger.error(err);
      }
    }

    // 删除监听器和 master维护的进程对象
    // remove all listeners to avoid memory leak
    worker.clean();
    this.workerManager.deleteWorker(data.workerId);
    
    // 通知 agent，app worker 死了
    // send message to agent with alive workers
    this.messenger.send({
      action: 'egg-pids',
      to: 'agent',
      data: this.workerManager.getListeningWorkerIds(),
    });

    // 所有 app worker 都启动的情况下，发送 app-worker-died 通知 parent 进程，app 退出了，如果正式环境是用 egg-scripts 启动的，那父进程就是 egg-scripts 进程，此时父进程并没有处理该事件消息，只是暴露出这么一个事件而已
    if (this.appWorker.isAllWorkerStarted) {
      // cfork will only refork at production mode
      this.messenger.send({
        action: 'app-worker-died',
        to: 'parent',
      });

    } else {
      // exit if died during startup
      this.logger.error('[master] app_worker#%s:%s start fail, exiting with code:1',
        worker.id, worker.workerId);
      process.exit(1);
    }
  }
```

上面的 `onAppExit` 函数里并没有处理 `App Worker` 重启，实际重启是在 `cfork` 库里重启的，`cfork` 包也会监听进程的 `exit` 事件，里面会判断是否需要重启，源码如下

```js
// cfork/index.js
function fork(options) {
  // 监听进程的退出事件
  cluster.on('exit', function (worker, code, signal) {
    var isExpected = !!disconnects[worker.process.pid];
    ......
    // 如果 IPC 通道先关闭了触发了disconnect，则不处理，因为cfork监听disconnect事件也会执行重启的逻辑
    if (isExpected) {
      delete disconnects[worker.process.pid];
      // worker disconnect first, exit expected
      return;
    }
    
    // 根据disableRefork变量判断，如果不需要重启则不重启
    // 当egg-cluster中主动调用杀死进程时，这个变量为true，表示不需要重启
    if (worker.disableRefork) {
      // worker is killed by master
      return;
    }

    unexpectedCount++;
    // 判断是否需要重启，默认重启次数小于60或最后一次的重启跟最先一个重启的事件间隔大于1分钟，就允许重启
    if (allow()) {
      // 这里 egg 会重启 app worker 进程
      newWorker = forkWorker(worker._clusterSettings, worker._clusterWorkerEnv);
      newWorker._clusterSettings = worker._clusterSettings;
      newWorker._clusterWorkerEnv = worker._clusterWorkerEnv;
      log('[%s] [cfork:master:%s] new worker:%s fork (state: %s)',
        utility.logDate(), process.pid, newWorker.process.pid, newWorker.state);
    } else {
      log('[%s] [cfork:master:%s] don\'t fork new work (refork: %s)',
        utility.logDate(), process.pid, refork);
    }
    cluster.emit('unexpectedExit', worker, code, signal);
  });
}
```



## 5.2 Agent 的退出

**核心逻辑**

* `Agent` 进程监听到进程的 `exit` 退出事件后，会发送 `agent-exit` 事件通知 `Master`
* `Master` 接收到 `agent-exit` 事件后会先向 `App Worker `发送 `egg-pids` 事件说明自己挂了
* 接着删除 `Master `中 `manager` 进程管理对象维护的 `agent` 进程对象
* 删除 `Agent` 进程的监听器
* 如果是在任何一个 `App Worker` 启动完成之后，`Agent` 才挂了，则在 1秒后会执行 `forkAgentWorker` 重启 `Agent` 进程
* 如果还没有任何一个 `App Worker` 启动完成，此时 `Agent` 挂了，则整个 `Master` 直接退出



**源码分析**

`Agent` 进程会监听 `exit` 退出事件，然后往 `Master` 进程发送 `agent-exit` 事件，源码如下

```js
// egg-cluster/lib/utils/mode/impl/process/agent.js
// child_process模块的exit事件监听agent进程退出后，发送 agent-exit 事件通知 master
worker.once('exit', (code, signal) => {
      this.messenger.send({
        action: 'agent-exit',
        data: {
          code,
          signal,
        },
        to: 'master',
        from: 'agent',
      });
    });
```

`Master` 处理 `Agent` 退出的逻辑，由 `onAgentExit` 函数实现，源码如下

```js
// egg-cluster/lib/master.js
this.on('agent-exit', this.onAgentExit.bind(this));

/**
   * Agent Worker exit handler
   * Will exit during startup, and refork during running.
   * @param {Object} data
   *  - {Number} code - exit code
   *  - {String} signal - received signal
   */
  onAgentExit(data) {
    if (this.closed) return;

    // 发送消息通知app worker
    this.messenger.send({
      action: 'egg-pids',
      to: 'app',
      data: [],
    });
    
    // 删除master维护的agent进程
    const agentWorker = this.agentWorker;
    this.workerManager.deleteAgent(agentWorker);

    const err = new Error(util.format('[master] agent_worker#%s:%s died (code: %s, signal: %s)',
      agentWorker.instance.id, agentWorker.instance.workerId, data.code, data.signal));
    err.name = 'AgentWorkerDiedError';
    this.logger.error(err);

    // 删除agent的监听器
    // remove all listeners to avoid memory leak
    agentWorker.clean();

    // 如果存在app启动成功，则1秒后，会自动重启agent进程，否则退出master进程
    if (this.isStarted) {
      this.log('[master] try to start a new agent_worker after 1s ...');
      setTimeout(() => {
        this.logger.info('[master] new agent_worker starting...');
        this.forkAgentWorker();
      }, 1000);
      this.messenger.send({
        action: 'agent-worker-died',
        to: 'parent',
      });
    } else {
      this.logger.error('[master] agent_worker#%s:%s start fail, exiting with code:1',
        agentWorker.instance.id, agentWorker.instance.workerId);
      process.exit(1);
    }
  }
```



## 5.3 Master 的退出

`Master` 会监听进程的 `exit` 事件，如果 `Master` 退出后，会执行 `onExit` 函数，处理逻辑如下

* 如果启动过时有传入 `Master` 的进程 `id`，则先删除进程 `id`
* 最后记录个退出日志而已



`Master` 如果接收到 `kill` 命令，此时会触发 `SIGTERM`、`SIGINT` 等信号，`Master` 监听到后会执行 `onSignal` 函数，处理逻辑如下

* 先获取一下堆数据，打印下日志

* 接着调用 `close()` 函数执行一些退出前置逻辑

* `close() ` 中会先杀死所有 `App worker`，默认 5 秒后执行，因为这里是主动退出，所以 `App Worker` 设置了不用重启，即 `worker.disableRefork = true; `

  > 可能第一次看源码时会不了这个参数的作用，且不知道是在哪里起的作用，我顺便向 `cfork` 提了一个 [PR](https://github.com/node-modules/cfork/pull/118)，提供一个设置 `worker.disableRefork` 的函数，这样从调用处就可以很方面的看出来这个 `disableRefork` 参数的作用 

* `App Worker` 杀死后，再杀死 `Agent` 进程，也是默认 5 秒后执行

* 最后 `Master` 进程自己再退出



**源码分析**

`onExit` 退出函数的逻辑如下

```js
// egg-cluster/lib/master.js
process.once('exit', this.onExit.bind(this));

onExit(code) {
    // 判断是否需要删除进程
    if (this.options.pidFile && fs.existsSync(this.options.pidFile)) {
      try {
        fs.unlinkSync(this.options.pidFile);
      } catch (err) {
        /* istanbul ignore next */
        this.logger.error('[master] delete pidfile %s fail with %s', this.options.pidFile, err.message);
      }
    }
   
    // 记录日志
    // istanbul can't cover here
    // https://github.com/gotwarlost/istanbul/issues/567
    const level = code === 0 ? 'info' : 'error';
    this.logger[level]('[master] exit with code:%s', code);
  }
```

`onSignal` 函数接收信号的逻辑如下

```js
// egg-cluster/lib/master.js

// https://nodejs.org/api/process.html#process_signal_events
// https://en.wikipedia.org/wiki/Unix_signal

// kill -2 master进程时，会触发SIGINT信号
// kill(2) Ctrl-C
process.once('SIGINT', this.onSignal.bind(this, 'SIGINT'));

// kill -3 master进程时，会触发SIGQUIT信号
// kill(3) Ctrl-\
process.once('SIGQUIT', this.onSignal.bind(this, 'SIGQUIT'));

// kill -15 master进程时，会触发SIGTERM信号
// kill(15) default
process.once('SIGTERM', this.onSignal.bind(this, 'SIGTERM'));

onSignal(signal) {
    if (this.closed) return;

    this.logger.info('[master] master is killed by signal %s, closing', signal);

    // 日志记录堆内存数据
    // logger more info
    const { used_heap_size, heap_size_limit } = v8.getHeapStatistics();
    this.logger.info('[master] system memory: total %s, free %s', os.totalmem(), os.freemem());
    this.logger.info('[master] process info: heap_limit %s, heap_used %s', heap_size_limit, used_heap_size);

    // 调用close函数
    this.close();
  }

// 执行close函数
 async close() {
    this.closed = true;
    try {
      await this._doClose();
      this.log('[master] close done, exiting with code:0');
      
      // app worker 和 agent 杀死后，master 自己退出
      process.exit(0);
    } catch (e) /* istanbul ignore next */ {
      this.logger.error('[master] close with error: ', e);
      process.exit(1);
    }
  }

 async _doClose() {
    // kill app workers
    // kill agent worker
    // exit itself
    const legacyTimeout = process.env.EGG_MASTER_CLOSE_TIMEOUT || 5000;
    const appTimeout = process.env.EGG_APP_CLOSE_TIMEOUT || legacyTimeout;
    const agentTimeout = process.env.EGG_AGENT_CLOSE_TIMEOUT || legacyTimeout;
    this.logger.info('[master] send kill SIGTERM to app workers, will exit with code:0 after %sms', appTimeout);
    this.logger.info('[master] wait %sms', appTimeout);
    try {
      // 先杀死所有app worker，默认5秒后执行
      await this.killAppWorkers(appTimeout);
    } catch (e) /* istanbul ignore next */ {
      this.logger.error('[master] app workers exit error: ', e);
    }
    this.logger.info('[master] send kill SIGTERM to agent worker, will exit with code:0 after %sms', agentTimeout);
    this.logger.info('[master] wait %sms', agentTimeout);
    try {
      // 再杀死agent，也是默认5秒后执行
      await this.killAgentWorker(agentTimeout);
    } catch (e) /* istanbul ignore next */ {
      this.logger.error('[master] agent worker exit error: ', e);
    }
  }

async killAppWorkers(timeout) {
    await this.appWorker.kill(timeout);
}

async kill(timeout) {
    await Promise.all(Object.keys(cluster.workers).map(id => {
      const worker = cluster.workers[id];
      // 对app worker 设置disableRefork变量，实际是在 cfork 包中起作用
      worker.disableRefork = true;
      return terminate(worker, timeout);
    }));
  }
```



# 6 优雅退出和平滑重启

优雅退出即在程序退出前执行一些逻辑，如释放资源之类的操作；`Egg` 的优雅退出是由 `graceful-process` 实现



## 6.1 graceful-process 库分析

优雅退出实际是由 `graceful-process` 包实现的，主要逻辑如下

* 提供 `getExitFunction` 包装传入的 `beforeExit` 函数，里面会先执行 `beforeExit` 函数，不管成功还是失败，最后都会调用 `process.exit(code)` 退出当前进程，如果进程有监听 `exit` 事件，就会被接收到

  > "当前进程"是指，如果是 `egg` 的 `Agent` 进程调用就是 `Agent`进程，如果是 `App` 进程调用就是 `App` 进程

* 监听 `SIGTERM` 信号，这里会调用 `exit(0)` ，`exit` 其实就是 `getExitFunction` 函数

* 监听 `exit` 事件，这里只是记录一下日志

* 如果是 `cluster` 启动的进程，比如 `egg` 的 `App` 进程，就会监听 `disconnect` 事件，如果进程是通过调用 `disconnect()` 方法退出的（`exitedAfterDisconnect` 为 `true`），则不打印日志，否则打印下日志

* 如果不是 `cluster` 启动的进程，比如 `egg` 的 `Agent` 进程（使用 `child_process `启动的），也是监听`disconnect` 事件，然后事件循环的 `setImmediate` 阶段执行退出当前进程



**源码分析**

```js
// graceful-process/index.js
module.exports = (options = {}) => {
    ......
    
    // 实际执行退出的处理逻辑，在退出前执行beforeExit方法，进行释放资源之类的操作
    const exit = getExitFunction(options.beforeExit, logger, label);

    // 当前进程(如app/agent)监听SIGTERM信号(kill 进程id)，调用exit(0)优雅退出
    process.once('SIGTERM', () => {
        printLogLevels.info && logger.info('[%s] receive signal SIGTERM, exiting with code:0', label);
        // 实际退出的方法
        exit(0);
    });
  
    // 监听exit，记录日志
    process.once('exit', code => {
        const level = code === 0 ? 'info' : 'error';
        printLogLevels[level] && logger[level]('[%s] exit with code:%s', label, code);
    });


    if (cluster.worker) {
        // cluster mode 
        // 即app进程监听
        // https://github.com/nodejs/node/blob/6caf1b093ab0176b8ded68a53ab1ab72259bb1e0/lib/internal/cluster/child.js#L28
        cluster.worker.once('disconnect', () => {
            // ignore suicide disconnect event
            if (cluster.worker.exitedAfterDisconnect) return;
            // cluster包内部有监听disconnect，所以此处没做优雅退出处理
            logger.error('[%s] receive disconnect event in cluster fork mode, exitedAfterDisconnect:false', label);
        });
    } else {
        // child_process mode 
        // agent进程监听，当master退出了，ipc通道关闭了，执行该监听，执行优雅退出
        process.once('disconnect', () => {
            // wait a loop for SIGTERM event happen
            setImmediate(() => {
                // if disconnect event emit, maybe master exit in accident
                logger.error('[%s] receive disconnect event on child_process fork mode, exiting with code:110', label);
                exit(110);
            });
        });
    }
}


function getExitFunction(beforeExit, logger, label) {
  if (beforeExit) assert(is.function(beforeExit), 'beforeExit only support function');

  return once(code => {
    if (!beforeExit) process.exit(code);
    // 先执行 beforeExit 函数
    Promise.resolve()
      .then(() => {
        return beforeExit();
      })
      .then(() => {
        logger.info('[%s] beforeExit success', label);
        process.exit(code);
      })
      .catch(err => {
        logger.error('[%s] beforeExit fail, error: %s', label, err.message);
        process.exit(code);
      });
  });

}
```



## 6.2 Agent 的优雅退出

`Agent` 进程调用了 `gracefulExit` 方法，在 `beforeExit` 属性传入 `() => agent.close()` 函数，源码如下

```js
// egg-cluster/lib/agent_worker.js 
AgentWorker.gracefulExit({
  logger: consoleLogger,
  label: 'agent_worker',
  beforeExit: () => agent.close(), // 退出前执行的方法，即生命周期函数beforeClose
});

// egg-cluster/lib/utils/mode/impl/process/agent.js
const gracefulExit = require('graceful-process');
static gracefulExit(options) {
    gracefulExit(options);
}
```



**`close()` 函数是在 `egg` 包中实现，其主要逻辑如下**

* 先删除 `uncaughtException` 监听

* 清除 `Agent` 存活定时器，因为 `Agent` 是一个普通进程，不像 `App Worker` 这种 `http` 进程是一个常驻进程，所以加了这个定时器是为了 `Agent` 的保活，这个定时器让 `Node.js` 有活跃事件使得事件循环会一直进行着

  > 关于 `Node.js` 的事件循环可以参考这篇[文章](https://zhuanlan.zhihu.com/p/622381734)

* 最后调用 `close()` 方法，这里 `Agent` 继承了 `EggApplication`，`EggApplication` 继承了 `EggCore`，`close() ` 的实现是在 `EggCore` 中

* `close()` 方法中会先调用通过生命周期函数 `beforeClose` 注册的方法，先注册的后执行，然后触发 `close` 事件，最后删除监听器，标识关闭



**源码如下**

```js
// egg/lib/agent.js
class Agent extends EggApplication {
    constructor(options = {}) {
          this.agentAliveHandler = setInterval(() => {}, 24 * 60 * 60 * 1000);
    }
  
    close() {
        process.removeListener('uncaughtException', this._uncaughtExceptionHandler);
        clearInterval(this.agentAliveHandler);
        return super.close();
    }
}

// egg/lib/egg.js
class EggApplication extends EggCore {
}
// egg-core/lib/egg.js
class EggCore extends KoaApplication {
   /**
   * Close all, it will close
   * - callbacks registered by beforeClose
   * - emit `close` event
   * - remove add listeners
   *
   * If error is thrown when it's closing, the promise will reject.
   * It will also reject after following call.
   * @return {Promise} promise
   * @since 1.0.0
   */
  async close() {
    if (this[CLOSE_PROMISE]) return this[CLOSE_PROMISE];
    this[CLOSE_PROMISE] = this.lifecycle.close();
    return this[CLOSE_PROMISE];
  }
}

// egg-core/lib/lifecycle.js
// 调用通过生命周期函数 beforeClose 注册的方法，先注册的后执行，closeFns里放着注册的beforeClose函数
// 触发close事件
// 删除监听器
// 标识为关闭
async close() {
    // close in reverse order: first created, last closed
    const closeFns = Array.from(this[CLOSE_SET]);
    for (const fn of closeFns.reverse()) {
      await utils.callFn(fn);
      this[CLOSE_SET].delete(fn);
    }
    // Be called after other close callbacks
    this.app.emit('close');
    this.removeAllListeners();
    this.app.removeAllListeners();
    this[IS_CLOSED] = true;
 }
```



## 6.3 App Worker 的优雅退出

`App Worker` 和 `Agent` 进程一样，也是调用了 `gracefulExit` 方法，`beforeExit` 属性传入 `() => app.close()` 函数，`close()` 函数因为 `App` 进程没有覆写，所以和 `Agent` 进程一样，也是继承自 `EggCore`，`close()` 的实现是也在 `EggCore` 中，具体解析课参考上面 `Agent` 的讲解

```js
// egg-cluster/lib/app_worker.js
gracefulExit({
    logger: consoleLogger,
    label: 'app_worker',
    beforeExit: () => app.close(), // 即生命周期函数beforeClose
});

// egg-cluster/lib/utils/mode/impl/process/app.js
const gracefulExit = require('graceful-process');
static gracefulExit(options) {
    gracefulExit(options);
}
```



## 6.4 App 服务的平滑重启(uncaughtException)

平滑重启是指在程序运行过程中，通过优雅地关闭和重新启动应用程序的方式进行更新或部署，以确保对正在进行的任务和连接的最小中断。



上面已经也讲过了 `App` 的优雅退出，其实 `App` 的 `http` 服务退出处理还没讲，下面来讲下 `http` 服务是怎么优雅关闭的。



`App` 在启动 `http` 服务后，会触发 `server` 事件，然后注册 `graceful()` 优雅退出和重启的代码，源码如下

```js
//egg/lib/application.js
class Application extends EggApplication {
  onServer(server) {
    // expose app.server
    this.server = server;
    // set ignore code
    const serverGracefulIgnoreCode = this.config.serverGracefulIgnoreCode || [];

    /* istanbul ignore next */
    graceful({
      server: [ server ],
      error: (err, throwErrorCount) => {
        const originMessage = err.message;
        if (originMessage) {
          // shouldjs will override error property but only getter
          // https://github.com/shouldjs/should.js/blob/889e22ebf19a06bc2747d24cf34b25cc00b37464/lib/assertion-error.js#L26
          Object.defineProperty(err, 'message', {
            get() {
              return originMessage + ' (uncaughtException throw ' + throwErrorCount + ' times on pid:' + process.pid + ')';
            },
            configurable: true,
            enumerable: false,
          });
        }
        this.coreLogger.error(err);
      },
      ignoreCode: serverGracefulIgnoreCode,
    });

    server.on('clientError', (err, socket) => this.onClientError(err, socket));

    // server timeout
    if (is.number(this.config.serverTimeout)) server.setTimeout(this.config.serverTimeout);
  }
  
  [BIND_EVENTS]() {
    ...
    // expose server to support websocket
    this.once('server', server => this.onServer(server));
  }

}
```



接着以 `App` 出现 `uncaughtException` 未捕获异常怎么平滑重启的，`App` 未捕获异常异常退出的整体流程如下

```
+---------+                 +---------+
|  Worker |                 |  Master |
+---------+                 +----+----+
     | uncaughtException         |
     +------------+              |
     |            |              |                   +---------+
     | <----------+              |                   |  Worker |
     |                           |                   +----+----+
     |        disconnect         |   fork a new worker    |
     +-------------------------> + ---------------------> |
     |         wait...           |                        |
     |          exit             |                        |
     +-------------------------> |                        |
     |                           |                        |
    die                          |                        |
                                 |                        |
                                 |                        |
```

**分析如下**

1. `graceful` 包中提供了未捕获异常的监听，`process.on('uncaughtException')`
2. 当监听到旧请求，对 `http` 连接设置 `Connection: close` 响应头，关闭当前正在使用的连接
3. 添加定时器，30 秒后执行，这里的作用是确保杀掉进程，因为调用了定时器的 `unref() `方法，如果进程退出了那这个定时器不会执行，如果进程还有其他活跃的事件使得事件循环一直循环着，那最终也会轮到这个定时器执行，等待一段时间（这里是 30 秒）执行定时器做最后的杀死进程保证
4. 如果是  `cluster` 启动的进程，如 `app` 进程，那调用 `http` 服务的 `server.close()`方法，表示不再接收新的连接
5. 当所有连接关闭后，最后调用 `disconnect()` 断开与 `Master` 的 `IPC` 通道


6. 在关闭了 `IPC` 通道后，`cfork` 监听了 `app` 进程的 `disconnect` 事件，然后根据条件判断是否重新 `fork` 重启一个新的 `App` 进程，其实这个过程就是上面讲过的 `App` 进程退出的逻辑



**`graceful` 包源码如下**

```js
module.exports = function graceful(options) {
    // 1 监听未捕获异常   
    process.on('uncaughtException', function (err) {
        servers.forEach(function (server) {
            if (server instanceof http.Server) {
                server.on('request', function (req, res) {
                    // Let http server set `Connection: close` header, and close the current request socket.
                    req.shouldKeepAlive = false;
                    if (!res._header) {
                        //  对http连接设置 Connection: close响应头，关闭当前的连接
                        res.setHeader('Connection', 'close');
                    }
                });
            }
        }

        // 2 设置定时函数
        // 如果事件循环中只剩下这一个setTimeout则此定时器不会执行，如果事件循环在killTimeout秒中还有其他事件，则该定时器会执行，则在killTimeout秒(默认30s)内执行关闭所有子进程，然后自己退出，
        // make sure we close down within `killTimeout` seconds
        var killtimer = setTimeout(function () {
            console.error('[%s] [graceful:worker:%s] kill timeout, exit now.', Date(), process.pid);
            if (process.env.NODE_ENV !== 'test') {
                // 在退出前先杀死所有子进程
                // kill children by SIGKILL before exit
                killChildren(function () {
                    process.exit(1);
                });
            }
        }, killTimeout);
        
       // 3 解除定时器引用，即使有定时器还未到执行时间，进程如果没其他活跃事件也可以正常退出，而不会让定时器影响到而进程常驻
       if (typeof killtimer.unref === 'function') {
            // only worked on node 0.10+
           killtimer.unref();
       }

    }

    var worker = options.worker || cluster.worker;
    // cluster mode 
    if (worker) {
        for (var i = 0; i < servers.length; i++) {
            var server = servers[i];
            // 4 调用http的close关闭方法，阻止接收新的连接，当所有连接结束后关闭会触发close事件
            server.close();
        }


        // Let the master know we're dead.  This will trigger a
        // 'disconnect' in the cluster master, and then it will fork
        // a new worker.
        worker.send('graceful:disconnect');
        // 5 触发disconnect事件，关闭ICP通道，通知master
        worker.disconnect();
    }
}

// 关闭了IPC通道后， 继续看cfork包里面监听了子进程的disconnect事件，会根据条件判断是否重新fork一个新的子进程
cluster.on('disconnect', function (worker) {
    ......
    // 存该pid
    disconnects[worker.process.pid] = utility.logDate();
    if (allow()) {
      // fork一个新的子进程
      newWorker = forkWorker(worker._clusterSettings);
      newWorker._clusterSettings = worker._clusterSettings;
});
```



**注意：Agent 捕获到未捕获异常时进程不会退出，只是打印日志**

`Agent` 捕获 `uncaughtException` 后最终调用了 `_uncaughtExceptionHandler` 方法，只是记录了一下日志，源码如下

```js
// egg/lib/agent.js
class Agent extends EggApplication {
    constructor(options = {}) {
       this._uncaughtExceptionHandler = this._uncaughtExceptionHandler.bind(this);
       process.on('uncaughtException', this._uncaughtExceptionHandler);
    }
  
     _uncaughtExceptionHandler(err) {
       if (!(err instanceof Error)) {
          err = new Error(String(err));
        }
        /* istanbul ignore else */
        if (err.name === 'Error') {
          err.name = 'unhandledExceptionError';
        }
        this.coreLogger.error(err);
  }

}
```



# 7 kill 杀死进程

## 7.1 一些信号的含义

| 信号    | 数字 | 含义                                                         | 执行                               |
| ------- | ---- | ------------------------------------------------------------ | ---------------------------------- |
| SIGINT  | 2    | 键盘的中断信号(如Ctrl-C)                                     | `kill 2 进程id`  或 `ctrl - c`     |
| SIGKILL | 9    | 发出杀死信号(无条件退出，SIGKILL 是一个特殊的信号，用于强制终止进程，无法被捕获或忽略） | `kill -9 进程id`                   |
| SIGTERM | 15   | 发出终止信号(Terminate信号,通知程序正常退出)                 | `kill 进程id` 或 `kill -15 进程id` |

`kill` 命令并不是直接杀死进程，只是发送一个信号给指定进程，比如 `process.kill(pid[, signal])` 对指定 `pid` 的进程发送一个信号，默认 `SIGTERM` 终止信号，进程可以通过注册信号处理程序来捕获和处理信号



## 7.2 SIGTERM 信号

**使用 `"Kill -15"` 杀死 `Master` 进程，触发 `SIGTERM` 信号，处理如下**

`Master` 会监听 `SIGTERM`、`SIGINT`、`SIGQUIT` 等信号，然后执行 `onSignal` 函数，处理流程可以参考上面“`Master` 的退出”章节

```js
// egg-cluster/lib/master.js

// https://nodejs.org/api/process.html#process_signal_events
// https://en.wikipedia.org/wiki/Unix_signal

// kill -2 master进程时，会触发SIGINT信号
// kill(2) Ctrl-C
process.once('SIGINT', this.onSignal.bind(this, 'SIGINT'));

// kill -3 master进程时，会触发SIGQUIT信号
// kill(3) Ctrl-\
process.once('SIGQUIT', this.onSignal.bind(this, 'SIGQUIT'));

// kill -15 master进程时，会触发SIGTERM信号
// kill(15) default
process.once('SIGTERM', this.onSignal.bind(this, 'SIGTERM'));

onSignal(signal) {
    ......
 }
```



**使用 `"Kill -15"` 杀死 `App` 进程，触发 `SIGTERM` 信号，处理如下**

1. 因为 `App` 进程使用了 `graceful-process` 包，里面有 `process.once('SIGTERM')` 会监听 `SIGTERM` 信号，所以会执行监听里的回调，最终会调用 `process.exit(0)`，然后触发 `exit` 事件退出
2. 最后触发 `App` 进程的 `exit` 事件后，就会执行上面 "`App` 退出" 的流程了



**使用 `"Kill -15"`  杀死 `Agent` 进程，触发 `SIGTERM` 信号，处理如下**

1. `Agent` 进程也类似 `App` 进程， `Agent` 进程也使用了 `graceful-process` 包，里面有 `process.once('SIGTERM')` 会监听 `SIGTERM` 信号，所以会执行监听里的回调，最终会调用 `process.exit(0)`，然后触发 `exit` 事件退出
2. 最后触发 `Agent` 进程的 `exit` 事件后，就会执行上面 "`Agent` 退出" 的流程了



## 7.3 SIGKILL 信号

`SIGKILL` 信号（通过 `kill -9` 发送）是一个特殊的信号，无法被进程捕获、处理或忽略。当进程收到 `SIGKILL `信号时，操作系统会立即终止该进程，无论进程当前正在执行什么操作。具体的处理逻辑如下：

1. 当你运行 `kill -9 <进程ID>` 命令时，操作系统会向指定的进程发送 `SIGKILL` 信号
2. 如果进程没有捕获或忽略 `SIGKILL` 信号，操作系统会立即终止该进程
3. 进程将被终止，所有与该进程相关的资源将被释放，包括打开的文件、网络连接等
4. 由于进程无法处理 `SIGKILL` 信号，因此进程不会触发任何事件或执行任何清理操作
5. 进程的终止可能会导致一些未完成的操作中断，并且可能会导致数据丢失或不一致性

注意：`SIGKILL` 是一个强制终止信号，应该谨慎使用。在正常情况下，最好使用可以被进程捕获和处理的终止信号（如 `SIGTERM`），以便进程有机会进行清理和终止操作。只有在必要的情况下，才应该使用 `kill -9` 命令发送 `SIGKILL` 信号来强制终止进程



## 7.4 Kill -9 杀死 Master后，App 和 Agent 怎么处理

1. ` App` 进程默认会自动退出

因为 `App` 是 `cluster.fork` 出来的， 在 `Master` 进程退出后 `cluster` 模块里面已经有做子进程退出处理了，`cluster` 源码如下

```js
 // https://github.com/nodejs/node/blob/6caf1b093ab0176b8ded68a53ab1ab72259bb1e0/lib/internal/cluster/child.js#L28
process.once('disconnect', () => {
    worker.emit('disconnect');

    if (!worker.exitedAfterDisconnect) {
      // Unexpected disconnect, master exited, or some such nastiness, so
      // worker exits immediately.
      process.exit(0);
    }
});
```

2. `Agent` 子进程默认不会自动退出，但它由 `graceful-process` 提供了退出的功能

* `Agent` 是`child_process.fork ` 出来的，在 `Master` 退出后，`Agent` 进程还在运行，但是它的父进程已经不在了，它将会被 ` init` 进程收养，从而成为孤儿进程

* 通过 ` child_process.fork` 出来的子进程，要实现父进程挂了子进程也跟着挂，需要在子进程添加实现方式，没办法只通过父进程来实现
* 所以有了 `graceful-process`模块，在子进程代码里面执行一下优雅退出逻辑，监听进程的 `disconnect` 事件，主进程退出会关闭 `ipc` 通道，`Agent` 进程监听到 `disconnect` 事件进行优雅退出

```js
// graceful-process/index.js
module.exports = (options = {}) => {
    process.once('disconnect', () => {
        // wait a loop for SIGTERM event happen
        setImmediate(() => {
            // if disconnect event emit, maybe master exit in accident
            logger.error('[%s] receive disconnect event on child_process fork mode, exiting with code:110', label);
            // 调用优雅退出方法
            exit(110); 
        });
    });
}
```



# 8 总结

本文讲了 `Egg.js` 中的多进程模型，包括进程的启动、退出、优雅退出等功能



# 9 参考

* [Egg 官方文档-多进程模型和进程间通讯](https://www.eggjs.org/zh-CN/core/cluster-and-ipc)
* [Egg系列分享 -egg-bin/scripts/cluster](https://cnodejs.org/topic/5f5844d2d22a6b1d622c8432)
* [启动和退出分析](https://www.yuque.com/jianzhen/ibifhs/lu7y3v)

