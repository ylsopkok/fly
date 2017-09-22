# Fly.js

Fly.js 是一个基于 promise 的，轻量且强大的Javascript http 网络库，它有如下特点：

1. 同时支持浏览器和 node 环境。
2. 支持 Promise API
3. 支持请求／响应拦截器
4. 自动转换 JSON 数据
5. **可以随意切换底层 http engine， 在浏览器环境中默认使用 XMLHttpRequest**
6. **h5页面内嵌到原生 APP 中，可以将 http 请求转发到 Native，在端上统一发起网络请求、进行 cookie 管理、证书验证。**
7. 内嵌到APP时，通过native engine 可以直接请求图片数据。
8. node下支持文件下载／上传、http代理等本地增强功能。
9. 高度可定制。
10. **非常非常轻量**。

## 官网

详细的文档请移步：[Flyio官网文档](https://wendux.github.io/dist/#/doc/flyio/readme) 。 官网http请求使用的正是fly，为了方便大家验证fly功能特性，官网对fly进行了全局引入，您可以在官网页面打开控制台直接验证。

## 安装

### 使用npm

```shell
npm install flyio
```

### 使用cdn

```javascript
<script src="https://unpkg.com/flyio/dist/fly.min.js"></script>
```

### umd

```http
https://unpkg.com/flyio/dist/fly.umd.min.js
```

## 例子

下面示例如无特殊说明，则在浏览器和node环境下都能正常执行。

### 发起GET请求

```javascript
var fly=require("flyio")
//通过用户id获取信息,参数直接写在url中
fly.get('/user?id=133')
  .then(function (response) {
    console.log(response);
  })
  .catch(function (error) {
    console.log(error);
  });

//query参数通过对象传递
fly.get('/user', {
      id: 133
  })
  .then(function (response) {
    console.log(response);
  })
  .catch(function (error) {
    console.log(error);
  });

```

### 发起POST请求

```javascript
fly.post('/user', {
    name: 'Doris',
    age: 24
    phone:"18513222525"
  })
  .then(function (response) {
    console.log(response);
  })
  .catch(function (error) {
    console.log(error);
  });
```

### 发起多个并发请求

```javascript
function getUserRecords() {
  return fly.get('/user/133/records');
}

function getUserProjects() {
  return fly.get('/user/133/projects');
}

fly.all([getUserRecords(), getUserProjects()])
  .then(fly.spread(function (records, projects) {
    //两个请求都完成
  }))
  .catch(function(error){
    console.log(error)
  })
```

### 直接通过request api发起请求

```javascript
//直接调用request函数发起post请求
fly.request("/test",{hh:5},{
    method:"post",
    timeout:5000 //超时设置为5s
 })
.then(d=>{ console.log("request result:",d)})
.catch((e) => console.log("error", e))
```

### 发送FormData

```javascript
 var formData = new FormData();
 var log=console.log
 formData.append('username', 'Chris');
 fly.post("../package.json",formData).then(log).catch(log)
```

注：Fly目前只在支持FormData的浏览器环境中支持FormData，node环境下对formData的支持方式稍有不同，详情戳这里 [Node 下增强的API ](#/doc/flyio/node)。



## 拦截器

Fly支持请求／响应拦截器，可以通过它在请求发起之前和收到响应数据之后做一些预处理。

```javascript

//添加请求拦截器
fly.interceptors.request.use((config,promise)=>{
    //给所有请求添加自定义header
    config.headers["X-Tag"]="flyio";
    //可以通过promise.reject／resolve直接中止请求
    //promise.resolve("fake data")
    return config;
})

//添加响应拦截器，响应拦截器会在then/catch处理之前执行
fly.interceptors.response.use(
    (response,promise) => {
        //只将请求结果的data字段返回
        return response.data
    },
    (err,promise) => {
        //发生网络错误后会走到这里
        //promise.resolve("ssss")
    }
)
```

如果你想移除拦截器，只需要将拦截器设为null即可：

```javascript
fly.interceptors.request.use(null)
fly.interceptors.response.use(null,null)
```



## 错误处理

请求失败之后，catch捕获到的err为Error的一个实例，有两个字段

```javascript
{
  message:"Not Find 404", //错误消息
  status:404 //如果服务器可通，则为http请求状态码。网络异常时为0，网络超时为1
}
```

**错误码**

| 错误码   | 含义                         |
| ----- | -------------------------- |
| 0     | 网络错误                       |
| 1     | 请求超时                       |
| 2     | 文件下载成功，但保存失败，此错误只出现node环境下 |
| >=200 | http请求状态码                  |



## 请求配置选项

可配置选项：

```javascript
{
  headers:{}, //http请求头，
  baseURL:"", //请求基地址
  timeout:0,//超时时间，为0时则无超时限制
  withCredentials:false //跨域时是否发送cookie
}
```

配置支持**实例级配置**和**单次请求配置**

### 实例级配置

实例级配置可用于当前Fly实例发起的所有请求

```javascript
//定义公共headers
fly.config.headers={xx:5,bb:6,dd:7}
//设置超时
fly.config.timeout=10000;
//设置请求基地址
fly.config.baseURL="https://wendux.github.io/"
```

### 单次请求配置

需要对单次请求配置时，需使用request方法，配置只对当次请求有效。

```javascript
fly.request("/test",{hh:5},{
    method:"post",
    timeout:5000 //超时设置为5s
})
```

**注：若单次配置和实例配置冲突，则会优先使用单次请求配置**



## 创建Fly实例

为方便使用，在引入flyio库之后，会创建一个默认实例，一般情况下大多数请求都是通过默认实例发出的，但在一些场景中需要多个实例（可能使用不同的配置请求），这时你可以手动new一个：

```javascript
//npm、node环境下
var  Fly=require("fio/dist/fly") //注意！此时引入的是 "fio/dist/fly"
var nFly=new Fly();

//CDN引入时直接new
var nFly=new Fly();
```



## Http engine

Fly 引入了Http engine 的概念，所谓 Http engine，就是真正发起http请求的引擎，这在浏览器中一般都是XMLHttpRequest，而在 node 环境中，可以用任何能发起网络请求的库／模块实现，Fly 可以自由更换底层 http engine ，Fly 正是通过更换 engine 而实现同时支持 node 和 browser 。值得注意的是，http engine 不局限于node 和 browser 环境中，也可以是 android、ios、electron，正是由于这些，Fly 才有了一个非常强大的功能——**请求重定向**。基于请求重定向，我们可以实现一些非常有用的功能，比如**将内嵌到 APP 的所有 http 请求重定向到 Native ，然后在端上( android、ios )统一发起网络请求、进行 cookie 管理、证书校验**。详情请戳 [Fly Http Engine ](#/doc/flyio/engine)。



## 体积

Fly 非常轻量，min 只有 4.6K 左右，GZIP 压缩后不到 2K, 体积是 axios 的四分之一。



## Finally

如果感觉 Fly 对您有用，欢迎 star 。


