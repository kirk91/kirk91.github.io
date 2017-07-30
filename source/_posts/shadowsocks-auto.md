---
title: 科学上网之auto-ss
tags:
  - shadowsocks
  - auto-ss
  - 科学上网
abbrlink: a2f56d46
date: 2015-09-04 23:07:47
---

很开心今年7月份顺利毕业，并成功进入互联网公司成为了一名PYTHON工程师，从此便踏上了一条“不归路” O(∩_∩)O~~；这条路崎岖不平，不但要写得了`code`，还要修得了`bug`，更重要地是要学会科学上网来武装自己，接下来我们就八一八那些年我们追过的科学上网。

`VPN`、`GoAgent`、`自由门`、`红杏`等科学上网工具，伴随着GFW的升级更新很多都已经失效。正所谓长江后浪推前浪，一浪更比一浪高，基于socks协议的[shadowsocks](https://github.com/shadowsocks/shadowsocks)由于其简单方便轻量稳定易于部署的特性，渐渐地被广大服务商和开发者所采用。

ShadowSocks的部署特别简单，但前提是需要在境外或者香港拥有一台vps，如果你拥有自己的vps那还犹豫什么赶紧动手试一下吧（ps：Amazon EC2使用信用卡可以免费使用一年云主机，just do it）。鉴于很多人没有自己的vps，一些SS服务商也很贴心，每天会为注册会员提供免费的账号，但免费的账号会定时更新而且需要用户手动选择最快的链路，如果你曾经使用过这样的免费账号，一定会感到很蛋疼~~~~(>_<)~~~~ 吐槽归吐槽，但他们确实也为我们提供了很多真实可用的服务，[ss-link](https://www.ss-link.com)就是一个这样的服务商，拥有多条国际线路，线路稳定速度平均在200ms左右，之前一直有使用它提供的免费账号，但需要常去网站查看最新的免费账号；作为一个developer怎能经常去刷网页呢，肯定是要使用更geek的方式去获取账号呀。

[`auto-ss`](https://github.com/alphapigger/ss-link-auto)就这样诞生了，它是一个基于python 2.7编写的命令行工具，默认情况下会帮助用户获取最新的SS账号，基本操作如下所示:


![](/images/auto-ss_demo.png)


- mode为`speed`时，`auto-ss`会自动获取免费账号，并测试这些账号的连接速度
- mode为`run`时，`auto-ss`会选择速度最快的线路启动sslocal服务


如果想安装或者获取`auto-ss helper`更多用法，请点击[HERE](https://github.com/alphapigger/ss-link-auto)


----

感谢[github](https://github.com)和[hexo](https://hexo.io/)的易用性，让我有了自己的博客，真的很开心☺️；这是毕业后写的第一篇博文，不到位的地方还请大家多多提建议，真心希望通过博客能够和大家分享自己毕业后的生活和工作，能够和大家一起进步！！！
