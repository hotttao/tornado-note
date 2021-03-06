# Tornado 源码解析

## 1. 涵盖内容
本文将结合以下内容解析 Tornado 源码
  - [tornado 官方文档](http://tornado-zh.readthedocs.io/zh/latest/guide/coroutines.html)
  - [Introduction To Tornado](http://demo.pythoner.com/itt2zh/index.html)
  - [tornado 源码](http://github.com/tornadoweb/tornado/tree/stable/tornado)

## 2. tornado 组成部分
Tornado 大体上可以被分为4个主要的部分:
1. 异步网络库: IOLoop/IOStream, 为 HTTP 组件提供构建模块
2. 协程库: tornado.gen, 允许异步代码写的更直接而不用链式回调的方式
3. HTTP的客户端和服务端实现 (HTTPServer and AsyncHTTPClient)
4. web框架: 包括创建web应用的 RequestHandler 类，还有很多其他支持的类


## 3. 参考资料
UML: http://www.shaheng.me/blog/2016/08/tornado.html

文档:
  - http://tornado-zh.readthedocs.io/zh/latest/index.html
  - http://www.tornadoweb.org/en/stable/index.html

系列博文:
  - **http://www.cnblogs.com/Bozh/archive/2012/07/22/2603976.html**
  - http://www.nowamagic.net/academy/detail/13321002
  - http://www.cnblogs.com/wupeiqi/tag/Tornado/
  - http://strawhatfy.github.io/categories/tornado/

协程:
  1. **http://blog.csdn.net/wyx819/article/details/45420017**
  2. **http://www.binss.me/blog/analyse-the-implement-of-coroutine-in-tornado/**
  3. **http://mp.weixin.qq.com/s/E-7bMIdmzerr7xpSqD0YXg**
  4. **http://www.jianshu.com/p/3e58f977b908**
  5. http://www.binss.me/blog/the-context-manager-of-python-and-the-applications-in-tornado/


主要对象:
  - ioloop: http://www.jianshu.com/p/1ec9a1c2fbc9
  - iostream: http://www.kaka-ace.com/tornado-httpserver-how-does-it-work/
  - application:
    - http://www.chengxulvtu.com/2017/10/10/tornado%E9%AB%98%E6%95%88%E5%BC%80%E5%8F%91%E5%BF%85%E5%A4%87%E4%B9%8B%E6%BA%90%E7%A0%81%E8%AF%A6%E8%A7%A3.html
    - http://www.itdadao.com/articles/c15a1258978p0.html
  - RequestHandle: http://www.nowamagic.net/academy/detail/13321023

模板引擎:
  - http://monklof.com/post/3/
  - http://blog.csdn.net/yueguanghaidao/article/details/42061293
  - http://www.cnblogs.com/liaofeifight/p/4962216.html
  - https://www.yuanmas.com/info/9eaJGY6oy5.html


## 4. 使用示例
  - http://github.com/tornadoweb/tornado/tree/stable/demos

## 5. 核心问题：
1. @gen.coroutine, @gen.engine 的区别
2. [stream_request_body 使用](https://github.com/tornadoweb/tornado/tree/master/demos/file_upload/)
