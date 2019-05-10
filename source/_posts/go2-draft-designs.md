---
title: Go 2 设计草案(译)
---
原文： [Go 2 draft Designs ](https://go.googlesource.com/proposal/+/master/design/go2draft.md )

![fff](/fiveyears.jpg)
作为Go 2设计过程五的一部分， 我们发布了这些设计草案，并在社区开始讨论关于以下三个主题： 
- 泛型 
- 错误处理
- 错误值语议

这些草案并非提案流程中的提案。 它们作为发起计论的起点， 最终目的是能够产生足好的设计，以便转化实际可落地的提案。  
<!-- more -->

每个设计草案都附加“问题概述”（类似于信件封面) . 问题概述指在提供草案设计的背景(上下文); 当然也是设计细节； 为实际设计文档提供一个一个平台，并为设计和构建提供框切实可行的框架。  它介绍了背景、目标、非目标、设计约束、一个设计的概要，以及我们认为最需要关注的领域的简短讨论， 以及与之前方法的比较。 

再次申明， 这些只是设计草案，并非官方意义上的提案。 没有相关提案问题。 我们稀望所有Go用户帮助我们改进草案，并使期最终转化为Go提案。 我们已经wiki页面用来收集和组织关于每个主题的反馈。 请帮助我们更新这些页面， 包括添加指向您自己反馈的链接。 


# 错误处理: 
- [概览](https://go.googlesource.com/proposal/+/master/design/go2draft-error-handling-overview.md)
- [设计草案](https://go.googlesource.com/proposal/+/master/design/go2draft-error-handling.md)
- [wiki 反馈页](https://golang.org/wiki/Go2ErrorHandlingFeedback)


# 错误值：
- [概览](https://go.googlesource.com/proposal/+/master/design/go2draft-error-values-overview.md)
- [错误检查设计草案](https://go.googlesource.com/proposal/+/master/design/go2draft-error-inspection.md)
- [错误打印设计草案](https://go.googlesource.com/proposal/+/master/design/go2draft-error-printing.md)
- [wiki 反馈页](https://golang.org/wiki/Go2ErrorValuesFeedback)

# 泛型 
- [概览](https://go.googlesource.com/proposal/+/master/design/go2draft-generics-overview.md)
- [设计草案](https://go.googlesource.com/proposal/+/master/design/go2draft-contracts.md)
- [wiki 反馈页](https://golang.org/wiki/Go2GenericsFeedback)

