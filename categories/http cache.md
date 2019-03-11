# 一、HTTP缓存
已复制到印象笔记中

这次我们从两个角度来看看http的缓存：**缓存控制**和**缓存校验**。
**缓存控制**：控制缓存的开关，用于标识请求或访问中是否开启了缓存，使用了哪种缓存方式。
**缓存校验**：如何校验缓存，比如怎么定义缓存的有效期，怎么确保缓存是最新的。

## 缓存控制

> 这里主要说的是服务端的缓存控制

在http中，控制缓存开关的字段有两个：Pragma 和 Cache-Control。nginx配置伪代码：

```nginx
add_header Cache-Control public;
add_header Pragma public;
```



### 1.Pragma头域

- HTTP/1.0 协议，已逐渐废弃
- 如果一个**响应报文**中同时出现Pragma和Cache-Control时，以Pragma为准。
- 在HTTP/1.1协议中，它的含义和Cache- Control:no-cache相同

> 客户端浏览器控制 Pragma的值为no-cache时，表示禁用缓存

![mark](http://po5loud60.bkt.clouddn.com/blog/20190310/aGx4LL0aHRfQ.png?imageslim)




### 2.Cache-Control
#### (1) Cache-Control(Response) 

- http1.1 中加入的新属性

- 在响应中使用Cache-Control 时，它可选的值有

  ![mark](http://po5loud60.bkt.clouddn.com/blog/20190310/ii1LPPQj8gIj.png?imageslim)

max-age=**[seconds]**：比较特殊，是一个混合属性，替代了Expires的过期时间

eg: 

```bash
cache-control: max-age=0, private, must-revalidate
```

##### **no-store优先级最高**

在Cache-Control 中，这些值可以自由组合，多个值如果冲突时，也是有优先级的，而no-store优先级最高。我们可以测试下：在[nginx](https://www.baidu.com/s?wd=nginx&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)中做如下配置：

```nginx
server {
    listen 88;
    root /opt/ms;
    index index.php index.html index.htm index.nginx-debian.html;
    
    location ~* ^.+\.(css|js|txt|xml|swf|wav)$ {
        add_header Cache-Control no-store;
        add_header Cache-Control max-age=3600;
        add_header Cache-Control public;
        add_header Cache-Control only-if-cached;
        add_header Cache-Control no-cache;
        add_header Cache-Control must-revalidate;
    }
}
```

在/opt/ms下增加个文件type.css，内容如下：

```css
a{
    color: #000000;
}
a:focus,a:hover {
    text-decoration: none;
    color: #708090;
}
```

配置好之后，reload下nginx，在浏览器访问地址http://127.0.0.1:88/type.css，可以看到响应头部包含nginx配置中的字段：

![mark](http://po5loud60.bkt.clouddn.com/blog/20190310/oqWWLvLukC8g.png?imageslim)

重复刷新访问，会发现每次的状态码都是200，原因是no-store的优先级最高，本地不保存，每次都需要服务器发送资源。

##### **public和private的选择**

如果你用了CDN，你需要关注下这个值。CDN厂商一般会要求cache-control的值为public，提升缓存命中率。如果你的缓存命中率很低，而访问量很大的话，可以看下是不是设置了private，no-cache这类的值。如果定义了max-age，可以不用再定义public，它们的意义是一样的。

##### **哪里设置Cache-Control**

在nginx中有个很常见的配置：

```nginx
location ~* ^.+\.(ico|gif|jpg|jpeg|png)$ { 
            expires      30d;
	}
```

这个指令等同于cache-control: max-age=2592000，同时你会在响应头部看到一个etag字段，这是由于nginx默认开启，如果要关闭可以增加个配置`etag off`。这个etag就是我们接下要看的缓存校验字段。



#### (2) Cache-Control(Request)

在**客户端**请求中使用Cache-Control 时，它可选的值有

![mark](http://po5loud60.bkt.clouddn.com/blog/20190310/mOVKtuSjDYlM.png?imageslim)

### 3.expires

http1.0时期的属性 ，现在默认浏览器均默认使用HTTP 1.1，所以它的作用基本忽略。

expires 的一个缺点就是，返回的到期时间是服务器端的时间，这样存在一个问题，如果客户端的时间与服务器的时间相差很大（比如时钟不同步，或者跨时区），那么误差就很大，所以在HTTP 1.1版开始，使用Cache-Control: max-age=秒替代。

nginx配置：

```nginx
location ~* \.(gif|jpg|jpeg|png) {
root  /var/mywww/html/public/
expires 3d;
}
# 上面表示，网站上所有的用正则匹配（不区分大小写） 所有以gif,jpg,png,jpeg结尾的文件，把它们放入客户端的缓存，3天不失效
```

![mark](http://po5loud60.bkt.clouddn.com/blog/20190310/qDVc1PImLLdE.png?imageslim)





## 缓存校验

> 缓存校验主要在浏览器端

### 1.Last-Modified 

> 位于Response Headers

- 表示文档的最后修改时间，当去服务器验证时会用到这个时间
- http1.0时期属性， 现在仍在使用

服务端在返回资源时，会将该资源的最后更改时间通过Last-Modified字段返回给客户端。客户端下次请求时通过If-Modified-Since或者If-Unmodified-Since带上Last-Modified，服务端检查该时间是否与服务器的最后修改时间一致：如果一致，则返回304状态码，不返回资源；如果不一致则返回200和修改后的资源，并带上新的时间。

**If-Modified-Since和If-Unmodified-Since的区别是：**

- If-Modified-Since：告诉服务器如果时间一致，返回状态码304
- If-Unmodified-Since：告诉服务器如果时间不一致，返回状态码412



### 2.ETag

>  位于Response Headers

- http1.1时期新加属性
- 因为以下几个原因增加此属性：
  - 某些服务器不能精确得到文件的最后修改时间， 这样就无法通过最后修改时间来判断文件是否更新了。
  - 某些文件的修改非常频繁，在秒以下的时间内进行修改. Last-Modified只能精确到秒。
  - 一些文件的最后修改时间改变了，但是内容并未改变。 我们不希望客户端认为这个文件修改了。



**etag的方式是这样**

服务器通过某个算法对资源进行计算，取得一串值(类似于文件的md5值)，之后将该值通过etag返回给客户端，客户端下次请求时通过If-None-Match或If-Match带上该值，服务器对该值进行对比校验：如果一致则不要返回资源。

**If-None-Match和If-Match的区别是：**

- If-None-Match：告诉服务器如果一致，返回状态码304，不一致则返回资源
- If-Match：告诉服务器如果不一致，返回状态码412



> 总结：
>
> 当服务端Last-Modified与ETag同时使用时，浏览器在验证时会同时发送If-Modified-Since和If-None-Match，按照http/1.1规范，如果同时使用If-Modified-Since和If-None-Match则服务端必须两个都验证通过后才能返回304；且nginx就是这样做的。因此实际使用时应该根据实际情况选择。
>
> 分布式的Web系统中，当访问落在不同的物理机上时会返回不同的ETag，进而导致304失效，降级为200请求,所以经常在使用中会关闭ETag。



### 3.expires

> 在Response Headers中
>
> 根据规范定义 Cache-Control 优先级大于expires。实际使用可以2个都用，或者仅使用Cache-Control 。一般情况下expires = 当前系统时间+ 缓存时间（cache-control: max-age）

```bash
设置客户端不缓存，并兼容http1.0的方式
Pragma : no-cache 
Expires：0
Cache-Control：no-store
等价
Pragma : no-cache  // Pragma为了兼容http1.0
Cache-Control：max-age=0  // 去掉了Expires属性，合并到max-age中
```

**缓存命中优先级:Cache-Control( http1.1) > Expires > Pragma(http1.0)来决定是否 (200 from cache)**

### 5.总结

缓存开关是： pragma， cache-control(Expires)。

缓存校验有：Expires，Last-Modified，etag。

从状态码的角度来看，它们的关系如下图：

![mark](http://po5loud60.bkt.clouddn.com/blog/20190310/vle1kz3r0GSB.png?imageslim)

cache-control的各个值关系如下图

![mark](http://po5loud60.bkt.clouddn.com/blog/20190310/r10STJ3Oa6cI.png?imageslim)



# 二、缓存场景

### 0.用户的行为对缓存的影响

![mark](http://po5loud60.bkt.clouddn.com/blog/20190310/r8m3IGR32pUy.png)

### 1.F5刷新

访问 http://res15.iblimg.com/respc-1/resources/v4.2/unit/jquery-1.8.2.min.js ，发送的Request Headers如下

```bash
# Request Headers
DNT: 1
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/71.0.3578.80 Safari/537.36

# Response Headers
Access-Control-Allow-Origin: *
Age: 297
Ali-Swift-Global-Savetime: 1551560462
Cache-Control: max-age=120
Connection: keep-alive
Content-Encoding: gzip
Content-Length: 34827
Content-Type: application/x-javascript
Date: Sat, 09 Mar 2019 14:22:56 GMT
EagleId: da5ed29515521416737682619e
Expires: Sat, 09 Mar 2019 14:24:56 GMT
Last-Modified: Thu, 06 Apr 2017 16:14:50 GMT
Server: Tengine
Timing-Allow-Origin: *
Vary: Accept-Encoding
Via: cache12.l2et15[12,304-0,H], cache11.l2et15[13,0], kunlun4.cn1259[0,200-0,H], kunlun1.cn1259[2,0]
x-amz-request-id: tx000000000000000408ebe-005c83b199-3146b931-default
X-BY: BLB-4202a35d1817
X-Cache: HIT TCP_HIT dirn:0:270223571
X-Swift-CacheTime: 300
X-Swift-SaveTime: Sat, 09 Mar 2019 14:22:56 GMT
```






**再按下F5刷新**，发送的Request Headers如下：

```bash
# General
Request URL: http://res15.iblimg.com/respc-1/resources/v4.2/unit/jquery-1.8.2.min.js
Request Method: GET
Status Code: 304 Not Modified
Remote Address: 218.94.210.116:80
Referrer Policy: no-referrer-when-downgrade

# Request Headers
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cache-Control: max-age=0
Connection: keep-alive
DNT: 1
Host: res15.iblimg.com
If-Modified-Since: Thu, 06 Apr 2017 16:14:50 GMT
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/71.0.3578.80 Safari/537.36

# Response Headers
Access-Control-Allow-Origin: *
Age: 18
Ali-Swift-Global-Savetime: 1551560462
Cache-Control: max-age=120
Connection: keep-alive
Content-Encoding: gzip
Content-Type: application/x-javascript
Date: Sat, 09 Mar 2019 14:28:00 GMT
EagleId: da5ed29515521416982554427e
Expires: Sat, 09 Mar 2019 14:30:00 GMT
Last-Modified: Thu, 06 Apr 2017 16:14:50 GMT
Server: Tengine
Timing-Allow-Origin: *
Vary: Accept-Encoding
Via: cache12.l2et15[13,304-0,H], cache4.l2et15[20,0], kunlun4.cn1259[0,304-0,H], kunlun1.cn1259[3,0]
x-amz-request-id: tx000000000000000408ebe-005c83b199-3146b931-default
X-BY: BLB-4202a35d1817
X-Cache: HIT TCP_IMS_HIT dirn:0:270223571
```



发现Request Headers 多了

```
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cache-Control: max-age=0
Connection: keep-alive
Host: res15.iblimg.com
If-Modified-Since: Thu, 06 Apr 2017 16:14:50 GMT
```

If-Modified-Since 是发送上次请求的Last-Modified，和服务器验证内容是否发生了变更。

响应码为304，表示服务器告知浏览器，你缓存的内容没有发生变化，可以继续使用。



**其他补充**
- 【Age:18】  一般用于缓存代理层cdn，表示内容在缓存代理层从创建到现在存活了多少时间(s)
- 【vary： Accept-Encoding】用于通知缓存服务器，对相同url有着不同版本的响应，比如压缩和非压缩。这里指定头部为 `Accept-Encoding`, 则cdn需要根据请求头中的`Accept-Encoding`来判断不同版本的缓存内容。例如上面面案例，F5刷新后，发送的`Accept-Encoding: gzip, deflate`，则返回内容经过压缩。
- 【Content-Encoding】值
    -  gzip　　表明实体采用GNU zip编码
    - compress 表明实体采用Unix的文件压缩程序
    - deflate　　表明实体是用zlib的格式压缩的
    - identity　　表明没有对实体进行编码。当没有Content-Encoding header时， 就默认为这种情况
    - gzip, compress, 以及deflate编码都是无损压缩算法，用于减少传输报文的大小，不会导致信息损失。 其中gzip通常效率最高， 使用最为广泛。
- via:一般用于代理层，表示访问最终网络经过了哪些代理层，用了什么协议
- X-Cache: HIT TCP_IMS_HIT dirn:0:270223571  是否命中
- Expires的值是一个GMT时间，表示该缓存的有效时间。
- Cache-Control 相关字段




### 2. ctrl+F5 强制刷新

作用：强制从服务器获取最新内容

```
# ctrl+F5的Request Headers
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cache-Control: no-cache
Connection: keep-alive
DNT: 1
Host: res15.iblimg.com
Pragma: no-cache
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/71.0.3578.80 Safari/537.36
```

发现多了：`Cache-Control: no-cache 和 Pragma: no-cache`。这个属于浏览器端缓存校验控制，表示禁用本地缓存，浏览器会向server请求资源





强制刷新效果和谷歌浏览器 “Disable cache” 效果一样
![mark](http://po5loud60.bkt.clouddn.com/blog/20190310/xkI3p0NO9XRo.png?imageslim)



### 3.from cache

正常F5或者ctrl+f5都会触发去服务器检验内容是否变化。**什么情况下不去服务器检验内容？就是from cache的时候。**那么怎么发生的吗？

答案就是：从A页面跳转到A页面



from memory cache：不访问服务器，直接读缓存，从内存中读取缓存。此时的数据时缓存到内存中的，当kill进程后，也就是浏览器关闭以后，数据将不存在。但是这种方式只能缓存派生资源。



from disk cache：不访问服务器，直接读缓存，从磁盘中读取缓存，当kill进程时，数据还是存在。因为是存在硬盘当中的，下次打开仍会from disk cache。这种方式也只能缓存派生资源


![mark](http://po5loud60.bkt.clouddn.com/blog/20190310/zwtf8PI2nYca.png?imageslim)

