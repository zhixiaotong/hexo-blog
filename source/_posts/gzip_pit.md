---
title: 使用gzip优化页面加载遇到的问题
date: 2021-01-21 15:35:06
tags: ["Javascript"]
category: "那些年我走过的坑"
thumbnail: /gallery/thumbnails/what-is-gzip.jpg
copyright: true
---
![u=1180411375,4109830813&fm=26&gp=0.jpg](https://upload-images.jianshu.io/upload_images/17756630-b0bfbc54bb9744b1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 背景

我司目前项目已经经过了很多次优化了，期间通过webpack打包实践了很多种方式，包括样式文件通过[**gulp**](https://www.gulpjs.com.cn/)实现命令行级别输出到static目录下，避免了此类css的打包，通过[**equire**](https://www.npmjs.com/package/babel-plugin-equire)插件实现了echarts的按需引入，换掉了momentjs替换成了轻量的dayjs，还有一些比较细的优化，在此就不一一列举了。
<!-- more -->

但是由于多页面配置，每个页面的js都挺大的，导致进入页面不是很流畅。期间曾经尝试了gzip，栽坑里了，由于需求原因又搁置了，近期手头工作比较少，又开始了踩坑的路途。

## 为什么使用gzip？

gzip是一种通用的压缩算法，可以降低网络传输的数据量，从而提高客户端浏览器的访问速度。由于浏览器向服务器发送请求，加载服务器响应的静态资源，当资源文件过大，会导致用户感受的加载时间较长，影响用户体验。

虽然浏览器还需要解压gzip文件，但这速度跟加载一个未压缩的文件相比，简直少得不要不要的了。

当然，我们可以通过采用其他优化方式并用提高页面加载速度，实在无济于事了，也可以加一朵旋转的*小菊花*[狗头]，但gzip的使用能大幅降低加载时间，何乐而不为呢？

## 在打包代码时使用gzip

由于项目是基于vue开发，在`vue.config.js`中添加下列配置即可。（react也大同小异）

```javascript
const CompressionPlugin = require('compression-webpack-plugin')
...
module.exports = {
  ...
  chainWebpack: config => {
    ...
    // 就这一行，让你爽到飞
    config.plugin('CompressionPlugin').use(CompressionPlugin, [])
  }
}
```

然后你就可以通过执行，查看`/dist/`文件夹下的变化了。

```javascript
vue-cli-service build
```

如果你需要比较详细的配置gzip打包，请移步[https://www.npmjs.com/package/compression-webpack-plugin](https://www.npmjs.com/package/compression-webpack-plugin)
配置里中有一个`deleteOriginalAssets`属性，意思是删除源文件，一般为`false`即可，同时保留源文件和压缩后的`.gz`后缀文件。
当然这不是本篇文章的重点。重点是什么？当然是**坑**了！

## nginx如何配置匹配.gz文件？

由于现在的浏览器默认支持gzip压缩，你可以在请求头里看到`Accept-Encoding: gzip, deflate`得以确认。

我司服务器使用nginx做请求转发和实现简单的负载均衡。在网上，不乏有如何在nginx上开启gzip配置的博客或文章。其实蛮简单的，类似一下命令键入打开nginx配置文件：

```shell
cd /usr/local/nginx/conf/
vim nginx.conf
```

再在`http`模块下添加这么几行配置：

```shell
gzip on; // 开启gzip
gzip_static on; // 开启gzip静态资源
gzip_comp_level 5; // 1-10，数值越大，压缩越狠，但越占用CPU时间，到达6左右，压缩已经不明显了
gzip_http_version 1.1 // 1.0或1.1，nginx默认HTTP 1.1
gzip_vary on; // 启用应答头"Vary: Accept-Encoding"
gzip_types text/plain text/css text/javascript application/javascript application/xml application/x-httpd-php application/vnd.ms.fontobject image/jpeg image/png image/gif image/svg+xml font/ttf font/opentype font/x-woff // 压缩类型
...
// 其他具体配置可自行搜索，此处就不列举了
```

### gzip和gzip_static的区别

- gzip：实时压缩，**Response Headers** 中 的**ETag**属性存在类似 W/"***"的内容。
- gzip_static：优先级高于gzip，会去在目录下优先找带.gz后缀的文件，没有再获取源文件，开启了gzip_static，无须在服务器打包，节约了服务器cpu的使用。

> 由于项目是在前端构建打包的时候启用了gzip，因为dist内已经有*.gz*后缀文件，`gzip_types`设置其实没啥作用（因为有了静态资源，不需要服务器实时压缩）。

### 坑点一

#### 问题

命令行中报错，内容如下显示：

```shell
nginx:[emerg] unknown directive "gzip_static" in /usr/local/nginx/conf/nginx.conf:35
```

意思是“gzip_static”指令未知，无法被识别。

#### 解决方式

由于安装nginx时缺少了相应配置，需要添加`with-http_gzip_static_module`配置，在此之前需要先执行以下命令：

```shell
// 打开nginx脚本目录
cd /usr/local/nginx/sbin/
// 查看nginx的版本和既有的配置
nginx -V
```

再在nginx安装目录下，重新编译和安装nginx，具体方式如下：

```shell
// 打开nginx安装目录
cd /nginx-1.9.9/
// 添加相应配置（除gzip_static配置外，其他配置均为通过nginx -V查看显示的之前既有配置）
./configure  --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module --with-http_flv_module --with-http_gzip_static_module
// 编译
make
make install
```

> 需键入`nginx -V`获取之前的configure，在之前的configure后面添加gzip_static模块

然后等待进程执行完，重新启动nginx即可。

### 坑点二

#### 问题

通过以上的方式，nginx已经支持了`gzip_static`模块，以便nginx优先匹配*.gz*后缀文件返回。执行以下命令重新nginx：

```shell
cd /usr/local/nginx/sbin/
nginx -s reload
```

好吧，正常情况下，是这样的。

但是...

如果你打开浏览器F12，刷新页面查看资源请求，发现静态资源的**Response Headers**属性中却没有`Content-Encoding: gzip`。喔豁...显然配置没有生效啊！

#### 解决方式

我们现在的问题发生在重启了nginx，新添加的模块却没有生效，浏览器还是获取的源文件。此时，我就开始百度了，百度大法好啊，各种回答都出来了。

##### 回答一

nginx没有生效，可能是你的配置有误，语法不合法？检查一下：

```shell
cd /usr/local/nginx/sbin/
// 添加-t执行检查
nginx -t -C /usr/local/nginx/conf/nginx.conf
```

结果命令行出现以下信息：

```shell
nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful
```

显示语法ok了，测试也通过了，说明正常启动了吧？打开浏览器，又查看了一遍资源加载......
![423.gif](https://upload-images.jianshu.io/upload_images/17756630-d6da038cc0397716.gif?imageMogr2/auto-orient/strip)

##### 回答二

第一种试过了，哭了。
那么nginx重启没生效，会不会是重启没成功啊？软的不行，来硬的呗。键入：

```shell
// 查看nginx进程，获取到nginx主进程pid为20468
ps -ef|grep nginx
// 强制删除
kill -9 20468
// 再次执行确保进程被删除
ps -ef|grep nginx
```

此时，进程中如果没有nginx，`kill`命令生效了。此时，我重新在命令行中进入nginx命令行目录，运行`nginx -s reload`，意想不到的问题出来了。

```shell
nginx: [emerg] bind() to [::]:80 failed (98: Address already in use)
```

我又重新试了kill命令又报错：

```shell
nginx: [alert] kill(20468, 1) failed (3: No such process)
```

![u=1023318312,4042958451&fm=26&gp=0.jpg](https://upload-images.jianshu.io/upload_images/17756630-224138efbac497cf.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
杀掉了主进程，却没有启动成功！
吓得我一个激灵看了一下页面，咦？
正常情况下，刷新页面，页面会由于nginx进程被`kill`导致页面无法访问。但是，页面依然能正常打开，俺不信邪，又试了好几个页面，果不其然都行。（不知道是该开心，还是该难过？）
![u=3318123593,1248177504&fm=26&gp=0.jpg](https://upload-images.jianshu.io/upload_images/17756630-f24f954987126828.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
nginx进程都没了，为啥还能访问页面？我又又问百度了，经过多方查探，发现有回答说是可能因为程序占用了端口导致`kill`不掉。在命令行键入以下命令：

```shell
// 或者netstat -ntlp | grep 80
netstat -ntlp
```

查看到进程列表，或者通过`fuser -n tcp 80`查看里面是否**nginx**的占用？

果不其然，的确有这个进程，再次通过`kill`命令删除PID进程号，再次执行之前命令，发现列表中nginx进程不复存在了。点此查看[彻底删除nginx进程](https://blog.csdn.net/nil_lu/article/details/82682385)

此时，我们刷新浏览器页面，地址访问不了了，确定nginx已经被彻底删除，此时执行[nginx重启命令](https://www.cnblogs.com/codingcloud/p/5095066.html)，测试`gzip_static`测试是否成功应用。不出意外，nginx应用gzip应该是成功了。此时，你应该能成功看到**Request Headers**中的`Content-Encoding: gzip`了，明显能感到页面加载的速度变快了。
![a421444a20a44623299aa10c8f22720e0cf3d70b.gif](https://upload-images.jianshu.io/upload_images/17756630-c0b76c1cc07d9518.gif?imageMogr2/auto-orient/strip)
如果你经过最开始的安装`gzip_static`后没有出现后续问题，那么恭喜你没有栽坑里[狗头]。如果栽了跟头也没关系，毕竟百度大法好啊（虽然google更香...）

这次记录就到这了，希望对你能有帮助~Skr
