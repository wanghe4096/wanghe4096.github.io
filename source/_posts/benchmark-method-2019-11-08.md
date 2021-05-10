---
title: 性能分析与优化的一般套路
date: 2019-11-08
---
![](/images/tested.jpg)


#压测性能瓶颈分析的一般套路

## 计算实例CPU很高
1. 代码中存在加密、解密、编码、解码，如何压缩、解压缩、json序列化和反序列化,会非常吃CPU资源  
尽可能去做本地对象缓存可以避免

## 内存很高
1. 代码中内存泄漏，可能是频繁申请内存， 垃圾回收不及时，或者是操作系统没有来得及回内存
利用go pprof 工具排查， 是否存在频繁调用new, make, 或者初始大对象的问题 ， 这里只能扣代码细节 


## 网络情况
 如果计算实例cpu不高，内存也不高，但是并发就是没有上去
1. 先排除施加压力的机器或程序是否给予足够的压力，比如施压链路中间有瓶颈， 或者施压机器性能不足，或者施压的并发数、连接数设置的太小 
2. 下游服务的性能瓶颈（可以用火焰图分析) ，比如依赖的业务服务或者是 数据库的瓶颈
3. 网络的情况如何，例如计算实例的连接数太高或者太低，或者是带宽达到及限（这种情况一般取决请求报文与反回报文的大小) 

类型情况排查细节比较多， 大多数常见问题可能是由于：
数据库连接池设置的太高（数据库会报错，too many open files ) 
数据库连接池设置的太低( 不报错， 数据库没有压力， 但是QPS上不去， 计算实例资源占用也不高，RT比较高)
数据库有时候也会报 invalid connection ，  这种情况可能是数据有问题， 也有可能是连接池使用了一个释放的连接句柄(可以理解成socket 套接字)) ， 如果不是数据库问题，一般我们用的数据库驱动都重新获取一个连接句柄继续
请求下游业务比较慢，超时时间设置的也比较大（可能也会报 too many open files )， 这类情况，如果下游业务可以降级，超时时间不设置的太在，不要超过1s ，一般情况下， 一个接口RT 在100ms以内还算可以接收,大于300ms基本上就是慢接口了

<!-- more -->