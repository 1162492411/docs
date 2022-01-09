---
title: "Github站点443超时问题解决"
date: 2022-01-09T09:38:11+08:00
draft: false
toc: TRUE
categories:
  - 软件问题
tags:
  - GitHub
  - 软件问题
---
# 问题描述

使用代理可以在浏览器正常访问github,但是本地推送代码时却仍然提示"Failed to connect to github.com port 443: Operation timed out"

# 解决方案
超时原因在于解析DNS时出现问题,那么我们只要在本地配置好该网站的dns记录即可解决

## 获取真实IP

一共需要获取以下三个网站的DNS

> github.com
>
> github.global.ssl.fastly.net
>
> assets-cdn.github.com

在[IPAddress](https://www.ipaddress.com/)可以查找到这些域名对应的真实IP


## 修改host

找到本地的hosts文件(Mac系统位于/etc/hosts),添加以下内容

> Ip1 github.com
>
> Ip2 github.global.ssl.fastly.net
>
> Ip3 assets-cdn.github.com
>
> Ip4 assets-cdn.github.com
>
> ip5 assets-cdn.github.com
>
> ....

## 刷新本地DNS

上一步配置完DNS记录后,我们还需要清理掉本地已经缓存的DNS,在控制台执行命令清除本地DNS

> Mac : sudo killall -HUP mDNSResponder;say DNS cache has been flushed
>
> Windows : ipconfig /displaydns ipconfig /flushdns


ps:如果最后还是提示 【connect to 127.0.0.01 port 1080: Connection refused】 把代理去掉： git config --global --unset http.proxy git config --global --unset https.proxy
