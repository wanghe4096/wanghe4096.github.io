---
title: 在Golang API 中避免内存泄漏
---
原文： https://hackernoon.com/avoiding-memory-leak-in-golang-api-1843ef45fca8

在你向生产环境发布你的Golang API之前， 你一定要读这篇文章。  根据在Kurio的真实案例，由于我们没有采用正确的方法，因此我们每次上线发布都变得极为艰难。 


几个星期以前，   我们正忙于在我们的主要服务中，修复一个离奇且难以发现的BUG 。 我尝试了许多办法调试和修复它。 这个BUG并非业务逻辑上的bug。 在生产环境上运行了几个星期。 由于我们的自动申缩机制的补救， 它看起来仍然运行良好。  

直到最后， 我们发现是我们的代码中没有正确的处理。  
<!-- more -->

# 架构
仅供参考， 我们采用微服务架构模式。 我们拥有一个统一的api网关。 我们称之为核心API， 它直接为我们的用户提供服务。 他的作用类似于api网关。 它的任务仅仅是处理用户请求，转发给需要的服务， 并且构建响应给用户。 这个 main API 完全由Go语言编写。 选择Golang的理由是另一个故事，这里不讨论。 

如果画出架构图， 我们的系统看起来是下面的样子： 
![Kurio architecture](https://cdn-images-1.medium.com/max/1600/1*b8KqwAtasbTzt96hYntjMQ.png)

# 问题
我们花了非常多的时间和精力来使用我们的main API, 但它频烦的荡机并且返回长的响应给我们的移动APP，有时会导致我们的API不能访问。 当我们的API大盘变红时， 这是一件非常危险的事情，给工程师很大压力，让我们的工程师变得焦头烂额。 

另外一个问题是， 我们的CPU和内存负载越来越高。 如果出现这样的问题我们只能手动重启并直到等它再次运行。 


![Our API response time up-to 86 seconds for a single request.](https://cdn-images-1.medium.com/max/1600/1*bqsD1tLb4gAvHw4kJOnvjA.jpeg)
Our API response time up-to 86 seconds for a single request.


![graph our API response time, and doing restart manually for safety.](https://cdn-images-1.medium.com/max/1600/1*hw-Mdt7ctshVu2SbsM4Vhw.png)
graph our API response time, and doing restart manually for safety.

这个BUG让我们的感到非常难受， 因为我们没有任何关于这个BUG的日志和关于这个BUG更多的精确的提示。 我们仅仅知道接口的响应时间变的很长。 CPU 和内存使用量持续增长。 这个BUG就像一个噩梦。 

# 阶段1： 使用自定义的http.Client
当我们开发这个服务的过程中， 我们真切的学到了这点。 那就是不要相信默认http.Client的配置 。 

我们使用自定义的 http.Client 来替代从http 包中默认的参数配置。 

```Go
client := http.Client{} //default 
```

我们根据我们的需要引入了一些配置。 因为我们需连接复用，因此我们在transport中加了一些配置并且控制maxidle连接复用。 

```Go
keepAliveTimeout:= 600 * time.Second
timeout:= 2 * time.Second
defaultTransport := &http.Transport{
    Dial: (&net.Dialer{
                     KeepAlive: keepAliveTimeout,}
           ).Dial,
    MaxIdleConns: 100,
    MaxIdleConnsPerHost: 100,
}
client:= &http.Client{
           Transport: defaultTransport,
           Timeout:   timeout,
}
```

这个配置帮我们降低了调用其他服务的最大开销时间。 

# 阶段2： 避免由于未关闭响应体导致的内存泄漏

这个阶段我们了解到： 如果我们想要将连接池复用的其他服务，我们必须在读完响应体后关闭它。

我们的 main API 仅仅是调用其他服务，我们犯了一个致命的错误。 由于我们的 main API 认为不论什么情况从http.Client可用连接都是可以复用的。 我们必须读响应体，即使我们不需要它。我也必须关闭响应体。  这两个被用于避免我们的服务器内存泄漏。 

在我们的代码中， 我们的忘了关闭我们的响应体（response.Body)。这点在我们的生产环境能导致巨大的灾难。  

解决方案是： 我关闭响应体并且读它尽管我们不需这个数据。下面是修复后的代码： 

```Go
req, err:= http.NewRequest("GET","http://example.com?q=one",nil)
if err != nil {
  return err
}
resp, err:= client.Do(req)
//=================================================
// CLOSE THE RESPONSE BODY
//=================================================
if resp != nil {
    defer resp.Body.Close() // MUST CLOSED THIS 
}
if err != nil {
  return err
}
//=================================================
// READ THE BODY EVEN THE DATA IS NOT IMPORTANT
// THIS MUST TO DO, TO AVOID MEMORY LEAK WHEN REUSING HTTP 
// CONNECTION
//=================================================
_, err = io.Copy(ioutil.Discard, resp.Body) // WE READ THE BODY
if err != nil { 
   return err
}
```

在我们读完这篇文章后，修复这个问题： 
- http://devs.cloudimmunity.com/gotchas-and-common-mistakes-in-go-golang

- http://tleyden.github.io/blog/2016/11/21/tuning-the-go-http-client-library-for-load-testing/

第一阶段和第二阶段帮在自动申缩的帮助下成功减少这个BUG出现的次数。 很好， 事实上从2017.3月以来，这个BUG再没有出现。 

# 阶段3： 在Golang channel 中做超时控制

稳定运行了几个月之后。 这个BUG再次出现。 在2018二月的第一个星期。 我们其中一个服务被main API 所调用。 然后了一个错误: content service 服务被踢掉。 原因是它无法访问。 

所以当我们的 "content service" 被踢掉， 我们的 "main API" 再次着火。 API仪表盘在次变红。 API 响应时间变得更高更高慢。 我们的CPU 和内存又次彪升，即使们使用了自动扩容。 

又一次， 我们努力想找到问题根源。 好的， 我们重启这个 "content service" ，他在一次运行良好。 

在那个过程中， 我们非常好奇。 为什么再一次出现这个问题。 因为我们认为， 我们已经在http.Client 中设置了超时deadline, 在这种情况下， 它将永远不可能发生。 
我们仔细研究了我们的代码中潜在的问题， 然后我们发现了一些非常危险的代码。 

简化后的代码，看看起来下面的样子： 
*ps: 这个函数只是一个例子， 但于我们的问题比类似

```Go
type sampleChannel struct{
  Data *Sample
  Err error
}

func (u *usecase) GetSample(id int64, someparam string, anotherParam string) ([]*Sample, error) {

	chanSample := make(chan sampleChannel, 3)
	wg := sync.WaitGroup{}

	wg.Add(1)
	go func() {
		defer wg.Done()
       		chanSample <- u.getDataFromGoogle(id, anotherParam) // just example of function
	}()

	wg.Add(1)
	go func() {
		defer wg.Done()
    		chanSample <- u.getDataFromFacebook(id, anotherParam)
	}()

	wg.Add(1)
	go func() {
		defer wg.Done()
    		chanSample <- u.getDataFromTwitter(id,anotherParam)
	}()

	wg.Wait()
	close(chanSample)

	result := make([]*Sample, 0)

	for sampleItem := range chanSample {
		if sampleItem.Error != nil {
			logrus.Error(sampleItem.Err)
		}
		if sampleItem.Data == nil { 
			continue
		}
         result = append(result, sampleItem.Data)
	}


	return result
}
```

如果我们看上面的代码， 他看起来没有什么问题。 但是这个函数被大量的调用。 在我们的main API 压力巨大。 因为这个函数会用巨大的处理做3个API调用。 
我们在channel中使用超时控制的方法来改进这个它。 因为使用上面的样式代码-使用WaitGroup 会等待所有的进程完成-我们必须等所有的api调用必须完成。  所以我们才可以处理并且返回响应给用户。 

这是我们犯的严重错误的其中之一。 当我们其中一个服务挂了以后，这段代码将导致一个场巨大的灾难。  因为那将要等待很长的时间直到挂掉的服务被重新拉起。  在每秒5k的调用中这当然是一场灾难。 

## 首次尝试解决方案： 

我们用增加超时的办法修改它。 所以我们的用户将不用在等待很长时间。 它们仅仅是获得一个服务器内部错误。 

```Go
func (u *usecase) GetSample(id int64, someparam string, anotherParam string) ([]*Sample, error) {
	chanSample := make(chan sampleChannel, 3)
	defer close(chanSample)
 	go func() {
       		chanSample <- u.getDataFromGoogle(id, anotherParam) // just example of function
	}()

 
	go func() {
    		chanSample <- u.getDataFromFacebook(id, anotherParam)
	}()

   
	go func() {
   		chanSample <- u.getDataFromTwitter(id,anotherParam)
	}()
 

	result := make([]*feed.Feed, 0)
	timeout := time.After(time.Second * 2)
		for loop := 0; loop < 3; loop++ {
			select {
			case sampleItem := <-chanSample:
				if sampleItem.Err != nil {
					logrus.Error(sampleItem.Err)
					continue
				}
				if feedItem.Data == nil {
					continue
				}
				 result = append(result,sampleItem.Data)
			case <-timeout:
			      err := fmt.Errorf("Timeout to get sample id: %d. ", id)
			      result = make([]*sample, 0)
			      return result, err
			}
		}

	return result, nil;
}

```

# 阶段4： 使用Contest 做超时控制

在做完第3阶段后， 我们的问题仍然没有完全被解决。 我们的main API 仍在消耗着很高的CPU和内存。这是因为，即使我们已经将服务器内部错误返回给我们的用户， 但我们的goroutine 仍然存在。 我们想要的是， 如果我们已经返回了响应， 那么所有相应的资源也要一同清理，没有例外， 包括在后台运行的goroutine 和api调用。 

之后， 我们读了这篇文章 ： 

http://dahernan.github.io/2015/02/04/context-and-cancellation-of-goroutines/

我们发现了一些在golang中还没有意识的到有趣的特性。 那就是使用Context 来帮助我们取消go-routine 。 

我们使用context.Context 来代替time.After 做超时处理。 使用这种新的方法， 使们的服务变得更加稳定。 

```Go
func (u *usecase) GetSample(c context.Context, id int64, someparam string, anotherParam string) ([]*Sample, error) {
	
  if c== nil {
    c= context.Background()
  }
  
  ctx, cancel := context.WithTimeout(c, time.Second * 2)
  defer cancel()
  
  chanSample := make(chan sampleChannel, 3)
  defer close(chanSample)
 	go func() {
       		chanSample <- u.getDataFromGoogle(ctx, id, anotherParam) // just example of function
	}()

 
	go func() {
    		chanSample <- u.getDataFromFacebook(ctx, id, anotherParam)
	}()

   
	go func() {
   		chanSample <- u.getDataFromTwitter(ctx, id,anotherParam)
	}()
 

	result := make([]*feed.Feed, 0)
	for loop := 0; loop < 3; loop++ {
	  select {
		case sampleItem := <-chanSample:
			if sampleItem.Err != nil {
				continue
			}
			if feedItem.Data == nil {
				continue
			}
			 result = append(result,sampleItem.Data)
		  // ============================================================
		  // CATCH IF THE CONTEXT ALREADY EXCEEDED THE TIMEOUT
		  // FOR AVOID INCONSISTENT DATA, WE JUST SENT EMPTY ARRAY TO 
		  // USER AND ERROR MESSAGE
		  // ============================================================
      		case <-ctx.Done(): // To get the notify signal that the context already exceeded the timeout
		      err := fmt.Errorf("Timeout to get sample id: %d. ", id)
		      result = make([]*sample, 0)
		      return result, err
		}
	}

	return result, nil;
}
```

所以我们在我们的代码中每个gorotuine 调用的时都使用了Context. 帮我们释放内存并帮助我们终止goroutine调用。 

总之， 为了更可控更可靠，我们也在我们的http请求中传入context . 


```Go
func ( u *usecase) getDataFromFacebook(ctx context.Context, id int64, param string) sampleChanel{
 
  req,err := http.NewRequest("GET","https://facebook.com",nil)
  if err != nil {
    return sampleChannel{
      Err: err,
    }
  }
  // ============================================================
  // THEN WE PASS THE CONTEXT TO OUR REQUEST.
  // THIS FEATURE CAN BE USED FROM GO 1.7
  // ============================================================
  if ctx != nil {
    req = req.WithContext(ctx) // NOTICE THIS. WE ARE USING CONTEXT TO OUR HTTP CALL REQUEST
  }
  resp, err:= u.httpClient.Do(req)
  if err != nil {
    return sampleChannel{
      Err: err,
    }
  }
  body,err:= ioutils.ReadAll(resp.Body)
  if err!= nil {
    return sampleChannel{
      Err:err,
    }
    sample:= new(Sample)
    err:= json.Unmarshall(body,&sample)
    if err != nil {
      return sampleChannle{
        Err:err,
      }
    }
    return sampleChannel{
      Err:nil,
      Data:sample,
    }
  }
```

通过这些设置和控制控制，我们的系统更安全可控。 

# 我们学到的

- 在生产环境中，任何时候都不要使用默认的配置 

只是不要使用默认的配置 。 如果你在正在构建一个高并发服务。 请不要使用默认配置 。 

- 多读、多试、多失败反复尝试。  
我们这些经验中学到了很多。 这些经验只能在真实的案例和真实用户场景收益。 在修复这个BUG的过程中，我很高兴能参与其。 

# 参考阅读
- http://dahernan.github.io/2015/02/04/context-and-cancellation-of-goroutines/
- http://devs.cloudimmunity.com/gotchas-and-common-mistakes-in-go-golang/index.html#close_http_resp_body
- http://tleyden.github.io/blog/2016/11/21/tuning-the-go-http-client-library-for-load-testing/