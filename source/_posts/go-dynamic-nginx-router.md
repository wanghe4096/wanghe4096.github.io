---
title: (译) 基于Go的动态Nginx 路由
---

## 导读
在Nitro 中， 我们需要一款专业的负载均衡器。 经过一番研究之后，Mihai Todor和我构建了基于Nginx、Redis 协议和Go-based 请路由器解决方案，其中nginx负责所有繁重工作，路由器本身并不承载流量。 这个解决方案在过去一年的生产环境中运行顺畅。 以下是我们所做的工作以及我们为什么那样做。 
<!-- more -->
 

## Why
  我们正在构建的新服务将位于负载均衡池之后，负责执行代价很高的计算任务，正因如此，我们需要做本地缓存。 为了缓存优化， 我们想尝试将相同资源的请求发送到同一主机上（如果这台主机是可用的)

	解决这个问题有很多现有方案，以下是一个不完全的清单列表：

- 利用cookie 维护session 黏性(Stickiness)
- 利用Header 
- 基于源IP的黏性(Stickiness) 
- HTTP重定向到正确实例

这个服务页面载入时将会被触发多次， 因此出于性能的考虑， HTTP重定向方式并不可行。 如果所有的入站请求都通过同样的负载均衡器，那么剩下的几种解决方案都可以正常工作。 另一方面， 如果你的前端是一个负载均衡器池， 你需要能够在它们之间共享状态或实现复杂的路由逻辑。 我们对当前需要在负载均衡器之间需要共享状态的设计变更并没有兴趣，因此我们为这个服务选择了更复杂的路由逻辑。 

## 我们的架构
 了解一下我们的设计架构也许能够帮你更好的理解我们的意图。

我们拥有一组前端负载均衡器，这些服务的实例被部署在Mesos, 以便根据服务规模和资源可用性进行扩容或申缩(come and go ) 。 将主机和端口号列表放入负载均衡器中不是问题，这已经成为我们平台的核心。 
	 
因为一切都在Mesos上运行， 并且我们拥一种简单的方式定义和部署服务，所以 添加任何新服务都很简单。 
 
   在Mesos之上， 我们在每处都运行着基于gossip的Sidecar 来管理服务发现。 我们的前端负载均衡器是由Lyft 的Envoy组成 , 它背后由Sidecar的Envoy 集成支持。 这能满足大部分服务的需求。 Envoy 主机运行在专用实例上， 但所有的服务都根据需要， 在主机之间迁移，由Mesos 和 Sigualarity 调度器执行。 
 
  仍在考虑中的Mesos 服务节点将拥有基于磁盘的本地缓存。 

## 设计
看着这个问题我们下了决定，我们着实想要一种一致性哈稀环。 我们可以让节点根据需要进出(come and go ) ，只有那些节点所服务的请求才会被重新路由。 剩下的所有节点将继续服务于任何公开的会话。 我们可以很简单地通过Sidecar数据来支持一致性哈稀环 (你可以用Mesos 或k8s代替) 。 Sidecar 健康检查节点， 我们可以靠这些健康检查节点判断它们在Sidecar中是否工作正常。
 
然后，我们需要某种一致性哈稀方法将流量导入到正确的节点中。它需要接收每一个请求， 识别问题资源， 然后将请求传递给其他已经准备处理该资源的服务实例。 
 
当然， 资源识别可以简单的通过URL处理，并且任何负载均衡器能够将他们分开来处理简单的路由。 所以我们只需要将他们与一致性哈稀关联起来，对此我们已经有一种解决方案。 
 
你可以在nginx 用lua 那样做， 也可在HAproxy 中用lua 。 在Nitro 里， 我们没有一个人是Lua 专家，并且显然没有库能够实现我们的需要。 理想情况下， 路由逻辑将在Go中实现， Go在我们的技术栈中是一门关键语言并且得到了很好的支持。 
 
Nginx 有着丰富的生态环境， 跳脱常规的思路还引发了一些很有趣的nginx插件。 这些插件中首选插件Valery Kholodko的nginx-eval-module 。 这个插件允许你从nginx到一个端点生成一个调用，并且将返回的结果评估为nginx的变量。 在其他可能的作用中， 这个插件的意义在于它允许您动态地决定哪个端点应该接收代理传递。 这就是我们想要做的。 你从Ngnix到某个地方生成一个调用， 获取一个结果后， 你可以根据返回的结果值生成路由决策。 你可以使用HTTP服务实现该请求的接收方。 该服务仅返回目标服务器端点的主机名和端口号的字符串。 这个服务始终保持一致性哈希，并且告知Nginx 每个请求流量路由的位置 ， 但是生成一个单独的HTTP请求，仍然有些笨重。 整个预期的回复内容将会是字符串10.10.10.5:23453。 通过HTTP，我们会在两个方向传递头部信息，这将大大超出响应正文的大小。 
 
于是我开始研究Nginx支持的其他协议， 发现Memcache协议和Redis协议它都支持。其中，对Go服务最友好的支持是Redis协议。所以那就是我们改进的方向。Nginx 中有两个Redis模块。，有一个适合通过nginx-eval-module 使用。 实现Redis Go语言最好的库是Redeo。Rodeo实现了一个极其简单的处理机制，非常类似于go标准库中的http 包。 任何redis协议命令将会包含一个handler 函数，并且它的写法非常简单。 相比Nginx插件，它能够处理更新版本的redis协议。 于是， 我摒弃了我的C技能，并补充了Nginx插件以使用最新的Redis协议编码。
 


于是， 我们最新的解决方案是： 

```
[Internet] -> [Envoy] -> [Nginx] -(2)--> [Service endpoint]
                             \
                          (1) \ (redis proto)
                               \
                                -> [Go router]
```

这个调用从公网进入， 触发一个Envoy 节点， 然后到一个Nginx节点.Nginx 节点(1) 询问路由器将请求送至何处。 然后Nginx节点（2）将请求送至指定的服务端点。 

## 实现
我们在Go中建立了一个库来管理由Sidecar或Hashicorp的Memberlist库支持的一致性哈希。我们称之为Ringman库。然后，我们将该库强制接入Redeo库支持的Redis协议请求的服务中。
 
这种方案只需要两个Redis命令：GET和SELECT。我们选择实现一些用于调试的的命令，其中包括INFO，可以用您想要的任何服务器状态进行回复。在两个必需的命令中，我们可以放心地忽略SELECT，这是用由于选择Redis DB以用于任何后续调用。我们只接受它，什么也不做。GET让所有的工作都很容易实现。以下是通过Redis和Redeo为Ringman端点提供服务的完整功能。 Nginx会传递它接收到的URL，然后从哈希环中返回端点。

```go
srv.HandleFunc("get", func(out *redeo.Responder, req *redeo.Request) error {
 if len(req.Args) != 1 {
  return req.WrongNumberOfArgs()
 }
 node, err := ringman.GetNode(req.Args[0])
 if err != nil {
  log.Errorf("Error fetching key '%s': %s", req.Args[0], err)
  return err
 }

 out.WriteString(node)
 return nil
})
```

这是Nginx使用以下配置调用：

```bash
# NGiNX configuration for Go router proxy.
# Relies on the ngx_http_redis, nginx-eval modules,
# and http_stub_status modules.

error_log /dev/stderr;
pid       /tmp/nginx.pid;
daemon    off;

worker_processes 1;

events {
  worker_connections  1024;
}

http {
  access_log   /dev/stdout;

  include     mime.types;
  default_type  application/octet-stream;

  sendfile       off;
  keepalive_timeout  65;

  upstream redis_servers {
    keepalive 10;

    # Local (on-box) instance of our Go router
    server services.nitro.us:10109;
  }

  server {
    listen      8010;
    server_name localhost;

    resolver 127.0.0.1;

    # Grab the filename/path and then rewrite to /proxy. Can't do the
    # eval in this block because it can't handle a regex path.
    location ~* /documents/(.*) {
      set $key $1;

      rewrite ^ /proxy;
    }

    # Take the $key we set, do the Redis lookup and then set
    # $target_host as the return value. Finally, proxy_pass
    # to the URL formed from the pieces.
    location /proxy {
      eval $target_host {
        set $redis_key $key;
        redis_pass redis_servers;
      }

      #add_header "X-Debug-Proxy" "$uri -- $key -- $target_host";

      proxy_pass "http://$target_host/documents/$key?$args";
    }

    # Used to health check the service and to report basic statistics
    # on the current load of the proxy service.
    location ~ ^/(status|health)$ {
      stub_status on;
      access_log  off;
      allow 10.0.0.0/8;    # Allow anyone on private network
      allow 172.16.0.0/12; # Allow anyone on Docker bridge network
      allow 127.0.0.0/8;   # Allow localhost
      deny all;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
      root   html;
    }
  }
}
```

我们调用Nginx和容器里的路由，让他们在同样的host上运行，这样我们就可以在其中实现较低成本的调用。

以下是我们建立的Nginx:

```bash
./configure --add-module=plugins/nginx-eval-module \
     --add-module=plugins/ngx_http_redis \
     --with-cpu-opt=generic \
     --with-http_stub_status_module \
     --with-cc-opt="-static -static-libgcc" \
     --with-ld-opt="-static" \
     --with-cpu-opt=generic
 
make -j8
```

## 性能
我们在自有环境中进行了细致的性能测试， 我们看到，通过Redis协议从Nginx到Go路由器的平均响应时间大约为0.2-0.3ms。由于来自上游服务的响应时间的中值大约为70毫秒，所以这是可以忽略的延迟。
一个更复杂的Nginx配置大概能够做更复杂的错误处理。服务一年后的可靠性非常好，性能一直很稳定。

## 结束语
如果您有类似需求，则可以复用大部分组件。只需按照上面的链接到实际的源代码。如果您有兴趣直接向Ringman添加对K8或Mesos的支持，我们会非常欢迎。
 
这个解决方案听起来有点黑客，不过它最终成为我们基础设施的重要补充。希望它能帮助别人解决类似的问题。
 
该文由Karl Mathias 发布于2018年5月7日 