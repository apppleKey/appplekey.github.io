# 手搓一个TinyPng压缩图片的WebpackPlugin

曾经发表过一篇性能优化的文章[《前端性能优化指南》](https://juejin.im/post/6844903844455907336)，笔者总结了一些在项目开发过程中使用过的性能优化经验。说句真话，性能优化可能在面试过程中会有用，实际在项目开发过程中可能没几个同学会注意这些性能优化的细节。

若经常关注性能优化的话题，可能会发现`无论怎样对代码做最好的优化也不及对一张图片做一次压缩好`。所以压缩图片成了性能优化里最常见的操作，不管是手动压缩图片还是自动压缩图片，在项目开发过程中必须得有。

自动压缩图片通常在`webpack`构建项目时接入一些第三方`Loader&Plugin`来处理。打开`Github`，搜素`webpack image`等关键字，Star最多还是`image-webpack-loader`和`imagemin-webpack-plugin`这两个`Loader&Plugin`。很多同学可能都会选择它们，方便快捷，简单易用，无脑接入。

可是，这两个`Loader&Plugin`存在一些特别问题，它们都是基于`imagemin`开发的。`imagemin`的某些依赖托管在国外服务器，在`npm i xxx`安装它们时会默认走`GitHub Releases`的托管地址，若不是规范上网，你们是不可能安装得上的，即使规范上网也不一定安装得上。所以笔者又刨根到底发表了一篇关于NPM镜像处理的文章[《聊聊NPM镜像那些险象环生的坑》](https://juejin.im/post/6844904192247595022)，专门解决这些因为网络环境而导致安装失败的问题。除了这个安装问题，`imagemin`还存在另一个大问题，就是压缩质感损失得比较严重，图片体积越大越明显，压缩出来的图片总有几张是失真的，而且总体压缩率不是很高。这样在交付项目时有可能被细心的QA小姐姐抓个正着，怎么和设计图对比起来不清晰啊！

### 工具

##### 图片压缩工具

此时可能有些同学已转战到手动压缩图片了。比较好用的图片压缩工具无非就是以下几个，若有更好用的工具麻烦在评论里补充喔！同时笔者也整理出它们的区别，供各位同学参考。

![工具集合](https:////p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a7b0f78bf8244d93bf683d2b845fee5f~tplv-k3u1fbpfcp-zoom-1.image)

| 工具                                      | 开源 | 收费 | API  | 免费体验                                             |
| ----------------------------------------- | ---- | ---- | ---- | ---------------------------------------------------- |
| [QuickPicture](https://www.tuhaokuai.com) | ✖️    | ✔️    | ✖️    | 可压缩类型较多，压缩质感较好，有体积限制，有数量限制 |
| [ShrinkMe](https://shrinkme.app)          | ✖️    | ✖️    | ✖️    | 可压缩类型较多，压缩质感一般，无数量限制，有体积限制 |
| [Squoosh](https://squoosh.app)            | ✔️    | ✖️    | ✔️    | 可压缩类型较少，压缩质感一般，无数量限制，有体积限制 |
| [TinyJpg](https://tinyjpg.com)            | ✖️    | ✔️    | ✔️    | 可压缩类型较少，压缩质感很好，有数量限制，有体积限制 |
| [TinyPng](https://tinypng.com)            | ✖️    | ✔️    | ✔️    | 可压缩类型较少，压缩质感很好，有数量限制，有体积限制 |
| [Zhitu](https://zhitu.isux.us)            | ✖️    | ✖️    | ✖️    | 可压缩类型一般，压缩质感一般，有数量限制，有体积限制 |

从上述表格对比可看出，免费体验都会存在`体积限制`，这个可理解，即使收费也一样，毕竟每个人都上传单张10多M的图片，哪个服务器能受得了。再来就是`数量限制`，一次只能上传20张，好像有个规律，压缩质感好就限制数量，否则就不限制数量，当然收费后就没有限制了。再来就是`可压缩类型`，图片类型一般是`jpg`、`png`、`gif`、`svg`和`webp`，`gif`压缩后一般都会失真，`svg`通常用在矢量图标上很少用在场景图片上，`webp`由于兼容性问题很少被使用，故能压缩`jpg`和`png`就足够了。当然`压缩质感`是最优考虑，综上所述，大部分同学都会选择**TinyJpg**和**TinyPng**，其实它俩就是兄弟，出自同一厂商。

在笔者公众号的微信讨论群里发起了一个简单的投票，最终还是**TinyJpg**和**TinyPng**胜出。

![工具投票](https://yangzw.vip/static/article/webpack-plugin/tool-vote.jpg)

##### TinyJpg/TinyPng存在问题

- 上传下载全靠`手动`
- 只能压缩`jpg`和`png`
- 每次只能压缩`20`张
- 每张体积最大不能超过`5M`
- 可视化处理信息不是特别齐全

##### TinyJpg/TinyPng压缩原理

**TinyJpg/TinyPng**使用智能有损压缩技术将图片体积降低，选择性地减少图片中相似颜色，只需很少字节就能保存数据。对视觉影响几乎不可见，但是在文件体积上就有很大的差别。而使用到`智能有损压缩技术`被称为**量化**。

**TinyJpg/TinyPng**在压缩`png文件`时效果更显著。扫描图片中相似颜色并将其合并，通过减少颜色数量将`24位png文件`转换成体积更小的`8位png文件`，丢弃所有不必要的元数据。

大部分`png文件`都有`50%~70%`的压缩率，即使视力再好也很难区分出来。使用优化过的图片可减少带宽流量和加载时间，整个网站使用到的图片经**TinyJpg/TinyPng**压缩一遍，其成效是再多的代码优化也无法追赶得上的。

![熊猫](https:////p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a516240b28ce49899fb40799950aac9a~tplv-k3u1fbpfcp-zoom-1.image)

##### TinyJpg/TinyPng开发API

查阅相关资料，发现**TinyJpg/TinyPng**暂时还未开源其压缩算法，不过提供了适合开发者使用的API。有兴趣的同学可到其[开发API文档](https://tinypng.com/developers)瞧瞧。

在`Node`方面，**TinyJpg/TinyPng**官方提供了[tinify](https://github.com/tinify/tinify-nodejs)作为压缩图片的核心JS库，使用很简单，看[文档](https://tinypng.com/developers/reference/nodejs)吧。可是换成开发API还是逃不过收费，你是想包月呢还是免费呢，想免费的话就继续往下看，土豪随意！

![图片压缩工具](https://yangzw.vip/static/article/webpack-plugin/tool-price.png)

### 实现

笔者也是经常使用**TinyJpg/TinyPng**的程序猿，收费，那是不可能的😂。寻找突破口，解决问题，是作为一位程序猿最基本的素养。`我们需明确什么问题，需解决什么问题`。

##### 分析

从上述得知，只需对**TinyJpg/TinyPng**原有功能改造成以下功能。

- 上传下载全自动
- 可压缩`jpg`和`png`
- 没有数量限制
- 存在体积限制，最大体积不能超过`5M`
- 压缩成功与否输出详细信息

> 自动处理

对于前端开发者来说，这种无脑的上传下载操作必须得自动化，省事省心省力。但是这个操作得结合`webpack`来处理，到底是开发成`Loader`还是`Plugin`，后面再分析。不过细心的同学看标题就知道用什么方式处理了。

> 压缩类型

`gif`压缩后一般都会失真，`svg`通常用在矢量图标上很少用在场景图片上，`webp`由于兼容性问题很少被使用，故能压缩`jpg`和`png`就足够了。在过滤图片时，使用`path模块`判断文件类型是否为`jpg`和`png`，是则继续处理，否则不处理。

> 数量限制

数量限制当然是不能存在的，万一项目里超过20张图片，那不是得分批处理，这个不能有。对于这种无需登录状态就能处理一些用户文件的网站，通常都会通过IP来限制用户的操作次数。有些同学可能会说，刷新页面不就行了吗，每次压缩20张图片，再刷新再压缩，万一有500张图片呢，你就刷新25次吗，这样很好玩是吧！

由于大多数Web架构很少会将应用服务器直接对外提供服务，一般都会设置一层`Nginx`作为代理和负载均衡，有的甚至可能有多层代理。鉴于大多数Web架构都是使用`Nginx`作为反向代理，用户请求不是直接请求应用服务器的，而是通过Nginx设置的统一接入层将用户请求转发到服务器的，所以可通过设置HTTP请求头字段`X-Forwarded-For`来伪造IP。

**X-Forwarded-For**指用来识别通过`代理`或`负载均衡`的方式连接到Web服务器的客户端最原始的IP地址的HTTP请求头字段。当然，这个IP也不是一成不变的，每次请求都需随机更换IP，骗过应用服务器。若应用服务器增加了伪造IP识别，那可能就无法继续使用随机IP了。

> 体积限制

体积限制这个能理解，也没必要搞一张那么大的图片，多浪费带宽流量和加载时间啊。在上传图片时，使用`fs模块`判断文件体积是否超过`5M`，是则不上传，否则继续上传。当然，交给**TinyJpg/TinyPng**接口判断也行。

> 输出信息

压缩成功与否得让别人知道，输出原始大小、压缩大小、压缩率和错误提示等，让别人清楚这些处理信息。

##### 编码

通过上述抽丝剥茧的分析，那么就开始着手编码了。

> 随机生成HTTP请求头

既然可通过`X-Forwarded-For`来伪造IP，那么得有一个随机生成HTTP请求头字段的函数，每次请求接口时都随机生成相关的请求头字段。打开[tinyjpg.com](https://tinyjpg.com)或[tinypng.com](https://tinypng.com)上传一张图片，通过`Chrome DevTools`分析`Network`发现其请求接口是`web/shrink`。另外每次请求也不要集中在单一的`hostname`上，随机派发到`tinyjpg.com`或`tinypng.com`上会更好。通过封装`RandomHeader`函数随机生成请求头信息，后续使用`https模块`以`RandomHeader()`生成的配置作为入参进行请求。

`trample`是笔者开发的一个`Web/Node`通用函数工具库，包含常规的工具函数，助你少写更多通用代码。详情请查看[文档](https://github.com/JowayYoung/trample)，顺便给一个Star以作鼓励。

![工具接口](https:////p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/760f2dda6b034554b9a7c32c3765b659~tplv-k3u1fbpfcp-zoom-1.image)

```js
const { RandomNum } = require("trample/node");

const TINYIMG_URL = [
    "tinyjpg.com",
    "tinypng.com"
];

function RandomHeader() {
    const ip = new Array(4).fill(0).map(() => parseInt(Math.random() * 255)).join(".");
    const index = RandomNum(0, 1);
    return {
        headers: {
            "Cache-Control": "no-cache",
            "Content-Type": "application/x-www-form-urlencoded",
            "Postman-Token": Date.now(),
            "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.116 Safari/537.36",
            "X-Forwarded-For": ip
        },
        hostname: TINYIMG_URL[index],
        method: "POST",
        path: "/web/shrink",
        rejectUnauthorized: false
    };
}
复制代码
```

> 上传图片与下载图片

使用`Promise`封装`上传图片`和`下载图片`的函数，方便后续使用`Async/Await`同步化异步代码。以下函数的具体断点调试就不说了，有兴趣的同学自行调试函数的入参和出参哈！

```js
const Https = require("https");
const Url = require("url");

function UploadImg(file) {
    const opts = RandomHeader();
    return new Promise((resolve, reject) => {
        const req = Https.request(opts, res => res.on("data", data => {
            const obj = JSON.parse(data.toString());
            obj.error ? reject(obj.message) : resolve(obj);
        }));
        req.write(file, "binary");
        req.on("error", e => reject(e));
        req.end();
    });
}

function DownloadImg(url) {
    const opts = new Url.URL(url);
    return new Promise((resolve, reject) => {
        const req = Https.request(opts, res => {
            let file = "";
            res.setEncoding("binary");
            res.on("data", chunk => file += chunk);
            res.on("end", () => resolve(file));
        });
        req.on("error", e => reject(e));
        req.end();
    });
}
复制代码
```

> 压缩图片

通过`上传图片`函数获取压缩后的图片信息，再依据图片信息通过`下载图片`函数生成本地文件。

```js
const Fs = require("fs");
const Path = require("path");
const Chalk = require("chalk");
const Figures = require("figures");
const { ByteSize, RoundNum } = require("trample/node");

async function CompressImg(path) {
    try {
        const file = Fs.readFileSync(path, "binary");
        const obj = await UploadImg(file);
        const data = await DownloadImg(obj.output.url);
        const oldSize = Chalk.redBright(ByteSize(obj.input.size));
        const newSize = Chalk.greenBright(ByteSize(obj.output.size));
        const ratio = Chalk.blueBright(RoundNum(1 - obj.output.ratio, 2, true));
        const dpath = Path.join("img", Path.basename(path));
        const msg = `${Figures.tick} Compressed [${Chalk.yellowBright(path)}] completed: Old Size ${oldSize}, New Size ${newSize}, Optimization Ratio ${ratio}`;
        Fs.writeFileSync(dpath, data, "binary");
        return Promise.resolve(msg);
    } catch (err) {
        const msg = `${Figures.cross} Compressed [${Chalk.yellowBright(path)}] failed: ${Chalk.redBright(err)}`;
        return Promise.resolve(msg);
    }
}
复制代码
```

> 压缩目标图片

完成上述步骤对应的函数后，就能自由压缩图片了，以下使用一张图片作为演示。

```js
const Ora = require("ora");

(async() => {
    const spinner = Ora("Image is compressing......").start();
    const res = await CompressImg("src/pig.png");
    spinner.stop();
    console.log(res);
})();
复制代码
```

你看，压缩完后笨猪都变帅猪了，能电眼的猪都是好猪。源码请查看[compress-img](https://github.com/JowayYoung/juejin-code/tree/master/compress-img)。

![电眼猪](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="800" height="600"></svg>)

若压缩指定文件夹里符合条件的所有图片，可通过`fs模块`获取图片并使用`map()`将单个图片路径映射为`CompressImg(path)`，再通过`Promise.all()`操作即可。在这里就不贴代码了，当作思考题，自行完成。

将上述压缩图片的功能封装成`Loader`还是`Plugin`呢？接下来会一步一步分析。

### Loader&Plugin

`webpack`是一个前端资源打包工具，它根据模块依赖关系进行静态分析，然后将这些模块按照指定规则生成对应的静态资源。

网上一大堆`webpack`教程，笔者就不再花大篇幅啰嗦了，相信各位同学都是一位标准的`Webpack配置工程师`。以下简单回顾一次`webpack`的组成、构建机制和构建流程，相信也能从这些知识点中定位出`Loader`和`Plugin`在`Webpack构建流程`中是处于一个什么样的角色地位。

```!
本文所说的webpack都是基于webpack v4
复制代码
```

> 组成

-  **Entry**：入口
-  **Output**：输出
-  **Loader**：转换器
-  **Plugin**：扩展器
-  **Mode**：模式
-  **Module**：模块
-  **Target**：目标

> 构建机制

- 通过Babel转换代码并生成单个文件依赖
- 从入口文件开始递归分析并生成依赖图谱
- 将各个引用模块打包成一个立即执行函数
- 将最终bundle文件写入`bundle.js`中

> 构建流程

- 初始
  - `初始参数`：合并命令行和配置文件的参数
- 编译
  - `执行编译`：依据参数初始`Compiler对象`，加载所有`Plugin`，执行`run()`
  - `确定入口`：依据配置文件找出所有入口文件
  - `编译模块`：依据入口文件找出所有依赖模块关系，调用所有`Loader`进行转换
  - `生成图谱`：得到每个模块转换后的内容及其之间的依赖关系
- 输出
  - `输出资源`：依据依赖关系将模块组装成块再组装成包(`module → chunk → bundle`)
  - `生成文件`：依据配置文件将确认输出的内容写入文件系统

##### Loader

**Loader**用于转换模块源码，笔者将其翻译为`转换器`。`Loader`可将所有类型文件转换为`webpack`能够处理的有效模块，然后利用`webpack`的打包能力对它们进行二次处理。

`Loader`具有以下特点：

- 单一职责原则(`只完成一种转换`)
- 转换接收内容
- 返回转换结果
- 支持链式调用

`Loader`将所有类型文件转换为应用程序的依赖图谱可直接引用的模块，所以`Loader`可用于编译一些文件，例如`pug → html`、`sass → css`、`less → css`、`es5 → es6`、`ts → js`等。

处理一个文件可使用多个`Loader`，`Loader`的执行顺序和配置顺序是相反的，即末尾`Loader`最先执行，开头`Loader`最后执行。最先执行的`Loader`接收源文件内容作为参数，其它`Loader`接收前一个执行的`Loader`的返回值作为参数，最后执行的`Loader`会返回该文件的转换结果。一句话概括：**富土康流水线厂工**。

`Loader`开发思路总结如下：

- 通过`module.exports`导出一个`函数`
- 函数第一默认参数为`source`(源文件内容)
- 在函数体中处理资源(可引入第三方模块扩展功能)
- 通过`return`返回最终转换结果(字符串形式)

```!
编写Loader时要遵循单一职责原则，每个Loader只做一种转换工作
复制代码
```

##### Plugin

**Plugin**用于扩展执行范围更广的任务，笔者将其翻译为`扩展器`。`Plugin`的范围很广，在`Webpack构建流程`里从开始到结束都能找到时机作为插入点，只要你想不到没有你做不到。所以笔者认为`Plugin`的功能比`Loader`更加强大。

`Plugin`具有以下特点：

- 监听`webpack`运行生命周期中广播的事件
- 在合适时机通过`webpack`提供的API改变输出结果
- `webpack`的Tapable事件流机制保证Plugin的有序性

在`webpack`运行生命周期中会广播出许多事件，`Plugin`可监听这些事件并在合适时机通过`webpack`提供的API改变输出结果。在`webpack`启动后，在读取配置过程中执行`new MyPlugin(opts)`初始化`自定义Plugin`获取其实例，在初始化`Compiler对象`后，通过`compiler.hooks.event.tap(PLUGIN_NAME, callback)`监听`webpack`广播事件，当捕抓到指定事件后，会通过`Compilation对象`操作相关业务逻辑。一句话概括：**自己看着办**。

`Plugin`开发思路总结如下：

- 通过`module.exports`导出一个`函数或类`
- 在`函数原型或类`上绑定`apply()`访问`Compiler对象`
- 在`apply()`中指定一个绑定到`webpack`自身的事件钩子
- 在事件钩子中通过`webpack`提供的API处理资源(可引入第三方模块扩展功能)
- 通过`webpack`提供的方法返回该资源

```!
传给每个Plugin的Compiler和Compilation都是同一个引用，若修改它们身上的属性会影响后面的Plugin，所以需谨慎操作
复制代码
```

##### Loader/Plugin区别

- 本质
  - `Loader`本质是一个函数，转换接收内容，返回转换结果
  - `Plugin`本质是一个类，监听`webpack`运行生命周期中广播的事件，在合适时机通过`webpack`提供的API改变输出结果
- 配置
  - `Loader`在`module.rule`中配置，类型是数组，每一项对应一个模块解析规则
  - `Plugin`在`plugin`中配置，类型是数组，每一项对应一个扩展器实例，参数通过构造函数传入

### 封装

##### 分析

从上述可知`Loader`和`Plugin`在角色定位和执行机制上有很多不一样，到底如何选择呢？各有各好，当然还是需分析后进行选择。

`Loader`在`webpack`中扮演着转换器的角色，用于转换模块源码，简单理解就是将文件转换成另外形式的文件，而本文主题是`压缩图片`，`jpg`压缩后还是`jpg`，`png`压缩后还是`png`，在文件类型上来说还是没有变化。`Loader`的转换过程是附属在整个`Webpack构建流程`中的，意味着打包时间包含了压缩图片的时间成本，对于追求`webpack`性能优化来说实属有点违背原则。而`Plugin`恰好是监听`webpack`运行生命周期中广播的事件，在合适时机通过`webpack`提供的API改变输出结果，所以可在整个`Webpack构建流程`完成后(全部打包文件输出完成后)插入压缩图片的操作。换句话说，打包时间不再包含压缩图片的时间成本，打包完成后该干嘛就干嘛，还能干嘛，压缩图片啊。

所以依据需求情况，`Plugin`作为首选。

##### 编码

依据上述`Plugin`开发思路，那么就开始着手编码了。

笔者把这个压缩图片的`Plugin`命名为**tinyimg-webpack-plugin**，`tinyimg`意味着**TinyJpg**和**TinyPng**合体。

新建项目，目录结构如下。

```txt
tinyimg-webpack-plugin
├─ src
│  ├─ index.js
│  ├─ schema.json
├─ util
│  ├─ getting.js
│  ├─ setting.js
├─ .gitignore
├─ .npmignore
├─ license
├─ package.json
├─ readme.md
复制代码
```

主要文件如下。

- src
  - index.js：入口函数
  - schema.json：参数校验
- util
  - getting.js：常量集合
  - setting.js：函数集合

安装项目所需模块，和上述[compress-img](https://github.com/JowayYoung/juejin-code/tree/master/compress-img)的依赖一致，额外安装`schema-utils`用于校验`Plugin`参数是否符合规定。

```sh
npm i chalk figures ora schema-utils trample
复制代码
```

> 封装常量集合和函数集合

将上述`compress-img`的`TINYIMG_URL`和`RandomHeader()`封装到工具集合中，其中常量集合增加`IMG_REGEXP`和`PLUGIN_NAME`两个常量。

```js
// getting.js
const IMG_REGEXP = /\.(jpe?g|png)$/;

const PLUGIN_NAME = "tinyimg-webpack-plugin";

const TINYIMG_URL = [
    "tinyjpg.com",
    "tinypng.com"
];

module.exports = {
    IMG_REGEXP,
    PLUGIN_NAME,
    TINYIMG_URL
};
复制代码
// setting.js
const { RandomNum } = require("trample/node");

const { TINYIMG_URL } = require("./getting");

function RandomHeader() {
    const ip = new Array(4).fill(0).map(() => parseInt(Math.random() * 255)).join(".");
    const index = RandomNum(0, 1);
    return {
        headers: {
            "Cache-Control": "no-cache",
            "Content-Type": "application/x-www-form-urlencoded",
            "Postman-Token": Date.now(),
            "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.116 Safari/537.36",
            "X-Forwarded-For": ip
        },
        hostname: TINYIMG_URL[index],
        method: "POST",
        path: "/web/shrink",
        rejectUnauthorized: false
    };
}

module.exports = {
    RandomHeader
};
复制代码
```

> 通过`module.exports`导出一个函数或类

```js
// index.js
module.exports = class TinyimgWebpackPlugin {};
复制代码
```

> 在`函数原型或类`上绑定`apply()`访问`Compiler对象`

```js
// index.js
module.exports = class TinyimgWebpackPlugin {
    apply(compiler) {
        // Do Something
    }
};
复制代码
```

> 在`apply()`中指定一个绑定到`webpack`自身的事件钩子

从上述分析中可知，在全部打包文件输出完成后插入压缩图片的操作，所以应该选择该时机对应的事件钩子。从[Webpack Compiler Hooks API文档](https://www.webpackjs.com/api/compiler-hooks)中可发现，`emit`正是这个`Plugin`所需的事件钩子。`emit`在**生成资源到输出目录前执行**，此刻可获取所有图片文件的数据和输出路径。

为了方便在特定条件下`启用功能`和`打印日志`，所以设置相关配置。

- **enabled**：是否启用功能
- **logged**：是否打印日志

在`apply()`中处理相关业务逻辑，可能使用到`Plugin`的入参，那么就得对参数进行校验。定义一个`Plugin`的`Schema`，通过`schema-utils`来校验`Plugin`的入参。

```json
// schema.json
{
    "type": "object",
    "properties": {
        "enabled": {
            "description": "start plugin",
            "type": "boolean"
        },
        "logged": {
            "description": "print log",
            "type": "boolean"
        }
    },
    "additionalProperties": false
}
复制代码
// index.js
const SchemaUtils = require("schema-utils");

const { PLUGIN_NAME } = require("../util/getting");
const Schema = require("./schema");

module.exports = class TinyimgWebpackPlugin {
    constructor(opts) {
        this.opts = opts;
    }
    apply(compiler) {
        const { enabled } = this.opts;
        SchemaUtils(Schema, this.opts, { name: PLUGIN_NAME });
        enabled && compiler.hooks.emit.tap(PLUGIN_NAME, compilation => {
            // Do Something
        });
    }
};
复制代码
```

> 整合`compress-img`到`Plugin`

在整合过程中会有一些小修改，各位同学可对比看看哪些细节发生了变化。

```js
// index.js
const Fs = require("fs");
const Https = require("https");
const Url = require("url");
const Chalk = require("chalk");
const Figures = require("figures");
const { ByteSize, RoundNum } = require("trample/node");

const { RandomHeader } = require("../util/setting");

module.exports = class TinyimgWebpackPlugin {
    constructor(opts) { ... }
    apply(compiler) { ... }
    async compressImg(assets, path) {
        try {
            const file = assets[path].source();
            const obj = await this.uploadImg(file);
            const data = await this.downloadImg(obj.output.url);
            const oldSize = Chalk.redBright(ByteSize(obj.input.size));
            const newSize = Chalk.greenBright(ByteSize(obj.output.size));
            const ratio = Chalk.blueBright(RoundNum(1 - obj.output.ratio, 2, true));
            const dpath = assets[path].existsAt;
            const msg = `${Figures.tick} Compressed [${Chalk.yellowBright(path)}] completed: Old Size ${oldSize}, New Size ${newSize}, Optimization Ratio ${ratio}`;
            Fs.writeFileSync(dpath, data, "binary");
            return Promise.resolve(msg);
        } catch (err) {
            const msg = `${Figures.cross} Compressed [${Chalk.yellowBright(path)}] failed: ${Chalk.redBright(err)}`;
            return Promise.resolve(msg);
        }
    }
    downloadImg(url) {
        const opts = new Url.URL(url);
        return new Promise((resolve, reject) => {
            const req = Https.request(opts, res => {
                let file = "";
                res.setEncoding("binary");
                res.on("data", chunk => file += chunk);
                res.on("end", () => resolve(file));
            });
            req.on("error", e => reject(e));
            req.end();
        });
    }
    uploadImg(file) {
        const opts = RandomHeader();
        return new Promise((resolve, reject) => {
            const req = Https.request(opts, res => res.on("data", data => {
                const obj = JSON.parse(data.toString());
                obj.error ? reject(obj.message) : resolve(obj);
            }));
            req.write(file, "binary");
            req.on("error", e => reject(e));
            req.end();
        });
    }
};
复制代码
```

> 在事件钩子中通过`webpack`提供的API处理资源

通过`compilation.assets`获取全部打包文件的对象，筛选出`jpg`和`png`，使用`map()`将单个图片数据映射为`this.compressImg(file)`，再通过`Promise.all()`操作即可。

整个业务逻辑结合了`Promise`和`Async/Await`两个ES6常用特性，它俩组合起来玩异步编程极其有趣，关于它俩更多细节可查看笔者这篇**4000点赞量**和**14万阅读量**的文章[《1.5万字概括ES6全部特性》](https://juejin.im/post/6844903959283367950)。

```js
// index.js
const Ora = require("ora");
const SchemaUtils = require("schema-utils");

const { IMG_REGEXP, PLUGIN_NAME } = require("../util/getting");
const Schema = require("./schema");

module.exports = class TinyimgWebpackPlugin {
    constructor(opts) { ... }
    apply(compiler) {
        const { enabled, logged } = this.opts;
        SchemaUtils(Schema, this.opts, { name: PLUGIN_NAME });
        enabled && compiler.hooks.emit.tap(PLUGIN_NAME, compilation => {
            const imgs = Object.keys(compilation.assets).filter(v => IMG_REGEXP.test(v));
            if (!imgs.length) return Promise.resolve();
            const promises = imgs.map(v => this.compressImg(compilation.assets, v));
            const spinner = Ora("Image is compressing......").start();
            return Promise.all(promises).then(res => {
                spinner.stop();
                logged && res.forEach(v => console.log(v));
            });
        });
    }
    async compressImg(assets, path) { ... }
    downloadImg(url) { ... }
    uploadImg(file) { ... }
};
复制代码
```

> 通过`webpack`提供的方法返回该资源

由于压缩图片的操作是在整个`Webpack构建流程`完成后，所以没有什么可返回了，故不作处理。

> 控制`webpack`依赖版本

由于`tinyimg-webpack-plugin`基于`webpack v4`，所以需在`package.json`中添加`peerDependencies`，用来告知安装该`Plugin`的模块必须存在`peerDependencies`里的依赖。

```json
{
    "peerDependencies": {
        "webpack": ">= 4.0.0",
        "webpack-cli": ">= 3.0.0"
    }
}
复制代码
```

> 总结

按照上述总结的开发思路一步一步来完成编码，其实是挺简单的。若需开发一些跟自己项目相关的`Plugin`，还是需多多熟悉[Webpack Compiler Hooks API文档](https://www.webpackjs.com/api/compiler-hooks)，相信各位同学都能手戳一个完美的`Plugin`出来。

`tinyimg-webpack-plugin`源码请戳[这里](https://github.com/JowayYoung/tinyimg-webpack-plugin)查看，**Star**一个如何，嘻嘻。

![电眼猪](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="800" height="600"></svg>)

### 测试

整个`Plugin`开发完成，接下来需走一遍测试流程，看能不能把这个`压缩图片的扩展器`跑通。相信各位同学都是一位标准的`Webpack配置工程师`，可自行编写测试Demo验证你们的`Plugin`。

在根目录下创建`test文件夹`，并按照以下目录结构加入文件。

```txt
tinyimg-webpack-plugin
├─ test
│  ├─ src
│  │  ├─ img
│  │  │  ├─ favicon.ico
│  │  │  ├─ gz.jpg
│  │  │  ├─ pig-1.jpg
│  │  │  ├─ pig-2.jpg
│  │  │  ├─ pig-3.jpg
│  │  ├─ index.html
│  │  ├─ index.js
│  │  ├─ index.scss
│  │  ├─ reset.css
│  └─ webpack.config.js
复制代码
```

安装测试Demo所需的`webpack`相关配置模块。

```sh
npm i -D @babel/core @babel/preset-env babel-loader clean-webpack-plugin css-loader file-loader html-webpack-plugin mini-css-extract-plugin node-sass sass sass-loader style-loader url-loader webpack webpack-cli webpackbar
复制代码
```

安装完成后，着手完善`webpack.config.js`代码，代码量有点多，直接贴链接好了，请戳[这里](https://github.com/JowayYoung/tinyimg-webpack-plugin/blob/master/test/webpack.config.js)。

最后在`package.json`中的`scripts`插入以下`npm scripts`，然后执行`npm run test`调试测试Demo。

```json
{
    "scripts": {
        "test": "webpack --config test/webpack.config.js"
    }
}
复制代码
```

### 发布

发布到`NPM仓库`上非常简单，仅需几行命令。若还没注册，赶紧去`NPM`上注册一个账号。若当前镜像为`淘宝镜像`，需执行`npm config set registry https://registry.npmjs.org/`切换回源镜像。

接下来一波操作就可完成发布了。

- 进入目录：`cd my-plugin`
- 登录账号：`npm login`
- 校验状态：`npm whoami`
- 发布模块：`npm publish`
- 退出账号：`npm logout`

------

若不想牢记这么多命令，可用笔者开发的`pkg-master`一键发布，若存在某些错误会立马中断发布并提示错误信息，是一个非常好用的集成创建和发布的NPM模块管理工具。详情请查看[文档](https://github.com/JowayYoung/pkg-master)，顺便给一个Star以作鼓励。

> 安装

```
npm i -g pkg-master
```

> 使用

| 命令                 | 缩写           | 功能     | 描述                                                |
| -------------------- | -------------- | -------- | --------------------------------------------------- |
| `pkg-master create`  | `pkg-master c` | 创建模块 | 生成模块的`基础文件`                                |
| `pkg-master publish` | `pkg-master p` | 发布模块 | 检测NPM的`运行环境`和`账号状态`，通过则自动发布模块 |

![发布模块](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="800" height="600"></svg>)

### 接入

> 安装

```
npm i tinyimg-webpack-plugin
```

> 使用

| 配置      | 功能         | 格式         | 描述                   |
| --------- | ------------ | ------------ | ---------------------- |
| `enabled` | 是否启用功能 | `true/false` | 建议只在生产环境下开启 |
| `logged`  | 是否打印日志 | `true/false` | 打印处理信息           |

在`webpack.config.js`或`webpack配置`插入以下代码。

> 在CommonJS中使用

```js
const TinyimgPlugin = require("tinyimg-webpack-plugin");

module.exports = {
    plugins: [
        new TinyimgPlugin({
            enabled: process.env.NODE_ENV === "production",
            logged: true
        })
    ]
};
复制代码
```

> 在ESM中使用

必须在`babel`加持下的Node环境中使用

```js
import TinyimgPlugin from "tinyimg-webpack-plugin";

export default {
    plugins: [
        new TinyimgPlugin({
            enabled: process.env.NODE_ENV === "production",
            logged: true
        })
    ]
};
复制代码
```

> 推荐一个零配置开箱即用的React/Vue应用自动化构建脚手架

`bruce-cli`是一个**React/Vue**应用自动化构建脚手架，其零配置开箱即用的优点非常适合入门级、初中级、快速开发项目的前端同学使用，还可通过创建`brucerc.js`文件来覆盖其默认配置，只需专注业务代码的编写无需关注构建代码的编写，让项目结构更简洁。使用时记得查看文档哟，喜欢的话给个Star。

当然，笔者已将`tinyimg-webpack-plugin`集成到`bruce-cli`中，零配置开箱即用走起。

- Github地址：请戳[这里](https://github.com/JowayYoung/bruce-cli)
- 官网文档：请戳[这里](https://yangzw.vip/source?id=bruce-cli)
- 掘金文档：请戳[这里](https://juejin.im/post/6844903763589726222)

### 总结

总体来说开发一个`Webpack Plugin`不难，只需好好分析需求，了解`webpack`运行生命周期中广播的事件，编写`自定义Plugin`在合适时机通过`webpack`提供的API改变输出结果。

若觉得`tinyimg-webpack-plugin`对你有帮助，可在[Issue](https://github.com/JowayYoung/tinyimg-webpack-plugin/issues)上`提出你的宝贵建议`，笔者会认真阅读并整合你的建议。喜欢`tinyimg-webpack-plugin`的请给一个[Star](https://github.com/JowayYoung/tinyimg-webpack-plugin)，或[Fork](https://github.com/JowayYoung/tinyimg-webpack-plugin)本项目到自己的`Github`上，根据自身需求定制功能。

### 结语

**❤️关注+点赞+收藏+评论+转发❤️**，原创不易，鼓励笔者创作更好的文章

**关注公众号`IQ前端`，一个专注于CSS/JS开发技巧的前端公众号，更多前端小干货等着你喔**

- 关注后回复`关键词`免费领取视频教程
- 关注后添加`我微信`拉你进技术交流群
- 欢迎关注`IQ前端`，更多**CSS/JS开发技巧**只在公众号推送

![img](https://yangzw.vip/static/frontend/account/IQ%E5%89%8D%E7%AB%AF%E5%85%AC%E4%BC%97%E5%8F%B7.jpg)


作者：JowayYoung
链接：https://juejin.im/post/6882551009219575815
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。