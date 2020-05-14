---
layout: post
title: "golang-1.11-modules-再见GoPATH"
subtitle: 'Golang技术迭代'
author: "Elvis"
header-style: text
tags:
  - Golang
  - 技术学习
---

### 首先先来个小例子说明问题：

**准备工作**  
1、升级golang 版本到 1.12 Go下载  
2、添加环境变量 GO111MODULE 为 on 或者auto  
3、GO111MODULE=auto  
准备完毕，非常简单吧！！

#### 首先尝试创建一个hello项目演示GoModule

 首先，在$GOPATH/src路径外的你喜欢的地方创建一个目录，cd 进入目录，新建一个hello.go文件，内容如下

```
package main

import "github.com/astaxie/beego"
import "mytest v0.0.0"

func main() {
    beego.Run()
}

```

按照过去的做法，要运行hello.go需要执行go get 命令 下载beego包到 $GOPATH/src

**但是，使用了新的包管理就不在需要这样做了**

先初始化模块 
>go mod init

再直接
>go run hello.go

稍等片刻… go 会自动查找代码中的包，下载依赖包，并且把具体的依赖关系和版本写入到go.mod和go.sum文件中。
查看go.mod，它会变成这样：

```
module hello

go 1.12

require github.com/astaxie/beego v1.11.1

```
*go.mod*文件的*require*关键字是引用，后面是包，最后*v1.11.1*是引用的版本号，然后*replace*处是替换本地依赖包(路径要处理妥当)的引用


**下面分析下GO-1.11版本主要的更新：**  

### 之前版本的诟病

1.在不使用额外的工具的情况下，Go的依赖包需要手工下载  
2.第三方包没有版本的概念，如果第三方包的作者做了不兼容升级，会让开发者很难受  
3.协作开发时，需要统一各个开发成员本地$GOPATH/src下的依赖包  
4.引用的包引用了已经转移的包，而作者没改的话，需要自己修改引用。  
5.第三方包和自己的包的源码都在src下，很混乱。对于混合技术栈的项目来说，目录的存放会有一些问题  
6.众所周知的原因，类似于 golang.org/x/... 的包会出现下载失败的情况。  

### Modules解决问题
1.自动下载依赖包  
2.项目不必放在GOPATH/src内了  
3.项目内会生成一个go.mod文件，列出包依赖  
4.所以来的第三方包会准确的指定版本号  
5.对于已经转移的包，可以用replace 申明替换，不需要改代码  
6.新增GOPROXY解决代理镜像问题  



**Modules是如何解决上文的诟病问题的。**


#### 问题一：依赖的包自动下载到哪里了

使用Go的包管理方式，依赖的第三方包被下载到了$GOPATH/pkg/mod路径下。

如果你成功运行了上面示例，可以在您的$GOPATH/pkg/mod 下找到一个这样的包 **github.com/astaxie/beego@v1.11.1**

####  问题二： 依赖包的版本是怎么控制的

在上一个问题里，可以看到最终下载在  $GOPATH/pkg/mod 下的包 github.com/astaxie/beego@v1.11.1 最后会有一个版本号 *1.11.1*，
也就是说，$GOPATH/pkg/mod里可以保存相同包的不同版本。

版本是在**go.mod**中指定的。

如果，在**go.mod**中没有指定，**go**命令会自动下载代码中的依赖的最新版本，本例就是自动下载最新的版本。

如果，在**go.mod**用**require**语句指定包和版本 ，go命令会根据指定的路径和版本下载包，
指定版本时可以用**latest**，这样它会自动下载指定包的最新版本；

版本号格式为 **vn.n.n** (n代表数字)
查看版本号命令

```bash
go list -m -versions github.com/astaxie/beego
```

#### 问题三： 协作开发是如何统一开发的包的

**维护统一的go.mod**

只需要项目维持统一的go.mod文件，每一次项目的加载将会根据go.mad文件中加载所需要的依赖到本地中。（所以一般都会将go.mod文件上传到git）

go.mod主要两个关键字，*require*关键字是引用，后面是包，最后*v1.11.1*是引用的版本号（一般是从网络自动下载的包），然后*replace*处是替换本地依赖包(路径要处理妥当)的引用


**go mod 命令如下图**

![](/img/in-post/gomod_command.png)



####  问题四：引用的包引用了已经转移的包，而作者没改的话，需要自己修改引用问题。

在go快速发展的过程中，有一些依赖包地址变更了。
以前的做法

>修改源码，用新路径替换import的地址  
>git clone 或 go get 新包后，copy到$GOPATH/src里旧的路径下


无论什么方法，都不便于维护，特别是多人协同开发时。

使用go.mod就简单了，在go.mod文件里用 replace 替换包，例如

```
replace golang.org/x/text => github.com/golang/text latest
```

这样，go会用 *github.com/golang/text* 替代 *golang.org/x/text*，原理就是下载*github.com/golang/text* 的最新版本到 *$GOPATH/pkg/mod/golang.org/x/text*下。  
**这样做就不需要修改代码，只需要修改go.mod文件。**

####  问题五：第三方包和自己的包的源码都在src下，很混乱。

本例里，用 go mod init hello 生成的go.mod文件里的第一行会申明
module hello

我们的项目就已经不在$GOPATH/src里了，那么引用自己怎么办？就用模块名+路径。

例如:
>"hello/utils"

这样就可以区分出自己项目的包和源码包的路径了，也不怕调用包的重名问题了。

深度：replace关键字的引用，有时间再写。

#### 问题六：众所周知的原因，类似于 golang.org/x/... 的包会出现下载失败的问题。

从前的解决方法有以下两种：  
1、下载包到本地放到$GOPATH/src  
2、设置代理  
如果你有代理，那么可以设置对应的环境变量：
>export http_proxy=http://proxyAddress:port  
>export https_proxy=http://proxyAddress:port  

或者，直接用 all_proxy：  
>export all_proxy=http://proxyAddress:port  

**现在Go11版本新增了包的官方代理：**  
新增了*GOPROXY*环境变量。如果设置了该变量，下载源代码时将会通过这个环境变量设置的代理地址，而不再是以前的直接从代码库下载。这无疑对我等无法科学上网的开发良民来说是最大的福音。  

更可喜的是，*goproxy.io*这个开源项目帮我们实现好了我们想要的。该项目允许开发者一键构建自己的 GOPROXY 代理服务。同时，也提供了公用的代理服务*https://goproxy.io*，我们只需设置该环境变量即可正常下载被墙的源码包了：
>export GOPROXY=https://goproxy.io  


例如下载：  
*go build/run/test/get*
```bash
# 解决墙问题，使用GOPROXY代理
GOPROXY=https://goproxy.io go build/run/test/get ${params}
GOPROXY=https://athens.azurefd.net go build/run/test/get ${params}  

```


---

## 思考的两个问题：

疑问：  
1、包的上层依赖是如何可以解决的？  
2、一个项目里面用了一个包的多版本（当然一般包都会向下兼容，但若没有是否可以解决？）