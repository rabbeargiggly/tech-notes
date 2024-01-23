# nacos-client下线无效



### 问题背景

公司有一个 diamond 框架封装了 nacos

各个 rpc 服务里都引入了这个 diamond 的一系列 jar 包依赖



另外在 diamond 后台里就有 rpc 后台的一些基本功能，其中就有主机列表

![img](https://github.com/rabbeargiggly/tech-notes/blob/main/problem-solving/2024-01-02-nacos-client%E4%B8%8B%E7%BA%BF%E6%97%A0%E6%95%88/resources/1.png)

在主机列表中可以对该主机进行『下线』操作

![img](https://github.com/rabbeargiggly/tech-notes/blob/main/problem-solving/2024-01-02-nacos-client%E4%B8%8B%E7%BA%BF%E6%97%A0%E6%95%88/resources/2.png)

### 需求背景

这个主机列表查询的是 RUNNING 状态下的主机，如果对主机进行下线的话，那么在主机列表里就不再显示了

所以要开发一个上线功能，并展示 24h 内下线的主机



### 问题描述

主机下线时，为了保存 24h，我将主机信息添加到了缓存中，使用的 hash 结构

上线功能我是在 diamon 后台对应的前端 vue 项目中，添加了一个按钮，并添加了一些参数，下线按钮同样也添加了几个必要参数来保证唯一；在 diamond 后台这个服务进行主机上线时，先取缓存中的主机信息，然后调用 nacos-client 中 namingService 的 registerInstance() 

在代码都部署完毕后，点击下线和上线，功能都正常，但是再点击下线的时候，发现主机会在短时间内（最长5秒）就变成了运行状态，下线按钮无效了。。。。。。



### 问题排查

因为点击下线按钮后，显示下线成功，但会自动上线，就怀疑是不是 nacos-server 没下掉

arthas 监控后发现下线接口调用确实是成功的，nacos-server 返回了 ok

![img](https://github.com/rabbeargiggly/tech-notes/blob/main/problem-solving/2024-01-02-nacos-client%E4%B8%8B%E7%BA%BF%E6%97%A0%E6%95%88/resources/3.png)




说明不是 nacos-server 的问题

那么为什么会下线后自动上线呢，于是怀疑心跳的问题

用 arthas 监控心跳方法，发现持续有心跳，stopped 字段为 false

![img](https://github.com/rabbeargiggly/tech-notes/blob/main/problem-solving/2024-01-02-nacos-client%E4%B8%8B%E7%BA%BF%E6%97%A0%E6%95%88/resources/4.png)


难道是下线逻辑中，从心跳集合中删除心跳信息这一步，没有删除成功，所以才会一直有心跳？

于是用 arthas 监控 trace 这个函数，看看执行到了哪一步

![img](https://github.com/rabbeargiggly/tech-notes/blob/main/problem-solving/2024-01-02-nacos-client%E4%B8%8B%E7%BA%BF%E6%97%A0%E6%95%88/resources/5.png)

```java
trace com.alibaba.nacos.client.naming.beat.BeatReactor removeBeatInfo
```

可以看到执行到了80行就结束了

![img](https://github.com/rabbeargiggly/tech-notes/blob/main/problem-solving/2024-01-02-nacos-client%E4%B8%8B%E7%BA%BF%E6%97%A0%E6%95%88/resources/6.png)


**说明下线操作中，从心跳集合删除心跳信息，没删成功**

**因为心跳集合里不存在这个心跳信息**

那这个心跳是从哪里来的呢

我把服务重新部署了一遍，然后第一次点击『下线』按钮，并监控这行代码的执行情况

![img](https://github.com/rabbeargiggly/tech-notes/blob/main/problem-solving/2024-01-02-nacos-client%E4%B8%8B%E7%BA%BF%E6%97%A0%E6%95%88/resources/7.png)

发现确实执行了，然后也真的下线了，也没有心跳了

![img](https://github.com/rabbeargiggly/tech-notes/blob/main/problem-solving/2024-01-02-nacos-client%E4%B8%8B%E7%BA%BF%E6%97%A0%E6%95%88/resources/8.png)


接着我点击『上线』按钮，发现也正常上线了，心跳也开始了

 

然后，我再一次点击『下线』按钮，这次开始就不正常了，我发现下线后，最多处于下线状态5秒就自动变成了运行状态

这个5秒刚好是心跳间隔

 

那说明不管下不下线，这个心跳应该是一直都在的

 

因为这个异常心跳，是在部署后出现『上线』操作后，才出现的情况，所以我研究上线的逻辑

发现最终调用的是 com.alibaba.nacos.client.naming.NacosNamingService#registerInstance #191

在这个函数里，有添加心跳信息的逻辑

![img](https://github.com/rabbeargiggly/tech-notes/blob/main/problem-solving/2024-01-02-nacos-client%E4%B8%8B%E7%BA%BF%E6%97%A0%E6%95%88/resources/9.png)

于是我用 arthas 监控这个函数发现，在 diamond-admin 项目中的 nacos-client 包里，居然添加了 dk_account_provider 服务的心跳 ？？？

![img](https://github.com/rabbeargiggly/tech-notes/blob/main/problem-solving/2024-01-02-nacos-client%E4%B8%8B%E7%BA%BF%E6%97%A0%E6%95%88/resources/10.png)


后来 watch 了一下入参，发现还真是，至此就真相大白了

 

### 结论

**简单来说，A 服务里添加了 B 服务的心跳并持续发送，所以 B 服务下不掉**

下线操作是调用服务端的http接口

例如要下线的主机是 dk_account_provider 下的一台主机，会调用该主机的 ip:port/dm_rpc/shutdown，这个接口最后会在 dk_account_provider 项目下的 nacos-client 包中删除心跳信息后，再去 nacos-server 进行注销

但上线是在使用的 updateHostInfo 函数，是在 diamond-admin 中添加 dk_account_provider 的心跳，这显然是不合理的，而且这个心跳会一直存在，导致下线一直无效

![img](https://github.com/rabbeargiggly/tech-notes/blob/main/problem-solving/2024-01-02-nacos-client%E4%B8%8B%E7%BA%BF%E6%97%A0%E6%95%88/resources/11.png)



### 解决

![img](https://github.com/rabbeargiggly/tech-notes/blob/main/problem-solving/2024-01-02-nacos-client%E4%B8%8B%E7%BA%BF%E6%97%A0%E6%95%88/resources/12.png)
