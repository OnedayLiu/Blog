# Egg源码解析之egg-cluster

<!-- TOC -->

- [Egg源码解析之egg-cluster](#egg源码解析之egg-cluster)
  - [egg-cluster是什么](#egg-cluster是什么)
  - [egg多进程模型](#egg多进程模型)
  - [egg-cluster源码解析](#egg-cluster源码解析)
    - [ready](#ready)
    - [Master(egg-cluster/lib/master.js)](#masteregg-clusterlibmasterjs)
    - [Master#constructor](#masterconstructor)
    - [Master#forkAgentWorker](#masterforkagentworker)
    - [Master#onAgentStart](#masteronagentstart)
    - [Master#forkAppWorkers](#masterforkappworkers)
    - [Master#onAppStart](#masteronappstart)
    - [Master#onAgentExit](#masteronagentexit)
    - [Master#onAppExit](#masteronappexit)
    - [Master#onReload](#masteronreload)
    - [Master#onExit](#masteronexit)
    - [Master#onSignal和Master#close](#masteronsignal和masterclose)

<!-- /TOC -->

## egg-cluster是什么

为了将多核CPU的性能发挥到极致，最大程度地榨干服务器资源，egg采用多进程模型，解决了一个Node.js进程只能运行在一个CPU上的问题，[egg-cluster](https://github.com/eggjs/egg-cluster)是用于egg多进程管理的基础模块，负责底层的IPC通道的建立以及处理各进程的通信

## egg多进程模型

![](/asserts/168aa630b7012b187c5f7ddd6872e2e130532548.png)

* **master** 主进程
* **worker** `master`的子进程，一般是根据服务器有多少个CPU启动多少个这样的`worker`进程，主要用于对外服务，处理各种业务层面的事情
* **agent** `master`的子进程，主要处理公共资源的访问，如文件监听，或者帮worker处理一些公共事务，如一些事情是不需要每个`worker`都做一次的，`agent`帮忙做完之后通知它们执行之后的操作

`master`类似于一个守护进程的存在：

* 负责`agent`的启动、退出、重启以及
* 负责各个`worker`进程的启动、退出、以及refork，在开发模式下负责重启
* 负责`agent`和各个`worker`之间的通信
* 负责各个`worker`之间的通信

各进程的启动顺序：

* `master`启动后先启动`agent`进程
* `agent`初始化成功后，通过`IPC`通道通知`master`
* `master`根据CPU的个数启动相同数目的`worker`进程
* `worker`进程初始化成功后，通过`IPC`通道通知`master`
* 所有的进程初始化成功后，`master`通知`agent`和各个`worker`进程应用启动成功

启动方式差异：

从上图可以看出，`master`启动`agent`和`worker`的方式明显不一样，启动`agent`使用的是`child_process`的fork模式，启动各个`worker`使用的是`cluster`的fork模式，为什么不能都使用同一种方式来启动？因为它们所负责处理的事情性质是不一样的，`agent`是类似于作为各个`worker`秘书的存在，只负责帮它们处理轻量级的服务，是不直接对外提供http访问的，所以`master`用`cluster.fork`把各个`worker`启动起来，并提供对外访问http访问，这些`worker`在`clustr`的预处理下能够对同一端口进行监听而不会产生端口冲突，同时进行负载均衡使用round-robin策略把收到的http请求合理地分配给各个`worker`进行处理

进程间通信：

`master`和`agent/worker`是real communication，`agent`和各个`worker`之间以及各个`worker`之间virtual communication

* `master`继承了events模块，拥有events监听、发送消息的能力，`master`进程自身是通过订阅者模式来进行事务处理的，所以在`master`的源码里面并没有看到过多的`callback hell`
* `master`是`agent`的父进程，可以通过IPC通道进行通信
* `master`是`worker`的父进程，可以通过IPC通道进行通信
* `agent`和各个`worker`之间是无法进行通信的，毕竟是不同进程，所以需要借助`master`的力量进行转发，`egg-cluster`封装了一个`messenger`的工具类，对各个进程间消息转发进行了封装
* 各个`worker`之间由于是不同进程，也是无法进行通信的，原理同上

各进程的状态通知

* `worker`启动成功后`master`会对其状态进行监听，对于退出或者失联的`worker` `master`是清楚的，在这情况下`master`会对这些`worker`之前所绑定的事件进行销毁防止内存泄露，并且通知`agent`，最后refork出同等数量的`worker`保证业务的顺利进行，对`worker`的fork和refork操作都是通过工具类`cfork`进行的
* `agent`启动成功后`master`会对其状态进行监听，对于退出或者失联的`agent` `master`是清楚的，在这情况下`master`会对这些`agent`之前所绑定的事件进行销毁防止内存泄露，并且通知各个`worker`，最后重启`agent`进程保证业务的顺利进行
* `master`退出了或者失联了，`worker`怎么办？不用担心，`cluster`已经做好了这样的处理，当父进程退出后子进程自动退出
* `master`退出了或者失联了，`agent`也像`worker`一样退出吗？然而并不是！这是`child_process.fork`和`cluster.fork`的不同之处，`master`退出了或者失联了，`agent`进程还继续运行，但是它的父进程已经不在了，它将会被`init`进程收养，从而成为孤儿进程，当这样的孤儿进程越来越多的时候服务器就会越来越卡。所以`master`退出后需要指定`agent`也一起退出！

开发模式

**开发**模式下`agent`会监听相关文件的改动，然后通知`master`对`worker`进行重启操作
> 开发模式下开启`egg-development`插件，对相关文件进行监听，监听到有文件改动的话向`master`发送`reload-worker`事件

## egg-cluster源码解析

### ready

读源码前需要理解两个模块的作用：

* **messenger**，负责`master`，`agent`，`worker`IPC通信的消息转发
* **cfork**，负责`worker`的启动，状态监听以及重启操作

写这篇文章的时候egg社区版最新版是1.6.0，下面的内容以该版本为准
egg是通过`index.js`作为入口文件进行启动的，输入以下代码然后就可以成功启动了

```js
const egg = require('egg');
egg.startCluster(options, () => {
  console.log('started');
});
```

入口文件代码如此简单，那egg底层做了些什么？比如`egg.startCluster`这个方法里面做了些什么？查看`egg`模块的代码后发现：

```js
exports.startCluster = require('egg-cluster').startCluster;
```

原来`egg.startCluster`是`egg-cluster`模块暴露的一个API

```js
// egg-cluster/index.js
const Master = require('./lib/master');
exports.startCluster = function(options, callback) {
  new Master(options).ready(callback);
};
```

可以发现`startCluster`主要做了这些事情

* 启动`master`进程
* egg启动成功后执行`callback`方法，比如希望在egg启动成功后执行一些业务上的初始化操作

### Master(egg-cluster/lib/master.js)

```js
// Master继承了events模块，拥有events监听、发送消息的能力
class Master extends EventEmitter {} 
```

### Master#constructor

`constructor`里面大致可以分为5个部分：

```js
constructor(options) {
  super();
  this.options = parseOptions(options);
  // new一个Messenger实例
  this.messenger = new Messenger(this);
  // 借用ready模块的方法
  ready.mixin(this);
  this.isProduction = isProduction();
  this.isDebug = isDebug();
  ...
  ...
  // 根据不同运行环境（local、test、prod）设置日志输出级别
  this.logger = new ConsoleLogger({ level: process.env.EGG_MASTER_LOGGER_LEVEL || 'INFO' });
  ...
}
```

```js
// master启动成功后通知parent、app worker、agent
this.ready(() => {
  this.isStarted = true;
  const stickyMsg = this.options.sticky ? ' with STICKY MODE!' : '';
  this.logger.info('[master] %s started on %s://127.0.0.1:%s (%sms)%s',
  frameworkPkg.name, this.options.https ? 'https' : 'http',
  this.options.port, Date.now() - startTime, stickyMsg);

  const action = 'egg-ready';
  this.messenger.send({ action, to: 'parent' });
  this.messenger.send({ action, to: 'app', data: this.options });
  this.messenger.send({ action, to: 'agent', data: this.options });
});
```

```js
// 监听agent退出
this.on('agent-exit', this.onAgentExit.bind(this));
// 监听agent启动
this.on('agent-start', this.onAgentStart.bind(this));
// 监听app worker退出
this.on('app-exit', this.onAppExit.bind(this));
// 监听app worker启动
this.on('app-start', this.onAppStart.bind(this));
// 开发环境下监听app worker重启
this.on('reload-worker', this.onReload.bind(this));

// 监听agent启动，注意这里只执行一次
this.once('agent-start', this.forkAppWorkers.bind(this));
```

```js
// master监听自身的退出及退出后的处理

// kill(2) Ctrl-C     监听SIGINT信号
process.once('SIGINT', this.onSignal.bind(this, 'SIGINT'));
// kill(3) Ctrl-\     监听SIGQUIT信号
process.once('SIGQUIT', this.onSignal.bind(this, 'SIGQUIT'));
// kill(15) default   监听SIGTERM信号
process.once('SIGTERM', this.onSignal.bind(this, 'SIGTERM'));

// 监听exit事件
process.once('exit', this.onExit.bind(this));
```

```js
// 监听端口冲突
detectPort((err, port) => {
  /* istanbul ignore if */
  if (err) {
    err.name = 'ClusterPortConflictError';
    err.message = '[master] try get free port error, ' + err.message;
    this.logger.error(err);
    process.exit(1);
    return;
  }
  this.options.clusterPort = port;
  this.forkAgentWorker(); // 如果端口没有冲突则执行该方法
});
```

### Master#forkAgentWorker

`master`进程以`child_process`模式启动`agent`进程

```js
forkAgentWorker() {
  this.agentStartTime = Date.now();
  const args = [ JSON.stringify(this.options) ];
  const opt = { execArgv: process.execArgv.concat([ '--debug-port=5856' ]) };
  
  // 以child_process.fork模式启动agent worker，此时agent成为master的子进程
  const agentWorker = this.agentWorker = childprocess.fork(agentWorkerFile, args, opt);
  
  // 记录agent的id
  agentWorker.id = ++this.agentWorkerIndex;
  
  this.log('[master] agent_worker#%s:%s start with clusterPort:%s',
  agentWorker.id, agentWorker.pid, this.options.clusterPort);

  // master监听从agent发送给master的消息， 并打上消息来源(msg.from = 'agent')
  // 将消息通过messenger发送出去
  agentWorker.on('message', msg => {
	if (typeof msg === 'string') msg = { action: msg, data: msg };
    msg.from = 'agent';
    this.messenger.send(msg);
  });
  
  // master监听agent的异常，并打上对应的log信息方便问题排查
  agentWorker.on('error', err => {
    err.name = 'AgentWorkerError';
    err.id = agentWorker.id;
    err.pid = agentWorker.pid;
    this.logger.error(err);
  });
  
  // master监听agent的退出
  // 并通过messenger发送agent的'agent-exit'事件给master
  // 告诉master说agent退出了
  agentWorker.once('exit', (code, signal) => {
    this.messenger.send({
      action: 'agent-exit',
      data: { code, signal },
      to: 'master',
      from: 'agent',
    });
  });
}
```

到这里，`agent worker`已完成启动，并且`master`对其进行监听，这里有个疑问
> agent启动成功后是如何通知master进行下一步操作的？

```js
const agentWorker = this.agentWorker = childprocess.fork(agentWorkerFile, args, opt);
```

以`child_process.fork`模式启动agent worker，读取的是`agent_worker.js`，截取里面的一段代码

```js
// egg-cluster/lib/agent_worker.js

agent.ready(() => {
  agent.removeListener('error', startErrorHandler);
  process.send({ action: 'agent-start', to: 'master' });
});
```

发现`agent`启动成功后调用`process.send()`通知`master`，`master`监听到该消息通过`messenger`转发出去

```js
// Master#forkAgentWorker
agentWorker.on('message', msg => {
  if (typeof msg === 'string') msg = { action: msg, data: msg };
  msg.from = 'agent';
  this.messenger.send(msg);
});
```

最终由`master`进行`agent-start`事件的响应

```js
// Master#constructor
...
...
this.on('agent-start', this.onAgentStart.bind(this));
...
this.once('agent-start', this.forkAppWorkers.bind(this));
...
...
```

### Master#onAgentStart

`agent`启动后的操作

```js
onAgentStart() {
  // agent启动成功后向app worker发送'egg-pids'事件并带上agent pid
  this.messenger.send({ action: 'egg-pids', to: 'app', data: [ this.agentWorker.pid ] });
  // 向app worker发送'agent-start'事件
  this.messenger.send({ action: 'agent-start', to: 'app' });
  this.logger.info('[master] agent_worker#%s:%s started (%sms)',
  this.agentWorker.id, this.agentWorker.pid, Date.now() - this.agentStartTime);
}
```

值得注意的是此时`app worker`还没启动，所以该消息会被丢弃，后续如果发生`agent`重启的情况会被`app worker`监听到

### Master#forkAppWorkers

`master`进程以`cluster`模式启动`app worker`进程

```js
forkAppWorkers() {
  this.appStartTime = Date.now();
  this.isAllAppWorkerStarted = false;
  this.startSuccessCount = 0;

  this.workers = new Map();

  const args = [ JSON.stringify(this.options) ];
  this.log('[master] start appWorker with args %j', args);
  
  // 以cluster模式启动app worker进程
  cfork({
    exec: appWorkerFile,
    args,
    silent: false,
    count: this.options.workers,
    // 在开发环境下不会进行refork，方便排查问题
    refork: this.isProduction,
  });

  // master监听各个app worker进程的消息
  cluster.on('fork', worker => {
    this.workers.set(worker.process.pid, worker);
    worker.on('message', msg => {
      if (typeof msg === 'string') msg = { action: msg, data: msg };
      msg.from = 'app';
      this.messenger.send(msg);
    });
    this.log('[master] app_worker#%s:%s start, state: %s, current workers: %j',
  worker.id, worker.process.pid, worker.state, Object.keys(cluster.workers));
  });
  
  // master监听各个app worker进程的disconnect事件并记录到log
  cluster.on('disconnect', worker => {
    this.logger.info('[master] app_worker#%s:%s disconnect, suicide: %s, state: %s, current workers: %j',
    worker.id, worker.process.pid, worker.exitedAfterDisconnect, worker.state, Object.keys(cluster.workers));
  });
  
  // master监听各个app worker进程的exit事件，并向master发送'app-exit'事件，将app worker退出后的事情交给master处理
  cluster.on('exit', (worker, code, signal) => {
    this.messenger.send({
      action: 'app-exit',
      data: { workerPid: worker.process.pid, code, signal },
      to: 'master',
      from: 'app',
    });
  });
  
  // master监听各个app worker进程的listening事件，表示各个app worker已经可以开始工作了
  cluster.on('listening', (worker, address) => {
    this.messenger.send({
      action: 'app-start',
      data: { workerPid: worker.process.pid, address },
      to: 'master',
      from: 'app',
    });
  });
}
```

`app worker`启动后，跟`agent`一样，通过`messenger`发`app-start`事件发送给`master`，由`master`继续处理

```js
// Master#constructor

...
...
this.on('app-start', this.onAppStart.bind(this));
...
...
```

### Master#onAppStart

`app worker`启动后的操作

```js
onAppStart(data) {
  const worker = this.workers.get(data.workerPid);
  
  ...
  
  // app worker启动成功后通知agent
  this.messenger.send({
   action: 'egg-pids',
   to: 'agent',
   data: getListeningWorker(this.workers),
  });
  
  ...
  
  // app worker准备好了
  if (this.options.sticky) {
   this.startMasterSocketServer(err => {
     if (err) return this.ready(err);
     this.ready(true);
   });
  } else {
   this.ready(true);
  }
}
```

这时`agent`和各个`app worker`已经ready了，`master`也可以做好准备了，执行`ready`后的操作，把`egg-ready`事件发送给`parent`、`app`、`agent`，告诉它们已经`ready`了，可以开始干活

```js
this.ready(() => {
  ...
  const action = 'egg-ready';
  this.messenger.send({ action, to: 'parent' });
  this.messenger.send({ action, to: 'app', data: this.options });
  this.messenger.send({ action, to: 'agent', data: this.options });
});
```

### Master#onAgentExit

`agent`退出后的处理

```js
onAgentExit(data) {
  ...
  // 告诉各个app worker，agent退出了
  this.messenger.send({ action: 'egg-pids', to: 'app', data: [] });
  
  ...
  // 记录异常信息，方便问题排查
  const err = new Error(util.format('[master] agent_worker#%s:%s died (code: %s, signal: %s)',
      agentWorker.id, agentWorker.pid, data.code, data.signal));
    err.name = 'AgentWorkerDiedError';
  this.logger.error(err);
  
  // 移除事件监听，防止内存泄露
  agentWorker.removeAllListeners();
  
  ...
  // 把'agent-worker-died'通知parent进程后重启agent进程
  this.log('[master] try to start a new agent_worker after 1s ...');
  setTimeout(() => {
    this.logger.info('[master] new agent_worker starting...');
    this.forkAgentWorker();
  }, 1000);
  this.messenger.send({
    action: 'agent-worker-died',
    to: 'parent',
  });
}
```

### Master#onAppExit

`app worker`退出后的处理

```js
onAppExit(data) {
  ...
  // 记录异常信息，方便问题排查
  if (!worker.isDevReload) {
    const signal = data.code;
    const message = util.format(
   '[master] app_worker#%s:%s died (code: %s, signal: %s, suicide: %s, state: %s), current workers: %j',
   worker.id, worker.process.pid, worker.process.exitCode, signal,
   worker.exitedAfterDisconnect, worker.state,
    Object.keys(cluster.workers)
    );
    const err = new Error(message);
    err.name = 'AppWorkerDiedError';
    this.logger.error(err);
  }
  
  // 移除事件监听，防止内存泄露
  worker.removeAllListeners();
  this.workers.delete(data.workerPid);
  
  // 发送'egg-pids'事件给agent，告诉它目前处于alive状态的app worker pid
  this.messenger.send({ action: 'egg-pids', to: 'agent', data: getListeningWorker(this.workers) });
  
  // 发送'app-worker-died'的消息给parent进程
  this.messenger.send({
    action: 'app-worker-died',
    to: 'parent',
  });
}
```

### Master#onReload

**开发**模式下监听文件的改动，对`app worker`进行重启操作

* 开发模式下开启`egg-development`插件，对相关文件进行监听，监听到有文件改动的话向`master`发送`reload-worker`事件

```js
process.send({
  to: 'master',
  action: 'reload-worker',
});
```

* `master`通过监听`reload-worker`事件后执行`onReload`方法

```js
this.on('reload-worker', this.onReload.bind(this));
```

* `onReload`通过`cluster-reload`模块进行重启操作

```js
onReload() {
  this.log('[master] reload workers...');
  for (const id in cluster.workers) {
    const worker = cluster.workers[id];
    worker.isDevReload = true;
  }
  require('cluster-reload')(this.options.workers);
}
```

### Master#onExit

`master`退出后的处理，该方法主要是打相关的log

### Master#onSignal和Master#close

测试的时候，`master`对收到的各个系统`signal`进行响应

```js
// kill(2) Ctrl-C
process.once('SIGINT', this.onSignal.bind(this, 'SIGINT'));
// kill(3) Ctrl-\
process.once('SIGQUIT', this.onSignal.bind(this, 'SIGQUIT'));
// kill(15) default
process.once('SIGTERM', this.onSignal.bind(this, 'SIGTERM'));
```

* 杀死各个`app worker`进程
* 杀死`agent`进程
* 退出`master`进程

```js
close() {
  this.closed = true;
  this.killAppWorkers();
  this.killAgentWorker();
  this.log('[master] close done, exiting with code:0');
  process.exit(0);
}
```