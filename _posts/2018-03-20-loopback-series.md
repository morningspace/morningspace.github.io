---
title: 我为什么要来IBM微讲堂讲LoopBack？
excerpt: "为IBM微讲堂的LoopBack系列课程作推广序"
categories: [blog, tech]
tags: [loopback]
toc: true
toc_sticky: true
---

开源技术*IBM微讲堂的[《Nodejs应用开发新秀——深入浅出LoopBack》](https://developer.ibm.com/cn/blog/2018/opentec-loopback/)系列即将于3月22日和各位同学见面了。作为这一系列课程的设计者与合作参与者，为了能够让大家更好的理解这门课程的设计背景与设计思路，以及LoopBack本身的来龙去脉，以便在开课之前能够有所铺垫，特撰此文与诸位分享。

## 缘起

这些年来，在IT领域与某些传统领域里，无数企业都在为数字化转型进行着艰苦卓绝的奋斗。而在这场快鱼吃慢鱼，而非大鱼吃小鱼的博弈游戏中，API经济的提出适逢其时，并且方兴未艾。API凭借其接口标准化，对开发者友好，开发语言无关，以及包含设计，构建，测试，运维在内的全生命周期理念，为企业实现快速构建与交付，从而快速响应市场及业务的变化带来了契机。而开源技术框架——LoopBack作为支撑这一理念的实践者，围绕API这一核心，从技术角度全方位的诠释了：如何通过技术手段将API经济付诸实际，从而找到了技术与业务的最佳契合点，让理想落到了实处。关于LoopBack如何覆盖API的全生命周期，此处不再赘述，有兴趣希望了解详情的读者，欢迎来参加本课程:-)

这两年里，因所在部门技术转型的机缘，笔者有幸开始接触基于Node.js的这一优秀的开源框架。通过这两年，将LoopBack大规模应用于产品代码及微服务部署，笔者以及身边的同事们对LoopBack的使用体会越发的深切。不仅如此，有时因为项目的需要，在LoopBack现有功能无法满足的情况下，也会深入解读代码，并对其加以定制和扩展，个中滋味颇值得玩味。

去年年底，开源技术*IBM微讲堂的负责人辗转联系到笔者，商讨新设课程的事宜。本着“开源与共享”的精神，双方一拍即合。此后，大家便逐渐开始进入了紧张的课程准备工作当中。在准备课程的过程中，笔者注意到，尽管LoopBack本身功能强大，但在国内，中文网络资源似乎相对较少；一些同业们对LoopBack的看法，也似乎有些许偏颇之处，颇有“酒香却怕巷深”的感觉。于是，为了能够让大家更为全面准确的了解LoopBack这一优秀的开源技术框架，这一系列课程的意义，似乎又深了一层。

## 五幕剧

本课程的设计，采用“五幕剧”的形式，每一幕对应一讲，分别是：

* 我们的第一个应用：初尝LoopBack的应用开发
* 进入MODEL的世界：理解LoopBack的核心概念
* 威力无比的百宝箱：扩展LoopBack的应用逻辑
* 缤纷多彩的数据源：打开LoopBack的数据之窗
* 内幕劲爆的最终章：深入LoopBack的方方面面

从基本的开发体验，到核心的概念理解，再到各种灵活的定制与扩展手段介绍，再到一些高阶议题的探讨与经验分享，由浅入深的涵盖了LoopBack的方方面面。

## TaskMe

我们还专为课程量身定制了一套完整的示例应用——一个用于管理个人任务的简单小程序，取名TaskMe。在课程最开始的时候，TaskMe只是一个简单的雏形。随着“剧情”的一步步推进，我们会在每一幕中为其逐步加入各种特性，完善其功能的同时，也展示了每一幕中所要阐述的各种概念的实际运用。

值得一提的是，包括TaskMe的源代码，以及课程配套的全部讲义在内，我们都将以开源形式发布到GitHub上。诸位可以访问[这里](https://github.com/morningspace/understanding-loopback)以获取所有内容。在每次课程开讲之前，我们会将与该次课程配套的讲义，以及该阶段的TaskMe项目源代码上传至此，以供大家提前预览，或者课后复习。对于讲义，因为采用的是reveal.js（同样是非常优秀的开源项目），再配合GitHub的Pages功能，大家可以直接在桌面浏览器或手机微信或浏览器中加以浏览，非常方便！

当然，也非常欢迎各位的反馈意见。以程序员最为熟悉的方式——任何你想讨论的问题，请以GitHub Issue的形式提交至[这里](https://github.com/morningspace/understanding-loopback/issues)，我们会积极响应！

好了，闲言碎语说了很多。在开始正式进入课程之前，先为诸位奉上一道“开胃菜”。尤其是对于尚未接触过LoopBack的同学们，让我们热个身，来看一看：要把“Hello World”搭出来，统供需要分几步？

## 热身

初尝Loopback给人印象最深的也许就是快速上手了。在此，笔者将为大家介绍如何运用LoopBack的命令行工具，快速搭建起一个允许客户端进行REST接口访问的服务。

### 第一步：安装

首先是安装。LoopBack的安装非常简单，其命令行工具完全遵循npm的标准安装模式。因此，熟悉Node.js的同学可以轻松领会，只要运行如下命令（注：此处以Linux，Mac OS等操作系统为例）：
```shell
$ npm install -g loopback-cli
```
就可以在本地对loopback-cli进行全局安装。有了命令行工具，接下来我们就可以创建应用啦！

### 第二步：创建应用

执行如下命令，会进入一个Wizard模式。只要根据命令行的提示逐一回答问题，并选择相应的选项，即可完成应用框架代码的自动生成。
```
$ lb
? What's the name of your application? hello-world
? Enter name of the directory to contain the project: hello-world

? Which version of LoopBack would you like to use? 3.x (current)
? What kind of application do you have in mind? hello-world (A project containing a controller,
including a single vanilla Message and a single remote method)
...
I'm all done. Running npm install for you to install the required dependencies.
If this fails, try running the command yourself.
... 
```
如上，我们告诉loopback-cli，创建一个名为hello-world的应用，且应用的源代码位于当前目录下名为hello-world的子目录内。此外，我们还指定了：

* LoopBack的版本号：LoopBack当前的版本为3.x，同时loopback-cli也支持老版本，此处我们选择最新的版本。
* 应用的代码模版：LoopBack提供了若干个预定义的代码模版，方便我们在此基础上进行定制与扩展。此处我们选择的是hello-world模版。

当一切准备就绪，loopback-cli会为我们生成相应的代码，并自动为我们安装当前应用所依赖的所有第三方npm包。待loopback-cli安装完所有依赖之后，我们会看到类似如下的提示：
```shell
Next steps:

  Change directory to your app
    $ cd hello-world

  Create a model in your app
    $ slc loopback:model

  Run the app
    $ node .
```
按照提示，进入hello-world目录，并执行node .命令，便可启动我们刚刚创建好的hello-world应用啦！

### 第三步：执行应用

如前，当我们利用node .命令成功启动了hello-world应用之后，会看到类似如下的提示：
```shell
Web server listening at: http://localhost:3000
Browse your REST API at http://localhost:3000/explorer
```
按照提示，我们打开浏览器，输入地址：http://localhost:3000/explorer 便会进入LoopBack为我们提供的API Explorer界面。LoopBack的API Explorer以图形化的界面为我们展示了当前应用所提供的所有REST APIs，包括每一个API的文档说明。使用者可以快速浏览每个API的用法。不仅如此，点击“Try it out!”按钮还可以实现和应用服务的实时交互，所以它是一份“可运行的API文档”。

![](/assets/images/loopback/hellow-world-1.png)

在本例中，LoopBack已经为我们的hello-world程序预先创建好了一个Message服务和User服务。点击“List Operations”即可以展开方式列出各个服务所提供的REST APIs。以Message为例，它提供了一个支持GET请求的/Messages/greet REST API。点击该API，可以进一步展开针对该API的文档说明，包括其所支持的每个URL参数以及返回结果的说明。除此以外，紧随其后还有一个“Try it out!”按钮。闲话少说，让我们即刻测试一下这个API吧！在名为msg的参数处填入你希望的值，然后点击按钮，看看是否会得到类似如下的结果呢？

![](/assets/images/loopback/hellow-world-2.png)

## 尾声，也是开始

看了上面的内容，也许你会疑虑，这种以模版为基础的代码生成方式，是否会对定制开发造成约束。或者，你的脑海里还有很多问题亟待解决。没关系，所有问题都将会在《Nodejs应用开发新秀——深入浅出LoopBack》系列课程中得到答案。欢迎各位前来参加这一课程，3月22日晚8点，笔者与你不见不散！
