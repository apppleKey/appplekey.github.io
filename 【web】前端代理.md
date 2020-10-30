### 前端代理

#### 1. 什么是代理

``代理服务器（Proxy Server）的功能是代理网络用户去取得网络信息。形象地说，它是网络信息的中转站，是个人网络和Internet服务商之间的中间代理机构，负责转发合法的网络信息，对转发进行控制和登记。``

![image-20200603112144952](./images/image-20200603112144952.png)



#### 2.为什么要代理

​	**方便开发**

 	1. 本地开发（前端代码在本地，**接口在远方dev/test/pro/prod**）   ：接口：本地-->线上
 	2. 线上问题解决 （**前端代码/接口在远方dev/test/pro/prod**,部分代码在本地）：文件 ：线上-->本地



​	

#### 3.如何代理

 1. vue脚手架集成代理

    ```javascript
    devServer: {
            proxy: {
                '/api': {  // 需要直接代理到线上环境的接口
                    target: 'http://ad.server.com',
                    changeOrigin: true,
                    headers: {
                        // 后端要校验请求源，那改下 host 或者 origin 不就美滋滋了？
                        Host: 'ad.server.com',
                        Origin: 'ad.server.com'
                    },
                },
                '/index.php':{
                  target: 'https://a03-dev-web.we-pj.com', //API服务器的地址
                  ws: true, //代理websockets
                  changeOrigin: true, // 是否跨域，虚拟的站点需要更管origin
                },
                '/api': {
                  target: 'https://a03-dev-web.we-pj.com', //API服务器的地址
                  ws: true, //代理websockets
                  changeOrigin: true, // 是否跨域，虚拟的站点需要更管origin
                  pathRewrite: {
               
                     '^/api': '/fire-api',
                  //  'http://localhost:8081/api/login' 转接为 http://localhost:8081/fire-api/login
                  
                  }
                },
             
            }
        }
    
    
    ```

    优点：代理简单方便，起项目即用

    缺点：改代理需要重启项目，实际代理地址不好捕捉

 2. nginx代理

     　nginx支持配置反向代理，通过反向代理实现网站的负载均衡。nginx通过proxy_pass_http 配置代理站点，upstream实现负载均衡。

​    

    配置     https://www.cnblogs.com/ysocean/p/9392908.html 


​    

    举例：
    
    ```javascript
    前端访问接口时，肯定不会使用ip地址来访问，因此使用nginx来做反向代理。
    nginx配置如下：
    
    upstream server {
      server 127.0.0.1:8083;
    }
    
    server {
        listen 80;
        server_name www.smallmage.com;
    
        location /api {
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header Host $http_host;
          proxy_set_header X-NginX-Proxy true;
    
          # value for proxy_pass has to match upstream name
          proxy_pass http://server;
          proxy_redirect off;
        }
    }
    
    上面配置的意思就是：当访问www.smallmage.com/api时就将请求丢给proxy_pass设置的服务，也就是127.0.0.1:8083，假如我们的node服务就监听着8083端口，那么node服务就会接收到该请求，然后再对该请求进行处理，最后返回相应的数据
    ```
    
    优点，配置丰富功能多  
    
     缺点：代理服务器本地做host（得想办法走到nginx代理服务器上）

 3. fiddler代理

    ![image-20200603100515913](./images/image-20200603100515913.png)

    教程 ： https://www.jianshu.com/p/99b6b4cd273c

    优点：快捷，可切换，对代码无污染

    缺点：会导致部分应用请求出问题

