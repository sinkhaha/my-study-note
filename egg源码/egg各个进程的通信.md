# 一、各个进程间的通信



## 1、 整体通信流程图

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/egg/%E9%80%9A%E4%BF%A1%E6%B5%81%E7%A8%8B%E5%9B%BE.png)



## 2、master中的消息发送类messenger

> egg-cluster/lib/utils/messenger.js

`egg-cluster`的master主进程中维护了一个`messenger对象`，messenger提供了`转发消息`的方法，在`parent、master、agent、app`之间提供通信的机制

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



### 各个进程的说明

1. parent进程： 即egg-bin/egg-scripts等启动egg-cluster中master的进程

2. master进程： 即egg-cluster的master主进程

   > master 继承了 `events 模块`，可以通过事件机制来进行消息的通信

3. agent进程： 即master主进程通过child_process.fork出来的子进程 

   > 相互可以通过 IPC 通道进行通信

4. app进程： 即master主进程通过cfork包(cluster.fork)出来的子进程

   > 相互可以通过 IPC 通道进行通信



注意：`agent`和`各个 worker`之间 、` worker和worker ` 之间是不同进程，无法直接进行通信，需要经过 master 转发，egg-cluster 封装了一个 `messenger 的工具类`，对各个进程间消息转发进行了封装



### messenger类的作用

1. master监听message事件，接收发给master的消息，如parent发给master的消息
2. send方法统一处理消息的转发

* master/agent/app 发消息给 parent是通过`process.send()`

* agent/app发消息给master是通过events事件机制的emit

* (parent/app/agent)发消息给app或agent，都得通过master转发，最后通信处理是在`sendmessage包`

  

**源码分析**

> egg-cluster/lib/utils/messenger.js

```js
class Messenger {
    // 1. master接收parent的消息
    constructor(master) {
        this.master = master;
        // 如果node进程是使用ipc通道衍生的，可以用process.send方法发消息给父进程，如果不是通过ipc通道衍生的，则process.send是undefined
       // 如用egg-bin的dev命令启动
        this.hasParent = !!process.send;
      
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

        // 基于sendmessage模块，可以指定进程id
        // parent -> master -> app
        // agent -> master -> app
        if (data.to === 'app') {
            debug('%s -> %s, data: %j', data.from, data.to, data);
            this.sendToAppWorker(data);
            return;
        }

        // 基于sendmessage模块，master只是中转
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



## 3、agent和app之间的消息发送类messenger

### messenger类的作用

1. `agent和app进程`都拥有一个`messenger对象`，解决agent和app之间的通信

2. messenger是agent和app之间通信的通道，它们之间的通信通过master进程进行`中转`，messenger监听到master进程发给agent或app进程的消息后进行处理

3. messenger也对外提供了一些`agent和app进程间通信的方法`（如broadcast、sendToApp、sendToAgent、sendRandom、sendTo等）

   > 可以在egg-ready事件后，调用相关方法在app和agent之间通信，如 app发消息通知agent ，app.messenger.sendToAgent('事件名', { xx:xx })



**源码分析**

> egg/lib/core/messenger/ipc.js

```js
class Messenger extends EventEmitter {
    constructor() {
        super();
        this.pid = String(process.pid);
        // pids of agent or app maneged by master
        // - retrieve app worker pids when it's an agent worker
        // - retrieve agent worker pids when it's an app worker
        this.opids = [];
        this.on('egg-pids', pids => {
            this.opids = pids;
        });
        
        this._onMessage = this._onMessage.bind(this);
        
        // (agent或app)接收来自master进程的消息
        process.on('message', this._onMessage);
     }
  
     _onMessage(message) {
         if (message && is.string(message.action)) {
             this.emit(message.action, message.data);
         }
     }
}
```



# 二、例子：agent和app进程通信的整体流程分析

## 例子：测试agent发送消息给app，即agent调sendToApp方法

在项目的agent.js调用`app.messenger.sendToApp(action, data)`方法发消息给全部app

```js
// 自己项目下的agent.js发消息给全部app
class AgentBootHook {
    constructor(app) {
        this.app = app;
      
        this.app.messenger.once('egg-ready', () => {
            // 测试agent发test-event给所有app
            this.app.messenger.sendToApp('test-event', { name: 'haha' });
        });
    }
    
}
module.exports = AgentBootHook;
```

### 分析

1. egg包messenger的`ipc.js`的sendToApp 

```js
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

2. `sendmessage包`的`send方法`，发消息给agent进程

```js
module.exports = function send(child, message) {
  // 非子进程(agent/app)直接触发emit
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
    
    // 注意：此时的child为agent进程实例，即发消息给agent
    return child.send(message);
  }
  ......
};
```

3. `egg-cluster`包的的master监听到发给`agent进程`的消息，然后转发消息给app进程

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
      
    // 增加调试日志
    // 2. agentWorker.onMessage= { action: 'test-event', data: { name: 'haha' }, to: 'app' }
    if (msg.action == 'test-event') {
        console.log(`2. agentWorker.onMessage=`, msg);
    }
   
    msg.from = 'agent';
    this.messenger.send(msg);
});
 

// egg-cluster/lib/utils/messenger.js
send(data) {
    if (data.to === 'app') {
        // 增加调试日志
        // 3. sendToAppWorker = {
        // action: 'test-event',
        // data: { name: 'haha' },
        // to: 'app',
        // from: 'agent'
        // }
        if (data.action == 'test-event') {
            console.log(`3. sendToAppWorker =`, data);
        }
        this.sendToAppWorker(data);
        return;
    }
}
 

// 然后还是sendmessage包的send方法 
child.send(message); // 此时的child为app进程实例，即发消息给app
 
```

4. egg包的`ipc.js`的`process.on('message', this._onMessage)`监听发给app进程的消息，然后触发事件

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

5. 最后在app进程中监听事件即可接收到消息



### 调试日志如下

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



### 总结

即agent发消息给app是`通过master中转`

* agent发消息给app 

* 先把发消息给agent进程，master监听到agent进程的消息

* master接收到agent进程的消息后发给所有app进程 

* app进程接收到消息触发emit方法，最后其他app进程监听对应事件即可接收到消息




# 三、egg中各个事件的触发与接收

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/egg/egg%E5%90%84%E8%BF%9B%E7%A8%8B%E7%9B%91%E5%90%AC%E7%9A%84%E6%B6%88%E6%81%AF.png)



## 参考

* https://eggjs.org/zh-cn/core/cluster-and-ipc.html

* https://www.yuque.com/jianzhen/ibifhs/xvfp27



