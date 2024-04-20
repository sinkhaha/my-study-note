# 一、整体

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/egg/common-bin.png)

# 二、egg-bin(开发者工具)

> 是egg的命令行调试工具，继承common-bin

## debug调试指令

> egg-bin/egg-bin/lib/cmd/debug.js

```js
// 创建egg-cluster进程，serverBin为egg-bin包start-cluster的绝对路径
const child = cp.fork(this.serverBin, eggArgs, options);
```



# 三、egg-scripts(正式环境)

> 继承common-bin，主要提供`start`和`stop`命令

## start命令

>  egg-scripts/lib/cmd/start.js

```js
// start的逻辑
class StartCommand extends Command {
    constructor(rawArgv) {
        super(rawArgv);
        // 使用egg包的startCluster启动进程
        this.serverBin = path.join(__dirname, '../start-cluster');

        // 参数定义
        this.options = {
            // 是否后台运行
            daemon: {
                description: 'whether run at background daemon mode',
                type: 'boolean',
            }
        }
    }

    * run(context) {
        // isDaemon 是否后台运行
        const isDaemon = argv.daemon;
        const command = argv.node || 'node';
        // whether run in the background.
        if (isDaemon) {
            const child = this.child = spawn(command, eggArgs, options);
            child.on('message', msg => {
                /* istanbul ignore else */
                if (msg && msg.action === 'egg-ready') {
                    this.isReady = true;
                    // 子进程独立，允许父进程独立于子进程退出
                    child.unref();

                    child.disconnect();
                    this.exit(0);
                }
            });
        }
    }
}
```



## stop命令

> egg-scripts/lib/cmd/stop.js

```js
// stop的逻辑
class StopCommand extends Command {
    // 参数定义......

    * run(context) {
        // findNodeProcess找到该服务启动的所有对应的进程
        let processList = yield this.helper.findNodeProcess(item => {
            const cmd = item.cmd;
            // 找出master进程 start-cluster
            return argv.title ?
                cmd.includes('start-cluster') && cmd.includes(util.format(osRelated.titleTemplate, argv.title)) :
                cmd.includes('start-cluster');
        });

        let pids = processList.map(x => x.pid);

        if (pids.length) {
            this.logger.info('got master pid %j', pids);

            // 通过pid杀死master，触发SIGTERM信号，master监听到调用close方法会杀死所有的app和agent
            this.helper.kill(pids);
            // 默认等待5秒
            // wait for 5s to confirm whether any worker process did not kill by master
            yield sleep(argv.timeout || '5s');
        }

        // 找到未被杀死的app/agent进程
        processList = yield this.helper.findNodeProcess(item => {
            const cmd = item.cmd;
            return argv.title ?
                (cmd.includes(osRelated.appWorkerPath) || cmd.includes(osRelated.agentWorkerPath)) && cmd.includes(util.format(osRelated.titleTemplate, argv.title)) :
                (cmd.includes(osRelated.appWorkerPath) || cmd.includes(osRelated.agentWorkerPath));
        });
        pids = processList.map(x => x.pid);

        // 发送SIGKILL杀死agent/app进程
        if (pids.length) {
            this.logger.info('got worker/agent pids %j that is not killed by master', pids);
            this.helper.kill(pids, 'SIGKILL');
        }

    }
}
```



---



### 参考

https://cnodejs.org/topic/5f5844d2d22a6b1d622c8432
