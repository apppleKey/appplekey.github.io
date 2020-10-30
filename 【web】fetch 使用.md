# fetch 使用



#### 兼容性

![img](https://upload-images.jianshu.io/upload_images/4262751-4cc6ec5fc73c6273.png?imageMogr2/auto-orient/strip|imageView2/2/w/756/format/webp)

解决：

```shell
//shell
cnpm install babel-polyfill es6-promise fetch-detector fetch-ie8 --save

//js
import 'babel-polyfill';
require('es6-promise').polyfill();
import 'fetch-detector';
import 'fetch-ie8';
//注意： fetch-detector 一定要在 fetch-ie8 之前引入
```

js 中引入

```javascript
import 'babel-polyfill';
require('es6-promise').polyfill();
import 'fetch-detector';
import 'fetch-ie8';
```

Fetch polyfill 的基本原理是探测是否存在 window.fetch 方法，如果没有则用 XHR 实现。这也是 github/fetch 的做法，但是有些浏览器（Chrome 45）原生支持 Fetch，但响应中有中文时会乱码，老外又不太关心这种问题，所以才有了 fetch-detector 和 fetch-ie8， 只在浏览器稳定支持 Fetch 情况下才使用原生 Fetch。这些库现在每天有几千万个请求都在使用，绝对靠谱！

#### WHAT

https://developer.mozilla.org/zh-CN/docs/Web/API/Fetch_API/Using_Fetch

Fetch 提供了对 [`Request`](https://developer.mozilla.org/zh-CN/docs/Web/API/Request) 和 [`Response`](https://developer.mozilla.org/zh-CN/docs/Web/API/Response) （以及其他与网络请求有关的）对象的通用定义。使之今后可以被使用到更多地应用场景中：无论是 service worker、Cache API、又或者是其他处理请求和响应的方式，甚至是任何一种需要你自己在程序中生成响应的方式。

它同时还为有关联性的概念，例如CORS和HTTP原生头信息，提供一种新的定义，取代它们原来那种分离的定义。

发送请求或者获取资源，需要使用 [`WindowOrWorkerGlobalScope.fetch()`](https://developer.mozilla.org/zh-CN/docs/Web/API/WindowOrWorkerGlobalScope/fetch) 方法。它在很多接口中都被实现了，更具体地说，是在 [`Window`](https://developer.mozilla.org/zh-CN/docs/Web/API/Window) 和 [`WorkerGlobalScope`](https://developer.mozilla.org/zh-CN/docs/Web/API/WorkerGlobalScope) 接口上。因此在几乎所有环境中都可以用这个方法获取到资源。

 `fetch()` 必须接受一个参数——资源的路径。无论请求成功与否，它都返回一个 Promise 对象，resolve 对应请求的 [`Response`](https://developer.mozilla.org/zh-CN/docs/Web/API/Response)。你也可以传一个可选的第二个参数 `init`（参见 [`Request`](https://developer.mozilla.org/zh-CN/docs/Web/API/Request)）。

一旦 [`Response`](https://developer.mozilla.org/zh-CN/docs/Web/API/Response) 被返回，就可以使用一些方法来定义内容的形式，以及应当如何处理内容（参见 [`Body`](https://developer.mozilla.org/zh-CN/docs/Web/API/Body)）。

你也可以通过 [`Request()`](https://developer.mozilla.org/zh-CN/docs/Web/API/Request/Request) 和 [`Response()`](https://developer.mozilla.org/zh-CN/docs/Web/API/Response/Response) 的构造函数直接创建请求和响应，但是我们不建议这么做。他们应该被用于创建其他 API 的结果（比如，service workers 中的 [`FetchEvent.respondWith`](https://developer.mozilla.org/zh-CN/docs/Web/API/FetchEvent/respondWith)）。

#### WHY

1.  传统的xhr请求

   ```
   var xhr = new XMLHttpRequest();
   xhr.open('GET', url);
   xhr.responseType = 'json';
   
   xhr.onload = function() {
     console.log(xhr.response);
   };
   
   xhr.onerror = function() {
     console.log("Oops, error");
   };
   
   xhr.send();
   ```

   

2. fetch 

   ```javascript
   fetch(url，{
        credentials: 'include', 
        headers: { 'Accept': 'application/json, text/plain, */*' } 
   }).then(function(response) {
     return response.json();
   }).then(function(data) {
     console.log(data);
   }).catch(function(e) {
     console.log("Oops, error");
   });
   
   
   ```

   现在看起来好很多了，但这种 Promise 的写法还是有 Callback 的影子，而且 promise 使用 catch 方法来进行错误处理的方式有点奇怪。不用急，下面使用 async/await 来做最终优化：

   ```javascript
   async function(){
      try {
     	 let response = await fetch(url);
   	 let data = response.json();
         console.log(data);
      } catch(e) {
         console.log(e);
      }
   }
   
   ```

   优点：

   1. 写异步代码就像写同步代码一样爽

   2. 语法简洁，更加语义化

   3. 基于标准 Promise 实现，支持 async/await

   缺点：

   1. Fetch 请求默认是不带 cookie 的，需要设置 fetch(url, {credentials: 'include'})

   2. 服务器返回 400，500 错误码时并不会 reject，只有网络错误这些导致请求不能完成时，fetch 才会被 reject。

   3. Fetch 和标准 Promise 的不足

      由于 Fetch 是典型的异步场景，所以大部分遇到的问题不是 Fetch 的，其实是 Promise 的。ES6 的 Promise 是基于 [Promises/A+](https://promisesaplus.com/) 标准，为了保持 **简单简洁** ，只提供极简的几个 API。如果你用过一些牛 X 的异步库，如 jQuery(不要笑) 、Q.js 或者 RSVP.js，可能会感觉 Promise 功能太少了。

   4. 其他

      ```
      没有 Deferred
      Deferred 可以在创建 Promise 时可以减少一层嵌套，还有就是跨方法使用时很方便。
      ECMAScript 11 年就有过 Deferred 提案，但后来没被接受。其实用 Promise 不到十行代码就能实现 Deferred：es6-deferred。现在有了 async/await，generator/yield 后，deferred 就没有使用价值了。
      
      没有获取状态方法：isRejected，isResolved
      标准 Promise 没有提供获取当前状态 rejected 或者 resolved 的方法。只允许外部传入成功或失败后的回调。我认为这其实是优点，这是一种声明式的接口，更简单。
      
      缺少其它一些方法：always，progress，finally
      always 可以通过在 then 和 catch 里重复调用方法实现。finally 也类似。progress 这种进度通知的功能还没有用过，暂不知道如何替代。
      
      不能中断，没有 abort、terminate、onTimeout 或 cancel 方法
      Fetch 和 Promise 一样，一旦发起，不能中断，也不会超时，只能等待被 resolve 或 reject。幸运的是，whatwg 目前正在尝试解决这个问题 whatwg/fetch#27
      ```

      

#### HOW

post请求

```csharp
var result = fetch('/api/post', { 
      method: 'POST', 
      credentials: 'include',
      headers: { 'Accept': 'application/json, text/plain, */*', 'Content-Type': 'application/x-www-form-urlencoded' }, 
      body: "a=100&b=200"  });
```

fetch 提交数据之后，返回的结果也是一个 Promise 对象，跟 get 方法一样，处理方式也一样。

##### 抽象 get 和 post

新建两个文件 `post.js 和 get.js`，分别对 get 方法 和 post 方法封装



```jsx
//  get.js
import 'whatwg-fetch'
import 'es6-promise'

export function get(url) {
  var result = fetch(url, {
      credentials: 'include',
      headers: {
          'Accept': 'application/json, text/plain, */*'
      }
  });

  return result;
}
```



```jsx
// post.js
import 'whatwg-fetch'
import 'es6-promise'

// 将对象拼接成 key1=val1&key2=val2&key3=val3 的字符串形式
function obj2params(obj) {
    var result = '';
    var item;
    for (item in obj) {
        result += '&' + item + '=' + encodeURIComponent(obj[item]);
    }

    if (result) {
        result = result.slice(1);
    }

    return result;
}

// 发送 post 请求
export function post(url, paramsObj) {
    var result = fetch(url, {
        method: 'POST',
        credentials: 'include',
        headers: {
            'Accept': 'application/json, text/plain, */*',
            'Content-Type': 'application/x-www-form-urlencoded'
        },
        body: obj2params(paramsObj)
    });

    return result;
}
```

这两个方法抽象之后，接下来我们新建一个`data.js`文件，在这个文件中引用



```tsx
// data.js
import { get } from './get.js'
import { post } from './post.js'

export function getData() {
    // '/api/1' 获取字符串
    var result = get('/api/1')

    result.then(res => {
        return res.text()
    }).then(text => {
        console.log(text)
    })

    // '/api/2' 获取json
    var result1 = get('/api/2')

    result1.then(res => {
        return res.json()
    }).then(json => {
        console.log(json)
    })
}

export function postData() {
    // '/api/post' 提交数据
    var result = post('/api/post', {
        a: 100,
        b: 200
    })

    result.then(res => {
        return res.json()
    }).then(json => {
        console.log(json)
    })
}
```