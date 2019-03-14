---
title: Elastic Shell发布暨晴耕工坊上线
excerpt: 新栏目“晴耕工坊”上线，开源项目Elastic Shell首发，纯脚本工具轻松管理Elasticsearch
categories: [tech]
tags: [elasticsearch, shell, news, studio]
toc: true
toc_sticky: true
---

经过一段时间的筹划，作为[“晴耕小筑”](/)的一档新栏目[“晴耕工坊”](/studio)终于和诸位见面了！该栏目汇集了“晴耕小筑”以往公开的原创小品。比如，近期发布的[Elastic Shell](https://github.com/morningspace/elastic-shell)，便是由“晴耕工坊”隆重推出的一个开源项目。其作用是提供一组Shell脚本，用来管理Elasticsearch，稍后还将详细介绍。除此以外，还有部分笔者早年的开源小品，也一并收录于此。未来，还会有更多开源项目通过“晴耕工坊”对外发布。所有开源项目，均可在“晴耕小筑”的[GitHub网站](https://github.com/morningspace)找到。下面我们就一起来近距离了解一下Elastic Shell吧:-)

## 什么是Elastic Shell

![](/assets/images/studio/elash.png)

Elastic Shell是一套完全用Bash Shell编写的，用于管理Elasticsearch的工具脚本。目前，它提供了针对index和snapshot的基本管理，并提供了针对reindex和Elasticsearch集群升级的辅助自动化。

本质上，Elastic Shell是一个Elasticsearch的客户端。所有功能都是通过curl发起针对Elasticsearch的REST API调用来完成的。

此外，Elastic Shell还提供了一些高阶的辅助功能，比如：logging，dry run，auto completion等。

## Elastic Shell的由来

开发Elastic Shell原本只是工作需要。当时笔者正在寻找一个工具，能够辅助完成将生产环境中的Elasticsearch集群数据迁移到另一个新的集群环境中去。

原则上，这项工作可以通过调用Elasticsearch的reindex REST API完成。但是，在生产环境中让运维团队每次都手工发起REST请求显然是不合适的。理想情况下，我们需要的是一个可供运维执行的程序，比如一段脚本，能够读取配置文件，然后自动发起reindex请求，并且能够在reindex执行期间记录日志，在reindex结束之后提供统计报告。

在网络上搜索了一阵之后，发现并没有一款工具能够完全满足需求，于是就萌生了自己编写工具的想法。这便是Elastic Shell的雏形。

## 为什么选择Shell

选择Shell脚本实现，出于两点考虑。

其一，利用Shell语言开发脚本，实现操作系统下的常见管理任务是以Linux为代表的操作系统十分常见的做法。Bash Shell是Shell脚本语言中最为常用的典型代表之一。利用它，再结合各种外部可执行程序，可以实现各种丰富而强大的功能。

其二，Elastic Shell的编写过程也是笔者对Shell编程的学习与实作过程。很久以前就曾在DOS下写过批处理文件，但那都是陈年往事。一直以来，对Linux下Shell脚本的那些稀奇古怪的书写方式不明觉厉，这次逮着机会有了深入的了解。在开发Elastic Shell期间，通过密集的网络搜索，自觉对Shell编程的理解突飞猛进，实作方面也精进了不少。诸位如果有兴趣阅读Elastic Shell的源码就会看到，其中使用了很多Shell编程中有意思的技巧，比如：

* 大量管道（pipe）的运用让代码简化和灵活了不少
* 各种字符串处理与比对方法，体现了Shell强大和丰富的处理能力
* 数组与Positional Parameters的使用随处可见
* ……

## 如何运行Elastic Shell

一如晴耕小筑以往推出的作品，Elastic Shell也支持在Docker容器内运行。容器运行的好处在于，Elastic Shell所依赖的第三方程序都已预装，且做了相应的配置，因此无需自己再手动安装和配置。

当然，将Elastic Shell置于容器外，以Standalone方式运行，同样也是支持的。并且，即便没有安装某些第三方依赖，Elastic Shell也依然能以某种“功能受限”的形态运行，详情请见GitHub上的[README](https://github.com/morningspace/elastic-shell)。

不仅如此，除了以可执行脚本的形式在命令行下运行外，Elastic Shell还支持交互运行模式，并且提供文本形态的图形化界面。这一特性可能是目前同类型工具中（包括[Curator](https://github.com/elastic/curator)在内）所缺少的。以图形界面运行，使用者可以借助菜单进行操作，从而免去了对诸多命令行参数的记忆。而交互运行模式，则比较适合需要较多人工干预的Elasticsearch管理任务，或者是以交互操作为主的一些实验场合。

## 如何使用Elastic Shell

关于Elastic Shell的具体使用，可以查看其在GitHub上的[README](https://github.com/morningspace/elastic-shell)。在随后的一段时间里，“晴耕小筑”计划还将推出若干小视频，用以演示Elastic Shell的诸多亮点功能，及其使用技巧，敬请期待！

## 和Curator的区别

有一个问题熟悉Elasticsearch及ELK的同学一定会问：Elastic Shell和[Curator](https://github.com/elastic/curator)到底有什么区别？Curator是ELK社区广泛使用的Elasticsearch索引管理工具，它是由Python编写的。希望进一步了解Curator的同学可以查阅其[官方文档](https://www.elastic.co/guide/en/elasticsearch/client/curator/current/about.html)。

Elastic Shell在个别功能上与Curator有小部分重叠，主要是针对index和snapshot的管理方面。这是Elastic Shell作为一个工具集，在完备性方面所必须提供的功能。Elastic Shell着眼更多的，是Curator因其自身定位不同而不具备的某些功能，比如对reindex和集群升级提供的辅助自动化。

Elastic Shell无意替代Curator，而是Curator的一个轻量级补充，并且它是纯Shell脚本编写而成的，无需Python语言环境，从而给使用者多了一种选择。

关于Elastic Shell和Curator的讨论，可参见笔者此前在Elasticsearch官方论坛里发起的一则[问答贴](https://discuss.elastic.co/t/looking-for-shell-based-elasticsearch-client-or-something-similar-to-curator-run-in-command-line/166009/5)。

最后，祝各位玩的开心:-) Have fun！