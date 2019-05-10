---
title:  压力测试中的Go http Client 参数调优
---
原文： http://tleyden.github.io/blog/2016/11/21/tuning-the-go-http-client-library-for-load-testing/

在Go中使用压力测试工具时，遇到了有数万个socket 处于TIME_WAIT的状态。 
本文列出发生这种情况的几原因， 并给出了如何修复这种情况。 
<!-- more -->