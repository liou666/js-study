### 进程与线程

**进程**：系统进行资源分配和调度的基本单位，是操作系统结构的基础。（例如：启动一个服务，打开一个qq软件，浏览器的每个tab栏都是一个单独的进程。）

**线程**：操作系统能够进行运算调度的最小单位，**线程是隶属于进程，被包含于进程之中**。一个线程只能隶属于一个进程，但是一个进程是可以拥有多个线程的。（例如：浏览器tab栏进程中有主线程、渲染线程、垃圾回收线程等。）

**对比**
名称|数据共享|资源消耗|稳定性
--  |:--:   |:--:    | :--:    
进程|**困难**（每个进程都拥有自己的独立空间地址、数据栈；默认情况下不能互相访问数据，只有建立了 IPC 通信，进程之间才可数据共享）|创建、销毁、切换时**系统开销较大**|进程间不会相互影响|
线程|**容易**（线程之间共享进程数据）| 创建、销毁、切换时**系统开销较小**| 一个线程挂掉将导致整个进程挂掉|

### 单线程

>**`javascript`是单线程，即进程中只有一个线程，同一时间内只能做一件事情**。

```js
const http=require("http");

function longComputation(){
    console.time('计算耗时');
    let sum = 0;
    for (let i = 0; i < 1e10; i++) {
      sum += i;
    };
    console.timeEnd('计算耗时');
    return sum;
}


http.createServer((req,res)=>{
    if(req.url==="/sum"){
        longComputation()
        res.end("sum")
    }
    res.end("succeed")
}).listen(3000)
```

上例用`node`开启一个本地服务，（也就是一个进程），由于`javascript`是单线程，所以在访问`http://localhost:3000/sum`时，**由于其中有耗时的计算，之后的任何请求都会被阻塞**，等待`longComputation`计算完成后才能被执行。

### node多进程
`node`默认可以基于`child_process`模块创建多进程。

`child_process`模块暴露了`spawn`,`fork`,`exec`,`execFile`四个方法。其中`fork`,`exec`,`execFile`都是基于`spawn`方法派生出来的。

#### 基本用法

`spawn(command[, args][, options])`
```js
//child.js
console.log("child")

//father.js
spawn("node",["child"]);

```
**上例中执行`father.js`不会有任何输出，因为子进程有自己独立的`stdio`**，父子进程共享`stdio`的方式有很多。如下：

```js
//child.js
console.log("child")

//father.js
const cp= spawn("node",["child.js"]);


1. cp.stdout.pipe(process.stdout);//steam流的方式，将子进程输出流向父进程

2. cp.stdout.on("data",chunk=>{ process.stdout.write(chunk.toString())})//监听data的方式，向父进程输出流写入子进程的输出流


/**
 * 也可以通过spawn的第三个参数options的stdio选项控制
 *options.stdio 选项用于配置在父进程和子进程之间建立的管道
 *为方便起见，options.stdio 可能是以下字符串之一：
 *'pipe': 相当于 ['pipe', 'pipe', 'pipe']（默认）
 *'overlapped': 相当于 ['overlapped', 'overlapped', 'overlapped']  与 'pipe' 相同，只是在句柄上设置了 FILE_FLAG_OVERLAPPED 标志
 *'ignore': 相当于 ['ignore', 'ignore', 'ignore']
 *'inherit': 相当于 ['inherit', 'inherit', 'inherit'] 或 [0, 1, 2] 通过相应的标准输入输出流传入/传出父进程
*/

```

#### 进程通讯
>**进程之间的通讯主要是通过IPC实现**

```js
//child.js
console.log("child");
process.on("message",(data)=>{
    console.log(data,"child pid:",process.pid);
    process.send("hello")
})


//father.js
const cp= spawn("node",["child.js"],{
    stdio:[0,1,2,"ipc"]
});
//如果没有开启ipc，执行cp.send方法会报错
cp.send("hi")

cp.on("message",(data)=>{
    console.log(data);
})

//执行father.js控制台输出
//child
//hi child pid: 31983
//hello

```
>默认情况下，父进程退出后，子进程也会自动退出。如果要让子进程独立于父进程运行，可以使用` options.detached`选项;

```js
//child.js (每一秒向text.txt中追加aaa)
const fs=require("fs")
setInterval(()=>{
    fs.appendFileSync("text.txt","aaa\n")
},1000)

//father.js 
const cp= spawn("node",["child.js"],{
   stdio:"ignore",
   detached:true 
});
cp.unref();

```
执行`father.js` 文件后，主进程退出，通过`get-process -Name node`查看`node`进程 ，子进程依然运行。

`fork`,`exec`,`execFile`都是基于`spawn`方法实现的，主要区别如下：

+ `fork`方法**默认开启`IPC`，默认指定为`node`命令**,`fork("test.js")`相当于 `spawn("node",["test.js"],{stdio:[0,1,2,"pic"],shell: false})`

+ `exec`方法会**缓冲命令生成的输出，并传递整个输出值给一个回调函数**。而不是使用流的方式。**但是该方法会创建一个`shell`去执行命令，所以效率不高**。

+ `execFile`方法和`exec`方法类似，但是**不会开启`shell`去执行命令**。

### master-worker 模式
`Node.js` 本身就使用的事件驱动模型，为了解决单进程单线程对多核使用不足问题，可以按照 `CPU` 数目**多进程启动**，理想情况下一个每个进程利用一个 `CPU`。

`Node.js` 提供了 `child_process` 模块支持多进程，通常使用 `child_process.fork(modulePath)` 方法可以调用指定模块，衍生新的`Node.js` 进程 `worker.js`。使主进程负责调度和管理工作进程，工作进程负责具体业务逻辑处理。

```js
//worker.js 启动web服务在不同的端口
const http = require('http');

const childServer = http.createServer((req, res) => {
    res.writeHead(200, {
        'Content-Type': 'text/plain'
    });
    res.end("child worker run"+', at pid: ' + process.pid);
});


const randomPort=parseInt(Math.random() * 10000);

childServer.listen(randomPort);

process.on('message', msg => {
    console.log(`worker get message: ${msg}`);
});
  
process.send(`${randomPort} ready`);
```
执行master.js,根据cpu的核数生成对应的服务数量
```js
//master.js
const {fork} =require("child_process");

const cpus=require("os").cpus()

for (let i = 0, len = cpus.length; i < len; i++) {
   const cp= fork('worker.js')
   cp.on("message",(data)=>console.log(data))
}

```

控制台输出结果:
`1621 ready;
8089 ready;
2572 ready;
5756 ready`
由于自己电脑为4核cpu，所以这里生成了4个对应的服务。

#### 句柄传递

在上面的例子中每个工作进程都使用了一个随机端口，**如果设置成一样的会出现端口号被占用的错误**。这个问题可以通过 `master` 监听 80 端口，分发请求给工作进程，工作进程使用不同的端口号解决。
```js
const {spawn,exec,fork} =require("child_process");
const cpus=require("os").cpus()
const http = require('http');

const server=http.createServer()
const workers=[]

//收集wroker
for (let i = 0, len = cpus.length; i < len; i++) {
   const worker= fork('./3.js');
   workers.push(worker)
}

server.listen(80,()=>{
    //master服务启动后将服务转发到子进程
    //这样在访问80端口时会自动分发到空闲的子进程服务中，也就实现了负载均衡
    workers.forEach(worker=>{
        worker.send("server",server)
    })
})

```

```js
//worker.js 启动web服务在不同的端口
const http = require('http');
const childServer = http.createServer((req, res) => {
    res.writeHead(200, {
        'Content-Type': 'text/plain'
    });
    res.end("child worker run"+', at pid: ' + process.pid);
});

process.on('message', (type, server) => {
     if (type==='server') {
         childServer.listen(server)//监听master进程中的服务
     }
})

```
#### 多个进程监听相同端口的原理
`Node.js` 对每个端口监听的时候设置了 `SO_REUSEADDR` 选项，允许不同的进程对相同的端口号监听

>setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

独立启动的进程服务器 `socket` 的**文件描述符`（listenfd）`不同**，所以监听相同的端口号会失败，而上面代码 `socket` 都使用 `master` 发送的 `socket`，所以可以监听成功。**多个应用监听相同的端口号时文件描述符同一时间只能被一个进程占用**，也就是说网络请求向服务器发送的时候只有一个进程可以抢占到对请求提供服务


#### 子进程出错时的处理方式

在上例中，虽然完成了服务的负载均衡，子进程出错后，该子进程会立即退出。所以为了服务的稳定性，应该在子进程出错退出后自动生成新的子进程。具体实现方式就是在**创建子进程时增加监听子进程的退出事件**。

```js
const {fork} =require("child_process");
const cpus=require("os").cpus()
const http = require('http');


const server = http.createServer();
const workers = {};

const createWorker=(server)=>{
    const worker = fork('worker.js'); 
    workers[worker.pid]=worker;

    console.log(`worker stady ${worker.pid}`);

    worker.send('server', server);
   //监听子进程退出，意外退出后自动生成新的子进程
    worker.on("exit",(code,signal)=>{
        console.log(`worker ${worker.pid} is died`,code,signal);
        Reflect.deleteProperty(workers,worker.pid)
        createWorker(server)
    })
}

server.listen(80);

for (let i = 0, len = cpus.length; i < len; i++) {
    createWorker(server)
}
```

#### cluster模块

`node.js`内置了`cluster`模块，可以方便的完成上述操作。用`cluster`模块实现`master-wroker`模式，如下:

```js
const cluster = require('cluster');
const http = require('http');
const numCPUs = require('os').cpus().length;
if (cluster.isPrimary) {
  console.log(`主进程 ${process.pid} 正在运行`);

  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

    //监听子进程的退出，意外推出后自动生成新的子进程
  cluster.on('exit', (worker, code, signal) => {
    console.log(`工作进程 ${worker.process.pid} 已退出`);
    cluster.fork();
  });
} else {
  http.createServer((req, res) => {
    res.writeHead(200);
    res.end(process.pid+'\n');
  }).listen(80);
  console.log(`工作进程 ${process.pid} 已启动`);
}

```
+ `cluster.isPrimary`的真假是由`process.env.NODE_UNIQUE_ID` 决定的。 如果` process.env.NODE_UNIQUE_ID` 未定义，则 `cluster.isPrimary` 为 `true`。
<br/>

+ `cluster.fork`方法本质上就是`fork("self.js")`,fork当前的文件，第一次执行时`process.env.NODE_UNIQUE_ID`未定义，`cluster.isPrimary`为`true`,接着将`process.env.NODE_UNIQUE_ID`赋予为一个独一无二的值,再次`fork`时，` cluster.isPrimary`为`false`。


































