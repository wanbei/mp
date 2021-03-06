# 打造前端监控系统

## 为什么需要监控
在讨论为什么需要监控之前，先来看一下监控一词的含义。监控是指监测并进行控制，根据词义我们很容易得出两个问题点，一是监测什么，二是控制什么？

一个软件生命周期主要包括如下几步：需求分析、软件设计、程序编码、软件测试、运行维护。其中运行维护是持续时间最长的，也是开发者最为关注的，因为在软件投入使用后会出现各种原因导致的异常，这样就需要进行纠错性维护。所以我们需要监测软件的运行状态，让软件运行状态变得更加可控。

## 监控分类
对于网站的监控可以大致分为两类：后端监控和前端监控。后端监控主要包括：服务器性能监测（CPU、内存、IO）、网络性能监测等。前端监控主要包括：JS 错误监控、API 调用监控等。

## 前端监控功能点概述
目前所实现的前端监控功能点包括：JS 错误信息监控、DOM元素监控。

JS 错误信息监控是指我们通过收集 JS 在客户端运行时的报错，之后对数据进行分析，用于排查一些异常状况。

DOM 元素监控是指通过开发者录入监控元素规则，之后由 JS 根据规则在页面中监测元素，并生成页面截图。

为什么会有 DOM 元素监控此项功能，这需要分析后端监控的能力，后端监控主要是通过抓取页面进行页面字符分析，仅仅可以监测 SSR 类型的页面，对于 CSR 类型的页面就无能为力了。

## 前端监控功整体设计

明确了要收集的信息之后，我们就要对整体进行设计了。比如：如何管理监控系统的接入、接入者如何使用这些功能，以及上述功能如何实现。

![整体设计](https://img12.360buyimg.com/imagetools/jfs/t1/87760/30/16668/104589/5e7eec1aE05ae2c28/2809d80504fd8974.png)

整体设计服务于系统拆解，这样就把功能点进行了解耦操作，便于日后的维护管理工作。



整体流程如下：

1. 使用者发出申请，审核通过，分配 sid 唯一标识（关键）。
2. 根据 sid 生成 JS 错误信息收集所用的 JS-SDK。
3. 接入者按照规则录入DOM元素监控规则。
4. Worker 生成等待处理的规则。
5. Worker 获取等待处理规则，进行处理。

根据上面的流程对系统进行了层的划分，如下图所示：

![层级划分](https://img10.360buyimg.com/imagetools/jfs/t1/107197/32/10692/125219/5e7f1e0cE439a06f4/b6c28dab678f18d6.png)


## 功能点分解

### JS 错误信息收集

#### JS-SDK
在 JS 中收集错误可用 try,catch 和 window.onerror，在进行选择时需要对此进行分析对比，已选择适合系统的方案。

##### try,catch

![try-catch](https://img13.360buyimg.com/imagetools/jfs/t1/93313/26/16700/100662/5e7f1e51Edf786973/8e22fc3937a2db41.png)

try,catch能够知道出错的信息，并且也有堆栈信息可以知道在哪个文件第几行第几列发生错误。

但是try,catch的方案有2个缺点：
1. 没法捕捉try,catch块，当前代码块有语法错误，JS解释器压根都不会执行当前这个代码块，所以也就没办法被catch住；

2. 没法捕捉到全局的错误事件，也即是只有try,catch的块里边运行出错才会被你捕捉到，这里的块你要理解成一个函数块。

关于第一个缺点，我们没有任何解决办法，原因上边说了，但是一般语法阶段我们是能在开发阶段/或者用工具检测到的，于是乎它就被忽略了。

第二个缺点应该怎么理解呢？try, catch只能捕捉到当前执行流里边的运行错误，对于异步回调来说，是不属于这个try,catch块的，我们可以改进这个方案，在功能打包构建时往文件块和 function块加入 try,catch。但这种方案对于 JS-SDK 来说并不现实，因为我们没办法让所有接入者都做这种操作。

##### window.onerror

![window.onerror](https://img10.360buyimg.com/imagetools/jfs/t1/105866/9/16768/237478/5e7f1e40E281e6345/54363be274c09a47.png)

window.onerror一样可以拿到出错的信息以及文件名、行号、列号，还可以在window.onerror最后return true让浏览器不输出错误信息到控制台。

window.onerror能捕捉到语法错误，但是语法出错的代码块不能跟window.onerror在同一个块（语法都没过，更别提window.onerror会被执行了）。

只要把window.onerror这个代码块分离出去，并且比其他脚本先执行（注意这个前提！）即可捕捉到语法错误。

对于跨域的JS资源，window.onerror拿不到详细的信息，需要往资源的请求添加额外的头部。

静态资源请求需要加多一个Access-Control-Allow-Origin头部，同时script引入外链的标签需要加多一个crossorigin的属性。经过这样折腾后一样能获取到准确的出错信息。

#### JS-SDK 服务端设计
JS-SDK 服务端技术选择了 Nginx+Lua+Redis，首先将上报错误数据通过 Lua 存储到 Redis，之后通过 Woker 方式对 Redis 数据排重落库。Woker 方式采用 Linux 定时任务管理，根据需要在服务器进行配置。

![流程图](https://img10.360buyimg.com/imagetools/jfs/t1/85693/40/16668/79683/5e7edfc7E9001e009/599787553ec1b60d.png)

那么为什么采用这种技术，考虑点是什么呢？

从业务角度出发，设计之初面向 PC 首页、商品详情页和搜索列表页，所以需要参照这些页面的流量情况。

从监控功能角度出发，JS 错误信息仅仅起到收集作用，对于报警功能暂无要求，时效性可容忍稍稍延迟，通过 Worker 调节处理速度，且 Redis 存储的是热数据，后期增加其他处理，对于时效没有影响，且处理速度快。功能扩展仅需在 Woker 增加功能。

数据库仅仅起到回看历史数据，数据分析等功能。

Nginx的优点：

1. Nginx是采用异步非阻塞的方式去处理请求的。
2. 轻量级，起动同样的 web 应用服务，Nginx 比 Apache 占用更少内存及资源。
3. 抗并发，Nginx 处理请求异步非阻塞而 Apache 则阻塞，高并发下 Nginx 能保持低资源低消耗高性能。


Lua的特点：

Lua由标准C编写而成，几乎在所有操作系统和平台上都可以编译，在目前所有脚本引擎中，Lua的速度是最快的。


Redis特点：

Redis 是一个高性能的key-value数据库。性能极高 – Redis能读的速度是110000次/s,写的速度是81000次/s 。

数据库设计：

由于 PC 首页、商品详情页、搜索列表页流量大，监控所采集的数据也都来自此，所以数据量增长速度快，于是采用了分库分表设计。

根据所面向系统进行分表，目前只是简单地分表方式即通过 sid 编码进行分表。

![分表图](https://img14.360buyimg.com/imagetools/jfs/t1/84816/21/16747/47346/5e7eebfeEcb1934ff/be26e5ab1ef1b9f3.png)

### DOM元素监控
DOM 元素监控所要实现的功能是插入指定测试用例，并对界面生成截图。这里选择使用 PhantomJS。

PhantomJS 已经停止维护了，可以采用 Chrome Headless 进行替换。

PhantomJS是一个可以用JavaScript编写脚本的无头web浏览器。它可以在Windows、macOS、Linux和FreeBSD上运行。

它使用QtWebKit作为后端，支持各种web标准(DOM处理、CSS选择器、JSON、Canvas和SVG)。

这里之所以选择 PhantomJS 主要考虑到可用 JavaScript 编写脚本，相比于 Selenium 更加轻量级，仅需安装 PhantomJS 软件即可，且部分测试同学也会用它做自动化测试。


![流程图](https://img10.360buyimg.com/imagetools/jfs/t1/108819/13/10556/163563/5e7ef176E5577460a/e5b224234eff3f2d.png)

#### PhantomJS Server 实现

在实现之前需要看一下功能，监测 DOM 元素需要编写规则，规则这块采用了 css 选择器，根据 css 选择器选出的元素或元素集合可进行个数、属性、文本内容的校验。

![选择器](https://img10.360buyimg.com/imagetools/jfs/t1/107776/5/10523/252663/5e7efba8Eb4a84259/768409619c6b2c9a.png)

![所处理属性](https://img13.360buyimg.com/imagetools/jfs/t1/107097/16/16776/67424/5e7efba8E170c75af/08564129aabbe5d3.png)

PhantomJS 服务本质上是一个 Node.js 的应用，通过 PM2 启动应用程序，之后会根据请求进来的路径去匹配需要出发的任务，通过 Node.js child_process 启动 shell 子进程，根据子进程的执行返回数据给 Reuqester。

![流程图](https://img13.360buyimg.com/imagetools/jfs/t1/94333/18/16679/47168/5e7ef624Ec4c51a5a/82e7abcd7c82a229.png)

##### 截图功能
截图的话我们一开始在本地测试保留了页面 img 所使用的图片，上线前关闭了 img 图片，因为带有 img 的截图图片太大，影响上传速度。

对于异步请求页面以及全屏截图的处理，由于设计之初只针对 PC 页面，PC 页面的楼层使用的是懒加载技术，为了处理这项限制，只需要调整浏览器视口便可将数据一次性加载完同时获得完整页面截图。



由于受限于 PhantomJS render API ，每次启动都会进行页面截图并上传，因为没办法等到监测出异常在截图，这样正常情况也会生成页面截图。

在初期阶段，会发现机器经常报警，CUP 利用率 100%，经过排查发现通过子进程启动的 PhantoJS 存在假死现象，无法正常终止，只能通过 Shell 强行关闭。

## CDN Docker

PhantomJS Server 本质上是一个 Node.js 的应用，我们可以通过 Npm 打包进行分发，但是受限于公司CDN的限制，不能通过 Npm 进行分发，只能通过 Docker 的方式进行分发。所以 PhantomJS Server 的基础上增加了 Docker 支持，使其可以部署在 CDN 节点上，更真是的模拟用户所处的网络环境。

CDN 节点部署可以避免由于 618、 11.11 活动期间网络频繁调整导致的网络波动对监控功能的影响。这也是部署 CDN 节点的目的。

## 结束语

本次并没有贴出大量代码，主要是因为在系统划分的各个层上面都有与之相对的技术实现，针对各个层的特性来选择技术实现。在了解整体以及设计思路之后，自己编码实现这些功能是相对容易的。

