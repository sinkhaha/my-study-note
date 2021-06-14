# 一、egg的整体认识



![](https://gitee.com/sinkhaha/picture/raw/master/img/egg/Egg%E6%95%B4%E4%BD%93%E8%AE%A4%E8%AF%86.png)



# 二、egg-cluster多进程模型

egg-cluster提供了多进程模型，有效的利用多核CPU，包含了进程的创建、重启、通信、守护等



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



#  三、master/agent/app进程的启动顺序

**启动时序如下**

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

1. master先启动（如通过egg-bin或egg-scripts启动master）

   > master 类似于一个守护进程

2. 然后master启动 agent 进程

3. agent 初始化成功后，通过 IPC 通道通知 master

4. master 根据 CPU 的个数启动相同数目的 worker 进程

5. worker 进程初始化成功后，通过 IPC 通道通知 master

6. 所有的进程初始化成功后，master 通知 agent 和各个 worker 进程应用启动成功



## 1、master的启动

**核心逻辑**

1. `egg-bin`或`egg-scripts`启动master进程
2. 初始化进程管理对象workerManager
3. 初始化进程通信对象messenger
4. 注册一些启动后要执行的方法
5. 监听app/agent的启动和退出事件
6. 启动agent
7. agent启动后，再启动app

**源码分析**

> egg-cluster/lib/master.js文件

```js
// egg-cluster/index.js
// 1. 启动master进程, 如egg-scripts包用spawn启动进程
exports.startCluster = function(options, callback) {
    new Master(options).ready(callback);
};

// egg-cluster/lib/master.js
class Master extends EventEmitter {
    constructor(options) {
        // 2. 初始化进程管理对象workerManager
        //   -> 管理worker和agent对象
        //   -> worker用map存储，key是pid，value是worker实例
        //   -> 获取所有worker所有pid方法
        //   -> 获取worker和agent的数量方法
        //   -> startCheck方法每10s检查一次agent和worker是否都活着，
        //      不是则增加异常计数，异常大于等于3次则触发exception事件
        this.workerManager = new Manager();

        // 3. 初始化各进程之间通信的包装类对象
        //   -> 提供统一的方法让parent、master、agent、app之间进行通信
        //      parent是父进程，比如使用egg-bin启动master，则此时egg-bin进程即parent进程
        this.messenger = new Messenger(this);

        // 4. 注册方法，在所有app启动完成后会执行(即onAppStart方法里的this.ready(true)触发执行）
        this.ready(() => {
            // 发送egg-ready，通知parent、app、agent进程
          
            // 正式环境开启定时检查agent和worker的状态
            if (this.isProduction) {
                this.workerManager.startCheck();
            }
        });
    }

    // 5. 监听app/agent的启动和退出事件
    this.on('agent-exit', this.onAgentExit.bind(this));
    this.on('agent-start', this.onAgentStart.bind(this));
    this.on('app-exit', this.onAppExit.bind(this));
    this.on('app-start', this.onAppStart.bind(this));
    // 在egg-development项目中的agent.js触发
    this.on('reload-worker', this.onReload.bind(this));
    // 监听agent启动事件，然后启动worker进程
    this.once('agent-start', this.forkAppWorkers.bind(this));

    // 监听各种信号
    // kill(2) Ctrl-C
    process.once('SIGINT', this.onSignal.bind(this, 'SIGINT'));
    // kill(3) Ctrl-\
    process.once('SIGQUIT', this.onSignal.bind(this, 'SIGQUIT'));
    // kill(15) default  ，如kill master进程id会触发
    process.once('SIGTERM', this.onSignal.bind(this, 'SIGTERM'));
    process.once('exit', this.onExit.bind(this));

    // 获取一个可用的端口，然后启动agent进程 
    this.detectPorts()
        .then(() => {
            // 7. 启动agent
            this.forkAgentWorker();
        });
        
    }
}

```



**master 类似于一个agent和app的守护进程的存在**

- 负责 agent 的启动、退出、重启
- 负责各个 worker 进程的启动、退出、以及 refork ，在开发模式下负责重启reload-worker
- 负责中转 agent 和各个 worker 之间的通信
- 负责中转各个 worker 之间的通信



## 2、agent的启动

**核心逻辑**

1. 调用启动agent
2. 执行agent启动程序，监听异常，添加agent优雅退出
3. 监听agent进程退出事件

**源码分析**

> egg-cluster/lib/master.js文件

```js
this.detectPorts()
    .then(() => {
        //  启动agent
        this.forkAgentWorker();
    });

forkAgentWorker() {
    // 使用child_process.fork启动，跟app使用cfork包有区别
    // egg-cluster/lib/agent_worker.js为启动文件
    const agentWorker = childprocess.fork(this.getAgentWorkerFile(), args, opt);
    
    // agent进程监听发给它的消息
    // 如agent进程调用sendToApp跟app进程通信时，会经过这里转发给app进程
    agentWorker.on('message', msg => {});
  
    // 监听error事件
    agentWorker.on('error', err => {});
  
    // agent退出事件，发消息通知master
    agentWorker.once('exit', (code, signal) => {
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
  
}

// agent进程的启动文件
getAgentWorkerFile() {
    return path.join(__dirname, 'agent_worker.js');
}


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
    this.logger.info('[master] agent_worker#%s:%s started (%sms)',
        this.agentWorker.id, this.agentWorker.pid, Date.now() - this.agentStartTime);
}

```

> egg-cluster/lib/agent_worker.js

```js
// 实例化agent 
// framework为egg包，即为egg中导出的Agent对象exports.Agent = require('./lib/agent');
const Agent = require(options.framework).Agent;

// egg/lib/agent.js
const agent = new Agent(options); 

// 注册方法，在agent启动后该注册的方法会被调用，触发agent-start事件，master监听到后开始启动app进程
agent.ready(err => {
    process.send({ action: 'agent-start', to: 'master' });
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



## 3、app worker的启动

> egg-cluster/lib/master.js文件

1. 监听到agent-start启动事件，然后使用cfork包启动app
2. 监听发给app的消息

```js
// 监听到agent启动事件后，调用启动app 
this.once('agent-start', this.forkAppWorkers.bind(this));

forkAppWorkers() {
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
    
    // 当新的进程被fork时触发
    cluster.on('fork', worker => { 
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
      
       // disconnect事件，此处只是打印日志，不做具体处理，实际处理在cfork包中
       cluster.on('disconnect', worker => {
           this.logger.info('[master] app_worker#%s:%s disconnect, suicide: %s, state: %s, current workers: %j',
           worker.id, worker.process.pid, worker.exitedAfterDisconnect, worker.state, Object.keys(cluster.workers));
        });
      
        // 监听退出事件，然后通知master，例如当app worker出现OOM被系统杀死时触发
        cluster.on('exit', (worker, code, signal) => {
            //  app worker退出后通知master 
            //   -> this.on('app-exit', this.onAppExit.bind(this));
            //     -> 不是开发环境，isDevReload为false，则记录错误信息
            //     ->移除所有监听器，发送egg-pids通知agent进程
            //     -> 判断isAllAppWorkerStarted的值为，即所有app worker是否都启动了
            //       ->都启动了，发送app-worker-died给parent进程，在正式环境下会refork重新启动一个新的app worker进程
            //       ->没有都启动，则退出当前app worker
            this.messenger.send({
                action: 'app-exit',
                data: {
                    workerPid: worker.process.pid,
                    code,
                    signal,
                },
                to: 'master',
                from: 'app',
            });
        }
                   
        // 监听listening，app worker调用listen()后触发，通知master
        cluster.on('listening', (worker, address) => {
            // app worker启动后通知master
            //   -> this.on('app-start', this.onAppStart.bind(this));
            //     -> 发送egg-pids给agent进程
            //     -> 当所有app woker启动后，发送egg-ready给app
            //     -> this.ready(true)触发执行master实例化时ready注册的方法 
            this.messenger.send({
                action: 'app-start',
                    data: {
                        workerPid: worker.process.pid,
                        address,
                    },
                    to: 'master',
                    from: 'app',
                });
             });
        }       
}

// worker进程的启动文件
getAppWorkerFile() {
    return path.join(__dirname, 'app_worker.js');
}
  

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
        // 触发master上注册的函数
        this.ready(true);
    }
}

  
```

> egg-cluster/lib/app_worker.js

```js
// 实例化app
// framework为egg包，
// 即为egg中导出的Application对象exports.Application=require('./lib/application');
const Application = require(options.framework).Application;
// egg/lib/application.js
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
```

##  4、agent和worker的启动方式差异

master 启动 agent 和 worker 的方式不一样

1. 启动 agent 使用的是 `child_process 的 fork`方法

2. 启动各个 worker 使用的是`cfork`包，里面使用的是 `cluster 的 fork`方法

   > 启动多个worker，在 cluster 的预处理下能对同一端口进行监听而不会产生端口冲突，cluster默认使用
   >
   > `round-robin轮询策略`进行负载均衡把收到的 http 请求合理地分配给各个 worker 进行处理

3. cfork包的作用：更方便的使用cluster去fork和restart，处理`uncaughtException`异常并记录日志



# 四、平滑重启/优雅退出

## 1、agent的平滑重启

> egg-cluster/lib/master.js文件

1. agent监听exit事件，发送agent-exit通知master
2. master接收到agent-exit重新启动agent进程

```js
//  agent监听退出事件，发送agent-exit事件给master，master监听后执行onAgentExit方法
agentWorker.once('exit', (code, signal) => { // 如SIGKILL信号(kill -9)
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


onAgentExit() {
    ...
    // 执行一些退出工作
    // 如果有app worker是启动状态才启动一个新的agent进程，否则master直接退出
    if (this.isStarted) {
       setTimeout(() => {
            this.logger.info('[master] new agent_worker starting...');
            this.forkAgentWorker();
        }, 1000);
        this.messenger.send({
            action: 'agent-worker-died',
            to: 'parent',
        });
    } else {
        process.exit(1);
    }
}
```

## 2、app的平滑重启

> 由cfork包实现，cfork包的index.js文件

1. cfork包监听disconnect事件
2. 然后判断是否需要重新fork一个新进程

```js
// 监听disconnect断连事件
cluster.on('disconnect', function (worker) {
    // worker已经死了，不需要refork
    // worker has terminated before disconnect
    var isDead = worker.isDead && worker.isDead();
    if (isDead) {
        return;
    }
    // 是否设置了不需要重新fork标识，比如主动杀死master时会杀死所有worker，此时不需要refork
    if (worker.disableRefork) {
        return;
    }

    disconnects[worker.process.pid] = utility.logDate();
    
    // 允许重新fork
    if (allow()) {
        // fork一个新的worker进程
        newWorker = forkWorker(worker._clusterSettings);
        newWorker._clusterSettings = worker._clusterSettings;
    }
});
  
// 工作进程exit事件 (类似disconnect事件)
cluster.on('exit', function (worker, code, signal) {
    // 设置了不需要refork标识
    if (worker.disableRefork) {
        // worker is killed by master
        return;
    }

    unexpectedCount++;
    
    // 重新fork
    if (allow()) {
        newWorker = forkWorker(worker._clusterSettings);
        newWorker._clusterSettings = worker._clusterSettings;
    } 
    cluster.emit('unexpectedExit', worker, code, signal);
});

var limit = options.limit || 60;
var duration = options.duration || 60000; // 1 min
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

    // 重启次数小于限制 或重启间隔大于 duration 才允许重启
    var span = reforks[reforks.length - 1] - reforks[0];
    var canFork = reforks.length < limit || span > duration;

    if (!canFork) {
        cluster.emit('reachReforkLimit');
    }

    return canFork;
 }

```



## 3、agent/app进程优雅退出

主要由`graceful-process包`实现优雅退出(在程序退出前执行一些逻辑，如释放资源之类的操作)，即在进程退出前执行某一些方法



> agent调用优雅退出 

```js
// egg-cluster/lib/agent_worker.js 
gracefulExit({
    logger: consoleLogger,
    label: 'agent_worker',
    beforeExit: () => agent.close(), // 退出前执行的方法，即生命周期函数beforeClose
});
```



> app调用优雅退出 

```js
// egg-cluster/lib/app_worker.js
gracefulExit({
    logger: consoleLogger,
    label: 'app_worker',
    beforeExit: () => app.close(), // 即生命周期函数beforeClose
});
```



>  优雅退出实际实现

```js
// graceful-process/index.js
module.exports = (options = {}) => {
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
                exit(110); // 优雅退出
            });
        });
    }
}

```



# 五、强制杀死master后agent的处理

**master 被使用`kill -9`强制杀死退出后，app和agent怎么处理？**

1. ` app` 子进程默认`会自动退出`

因为app是`cluster.fork`出来的，在master进程退出后`cluster`模块已经有做子进程退出处理了，代码如下

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



2. `agent` 子进程默认`不会自动退出`，由`graceful-process`提供退出功能

* agent是`child_process.fork`出来的，在master退出后，agent 进程还在运行，但是它的父进程已经不在了，它将会被` init 进程`收养，从而成为`孤儿进程`

* 通过` child_process.fork` 出来的子进程，要实现父进程挂了子进程也跟着挂，需要在子进程添加实现方式，没办法只通过父进程来实现
* 所以有了`graceful-process模块`，在子进程代码里面执行一下`优雅退出`逻辑，process`监听disconnect事件`，主进程退出ipc通道关闭，agent进程监听到事件进行优雅退出

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



# 六、app进程退出处理

## 1、 app出现uncaughtException时优雅退出

**app未捕获异常异常退出整体流程**

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



**`graceful库`提供未捕获异常的优雅退出**

> egg/lib/application.js

1. graceful中使用了`process.on('uncaughtException')`捕获异常（在egg包的application.js文件）
2. 并在回调函数里面`关闭当前正在使用的TCP连接`
3. 添加定时确保杀掉进程，等待一段时间是为了给一定时间处理完当前连接中的请求，默认30s
4. `server.close()`不再接收新的连接，当所有连接关闭后，` 断开与master的IPC通道`，不再接受新的请求
5. 调用`disconnect()`，在关闭了IPC通道后，cfork监听了app进程的disconnect事件，然后根据条件判断是否重新fork一个新的app进程



**源码分析**

> graceful包

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
        // 如果事件循环中只剩下这一个setTimeout则此定时器不会执行，如果如果事件循环在killTimeout秒中还有其他事件，则该定时器会执行，则在killTimeout秒(默认30s)内执行关闭所有子进程，然后自己退出，
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
       if (typeof killtimer.unref === 'function') {
            // only worked on node 0.10+
           killtimer.unref();
       }

    }

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



## 2、app出现OOM、系统异常重新fork的逻辑

和未捕获异常发生时不一样，出现`未捕获异常`还有机会让进程继续执行，而`OOM/系统异常`等只能够让当前app进程`直接退出`

1. `egg-cluster/lib/master.js` 的`cluster.on('exit')`，触发`app-exit`事件进行一些关闭的后续处理
2. `cfork`包的`cluster.on('exit')` 会重新判断是否去 fork 一个新的 Worker

```js
cluster.on('exit', function (worker, code, signal) {
    // 是程序异常会通过监听uncatughException重新fork一个子进程，所以这里直接返回
    var isExpected = !!disconnects[worker.process.pid];
    if (isExpected) {
      delete disconnects[worker.process.pid];
      // worker disconnect first, exit expected
      return;
    }
  
    // 是master杀死的子进程，不需要fork
    if (worker.disableRefork) {
      // worker is killed by master
      return;
    }

    if (allow()) {
      newWorker = forkWorker(worker._clusterSettings);
      newWorker._clusterSettings = worker._clusterSettings;
    } else {
      ......
    }
    cluster.emit('unexpectedExit', worker, code, signal);
});

```





# 七、执行kill操作

## 一、一些信号的含义

| 信号    | 数字 | 含义                                         | 执行                               |
| ------- | ---- | -------------------------------------------- | ---------------------------------- |
| SIGINT  | 2    | 键盘的中断信号(如Ctrl-C)                     | `kill 2 进程id`  或 `ctrl - c`     |
| SIGKILL | 9    | 发出杀死信号(无条件退出)                     | `kill -9 进程id`                   |
| SIGTERM | 15   | 发出终止信号(Terminate信号,通知程序正常退出) | `kill 进程id` 或 `kill -15 进程id` |

## 二、杀死Master

### 1、kill -15 master进程

1. master监听到`SIGTERM`信号
2. 调`this.close`方法，先杀`所有worker`，再杀`agent`(默认会等待5s后执行，超时则会被强制关闭)
3. 最后master退出`process.exit(0)`

**源码如下**

```js
// egg-cluster/lib/master.js 
// 触发master.js的SIGTERM
process.once('SIGTERM', this.onSignal.bind(this, 'SIGTERM'));

onSignal(signal) {
    if (this.closed) return;

    this.logger.info('[master] receive signal %s, closing', signal);
    this.close();
}

close() {
    this.closed = true;
    const self = this;
    co(function* () {
      try {
        // 先关闭所有app，再关闭所有agent
        yield self._doClose();
        self.log('[master] close done, exiting with code:0');
        process.exit(0);
      } catch (e) /* istanbul ignore next */ {
        this.logger.error('[master] close with error: ', e);
        process.exit(1);
      }
    });
  }
```

**执行流程**

1. master的`process.once('SIGTERM', this.onSignal.bind(this, 'SIGTERM'));`

2. 调master中`close`方法的`_doClose`方法的`killAppWorkers`杀死app进程(触发`SIGTERM`)

3. cfork包`cluster.on('disconnect')`监听到ipc断开，判断是否需要重新fork

4. master中`cluster.on('disconnect')`，只是打印日志

5. graceful包的`process.once('SIGTERM')`(由第2步触发的)接着触发`exit事件`

6. graceful包监听到`process.once('exit')`(由第5步触发)

7. cfork包里的`cluster.on('exit')`(由第5步触发)

8. 调master中close方法的`_doClose`方法的的`killAgentWorker`

9. 最后master退出


**调试打印日志如下**

```bash
2021-04-07 21:17:57,857 INFO 61376 [master] receive signal SIGTERM, closing
2021-04-07 21:17:57,857 INFO 61376 [master] send kill SIGTERM to app workers, will exit with code:0 after 5000ms
2021-04-07 21:17:57,857 INFO 61376 [master] wait 5000ms
[2021-04-07 21:17:57.924] [cfork:master:61376] worker:61389 disconnect (exitedAfterDisconnect: true, state: disconnected, isDead: false, worker.disableRefork: true)
[2021-04-07 21:17:57.925] [cfork:master:61376] don't fork, because worker:61389 will be kill soon
2021-04-07 21:17:57,925 INFO 61376 [master] app_worker#1:61389 disconnect, suicide: true, state: disconnected, current workers: []
Waiting for the debugger to disconnect...
[2021-04-07 21:17:57.941] [cfork:master:61376] worker:61389 exit (code: 0, exitedAfterDisconnect: true, state: dead, isDead: true, isExpected: false, worker.disableRefork: true)
2021-04-07 21:17:58,327 INFO 61376 [master] send kill SIGTERM to agent worker, will exit with code:0 after 5000ms
2021-04-07 21:17:58,327 INFO 61376 [master] wait 5000ms

2021-04-07 21:17:58,792 INFO 61376 [master] exit with code:0

```



### 2、kill -2 master进程 或 Ctrl +C

> egg-cluster/lib/master.js

```js
// 触发master.js的SIGINT 实际具体执行逻辑和上面 kill -15 一样
process.once('SIGINT', this.onSignal.bind(this, 'SIGINT'));
```



### 3、kill -9 master进程

> `kill -9` 会触发`SIGKILL信号`，但master没监听`SIGKILL信号`

**app进程退出处理**

1. `graceful-process包`的`cluster.worker.once('disconnect')`

   >  cluster内部也会监听到`disconnect`，然后执行`exit(0)`，触发`exit事件`

2. `graceful-process包`的`process.once('exit')`监听到步骤1的`exit事件`

**agent进程退出处理**

1. `graceful-process包`的`process.once('disconnect')`触发`process.exit(110)`

2. `graceful-process包`的`process.once('exit')`监听到步骤1的`exit`事件

   

**源码如下**

```js
// graceful-process包
process.once('exit', code => {
    const level = code === 0 ? 'info' : 'error';
    printLogLevels[level] && logger[level]('[%s] exit with code:%s', label, code);
});

if (cluster.worker) {
    // cluster mode
    // https://github.com/nodejs/node/blob/6caf1b093ab0176b8ded68a53ab1ab72259bb1e0/lib/internal/cluster/child.js#L28
    cluster.worker.once('disconnect', () => { // cluster内部也会监听到disconnect然后执行exit
        // ignore suicide disconnect event
        if (cluster.worker.exitedAfterDisconnect) return;
        logger.error('[%s] receive disconnect event in cluster fork mode, exitedAfterDisconnect:false', label);
    });
} else {
    // child_process mode
    process.once('disconnect', () => {
        // wait a loop for SIGTERM event happen
        setImmediate(() => {
            // if disconnect event emit, maybe master exit in accident
            logger.error('[%s] receive disconnect event on child_process fork mode, exiting with code:110', label);
            exit(110);
        });
    });
}
```

**调试打印日志如下**

```js
2021-04-07 21:22:21,892 ERROR 61850 [app_worker] receive disconnect event in cluster fork mode, exitedAfterDisconnect:false

2021-04-07 21:22:21,894 ERROR 61844 [agent_worker] receive disconnect event on child_process fork mode, exiting with code:110
2021-04-07 21:22:21,899 ERROR 61844 [agent_worker] exit with code:110

```



## 三、杀死agent

### 1、Kill -15 agent进程

`graceful-process包`和`master中的agentWorker`都能监听到`SIGTERM信号`

**graceful-process包**

1. `graceful-process包`的`process.once('SIGTERM')`监听到`SIGTERM`信号，调用`process.exit(0)`，然后触发`exit事件`
2. `graceful-process包`的`process.once('exit')`监听到`exit`后打印日志

**源码如下**

```js
// graceful-process
process.once('SIGTERM', () => {
    printLogLevels.info && logger.info('[%s] receive signal SIGTERM, exiting with code:0', label);
    exit(0); // 里面调用了process.exit(0)  agent进程退出
});

process.once('exit', code => {
    const level = code === 0 ? 'info' : 'error';
    printLogLevels[level] && logger[level]('[%s] exit with code:%s', label, code);
});
```

**master中的agentWorker**

1. `agentWorker.once('exit',(code, signal))`监听到`SIGTERM信号`，然后`agent-exit`事件
2. master监听到`agent-exit`重新fork一个agent进程

```js
// master.js 
agentWorker.once('exit', (code, signal) => {  // signal 为 SIGTERM
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

### 2、Kill -9 agent进程

1. `agentWorker.once('exit',(code, signal))`监听到`SIGKILL信号`，然后`agent-exit`事件
2. 然后master监听到`agent-exit`重新fork一个agent进程

```js
// master.js
agentWorker.once('exit', (code, signal) => { // signal 为 SIGKILL
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



## 四、杀死app

### 1、Kill -15 app进程

**执行流程**

1. `graceful-process包`的`process.once('SIGTERM')`监听到`SIGTERM`信号，调用`process.exit(0)`，然后触发`exit事件`
2. `graceful-process包`接收到exit事件`process.once('exit')`后打印日志
3. `cfork包`的`cluster.on('disconnect')`监听到disconnect事件，然后判断是否要重新fork一个app进程
4. master里的worker监听`cluster.on('disconnect')`只是打印日志
5. cfork包的`cluster.on('exit')`监听到exit事件，判断是否需要fork
6. master里的worker监听 `cluster.on('exit')`，触发`app-exit事件`

### 2、Kill -9 app进程

**执行流程**

1. cfork包的`cluster.on('disconnect')`，判断是否重新fork
2. master文件worker监听disconnect，即`cluster.on('disconnect')`
3. cfork包的`cluster.on('exit')`
4. master里的worker监听 `cluster.on('exit')`，触发`app-exit事件`





# 八、egg使用到的一些库

1. get-ready： 注册一次性事件

2. ready-callback： 在注册的异步任务执行完后执行注册的回调

3. sendmessage： 进程间通信

4. detect-port： 检查端口是否被占用

5. cfork库：更好创建子进程

6. extend2：克隆对象

7. utility：工具库， 包括md5，sha1，sha256等

8. is-type-of：判断类型

9. co： 执行generator函数

10. node-homedir：  获取用于的home文件夹路径

11. common-bin： 命令行库

12. cluster-client：  Leader/Follower 模式

13. semver： 判断npm的版本

14. once：使函数只执行一次

15. debug： 调试

16. autod：更新依赖到最新版本

    

## 参考

* https://zhuanlan.zhihu.com/p/25457918

* https://cnodejs.org/topic/5f5844d2d22a6b1d622c8432

* https://eggjs.org/zh-cn/core/cluster-and-ipc.html

* https://zhuanlan.zhihu.com/p/62892856

* https://www.yuque.com/jianzhen/ibifhs/cluster

* https://www.imooc.com/article/301846?block_id=tuijian_wz

