---
title: 晴耕小记
excerpt: 老铺新张，重装上线
categories: [tech]
tags: [thoughts]
toc: true
toc_sticky: true
---

> 今天是[“勤耕小筑”](https://morningspace.github.io/)正式上线的第一天！

经过前阵子断断续续的“筹备”与“试运行”，个人网站又重新开张了！本来十一期间就已经准备就绪，也许是“仪式感”在作祟，总觉得缺个小记，想写点什么，加上平时只有工作之余的碎片时间，所以一直拖到了现在。

## Git Pages取代WordPresss

几个月前，忽然发觉以前网站的运营商服务到期了，鉴于对其服务质量不慎满意，于是决定换址。

一番考察之后，选择了[Git Pages](https://pages.github.com/)作为站点发布方案。之所以如此选择，是觉得基于[GitHub](https://github.com/)的发布模式比较符合自己作为程序员的工作习惯：和代码提交一样，用[Markdown](https://daringfireball.net/projects/markdown/)在本地写好的素材，通过Git客户端推送到远程的Git库，大概几秒钟之后，就可以看到网站更新后的内容了。

而且，因为[Jekyll](https://jekyllrb.com/)生成的全部都是静态页面，所以网站加载的速度很快。和以前的[WordPress](https://wordpress.org/)相比，没有了数据库以及PHP，部署和维护也都变得更加简单了。

不仅如此，当我看过一位国外同事所使用的Jekyll主题是如此强大之后，更加坚定了选择GitHub + Jekyll的建站方案。感谢[Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/)，这大概是[目前基于Jekyll建站的主题之中最为流行的一款](https://github.com/topics/jekyll-theme)了吧。

## Staticman实现评论功能

由于全部内容都基于静态页面，默认情况下，Git Pages生成的站点是没有评论功能的。倒是有不少第三方评论系统可供集成，其中最为流行的莫过于[disqus](https://disqus.com/)。只是，由于某些众所周知的原因，我们在这里是无法访问到disqus的。

不过功夫不负有心人，一番摸索之后，我找到了自认为最适合自己的评论系统：[staticman](https://staticman.net/)。它的独特之处在于，作为第三方系统，staticman自身并不存储评论内容，而是借助于GitHub上一个特殊的Robot账号，将评论和普通的源码文件一样，以文件的形式发往远端的Git库——这和我准备网站素材的方式完全一致，简直就是绝配！

## reCAPTCHA提供验证码服务

有了评论功能，自然少不了防止恶意攻击的验证码。对于这一点，我也曾经深受其苦。在运营商那里租用的MySQL数据库空间，由于不断涌现的垃圾评论，很容易就会被撑爆。

为了解决这一问题，Google的[reCAPTCHA](https://www.google.com/recaptcha/intro/)是目前使用最为广泛的一种方案。不过因为同样的原因，这里是无法访问的。好在有国内的景象，对主题稍加改造之后，就可以轻松集成进来。

## 近期计划与展望

由于网站初建，目前内容不多。近期的计划，是将一些过去所写的文章从旧址逐渐搬迁过来。同期还会开设微信公众号与[YOUKU自频道](http://i.youku.com/morningspace)(在建)，欢迎各位扫描网站下方的二维码订阅并关注:-)

回顾过去，从2002年开始制作自己的个人网站算起，兜兜转转，起起伏伏，至今已走过了16个年头。“勤耕小筑”，寄托着一份小小的“执念”：人生际遇，或有阴晴圆缺；晴时耕作，雨时读书；工作之余，将点滴记录下来；并将沿途所见，与诸君同享。希望能够践行此言！
