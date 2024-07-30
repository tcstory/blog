---
title: endeavor os 之代理设置
date: 2023-10-15 16:22:00
tags:
---

作为程序员, 我们通常需要设置好网络, 最近, 我就从 https://manateelazycat.github.io/2023/06/04/best-proxy/ 了解到了一个新的代理工具 proxy-ns, 了解后, 确实比 proxychains 好用(下面我会提到 proxychains 的缺点).

但是配置 proxy-ns 的过程, 也很艰难. 所以, 我记录一下, 大致需要的配置过程.

## 安装

安装的过程我就不说了, 你可以上看上面的文章.

## 安装后的配置

由于我的 clash serve 跑在端口 7890 上, 所以你需要修改一下端口, 并且我发现, 配置文件默认的 dns 服务器 `9.9.9.9` 似乎在我这里无法工作, 所以, 我把他给替换掉了.  所以, 你只需要修改下面这两项:

```bash
SOCKS5_PORT=7890
DNS=8.8.8.8
```

然后重启服务: `sudo systemctl restart proxy-nsd@main.service`

然后, 执行 `proxy-ns curl www.google.com` 来观察 proxy-ns 是否正常工作

## 关闭防火墙

如果依旧发现 proxy-ns 无法正常工作, 那么可能是防火墙导致的. 你可以试着执行 `sudo systemctl stop firewalld ` 来关闭防火墙.

起码在我这里, 这样操作后, 就一切正常了.

不过, 你当然不希望那么暴力的把防火墙关闭, 所以, 你可以按照下面的方法来进行配置, 让 proxy-ns 能正常工作.

```bash
firewall-cmd --zone=public --rm-port=7890/tcp --permanent
firewall-cmd --reload
```

## proxychains 的缺陷

上面我提到了, proxy-ns 比 proxychains 好用, 原因有两个:

1. 有时候, 即使里使用 proxychains 来构建或者安装软件, 但是这个软件内部的构建命令, 似乎还是不会走代理
2. proxychains 的 proxy_dns 配置项似乎有缺陷, 当你开启它的时候(他原本就是默认开启的), 如果里执行 `proxychains git clone git@github.com:gukaifeng/hexo.git` , 那么你会遇到 `ssh: Could not resolve hostname github.com: Unknown error`. 即使我查看相关的 issue, 他们提到了这个问题已经被修复, 但是在我这里依旧会复现

## proxy-ns 的 dns 代理

我本来想让 proxy-ns 使用我 clash 的 dns 服务, 但是我的 dns 服务端口是 33535, 不是默认的 53. proxy-ns 的配置项里, 似乎不支持配置 dns 的端口. 所以, 对于这一块, 我是有点担心的

