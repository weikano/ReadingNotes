### 7.9 控制缓存的能力
#### 7.9.1 no-Store与no-Cache
> no-cache：禁止缓存对响应进行复制。缓存通常会像非缓存代理服务器一样，向客户端转发一条no-store响应，然后删除对象
>
> no-cache：可以存储在本地缓存中。只是在与原始服务器进行新鲜度验证之前，缓存不能提供给客户端使用
>
> HTTP/1.1中提供了Pragma: no-cache首部

#### 7.9.2 max-age response header
> Cache-Control: max-age=3600表示3600秒后过期

#### 7.9.3 Expires response header
> 不推荐使用，指定的是实际的过期日期而不是相对秒数。

#### 7.9.4 must-revalidate response header
> Cache-Control: must-revalidate：在事先没有跟原始服务器进行再验证的情况下，不能提供这个缓存对象的陈旧副本。

#### 7.9.5 试探性过期
#### 7.9.6 客户端的新鲜度限制
> ![image](https://raw.githubusercontent.com/weikano/NoteResources/master/HTTP-Guide/5.png)