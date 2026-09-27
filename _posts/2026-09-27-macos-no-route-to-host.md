---
layout: post
title: "神秘的 No Route to Host"
date: 2026-09-27
---

在 macOS 上用 OrbStack 的 Kubernetes 部署 ArgoCD，Service 设置为 LoadBalancer 类型后，OrbStack 会自动分配一个内网 IP，浏览器和 `curl` 都能正常访问。然而命令行工具 `argocd` 一连，却报了 `no route to host`：

```console
% argocd login argocd.dba.berryfl.asia --sso
{"level":"fatal","msg":"error dial proxy: dial tcp 192.168.139.2:443: connect: no route to host","time":"2026-09-27T16:15:43+08:00"}
```

这个错误看起来明显是 IP 路由问题，可浏览器和 `curl` 访问正常又与之矛盾。用 `netstat -nr` 检查路由，流量确实经过 OrbStack 设置的网桥，路由本身也看不出毛病：

```console
% netstat -nr -f inet
Routing tables

Internet:
Destination        Gateway            Flags               Netif Expire
  ... Skip ...
192.168.139.2      1a.78.30.64.16.a9  UHLWI           bridge100      !
```

再用 `tcpdump` 在网桥接口上抓包：浏览器和 `curl` 的包都能抓到，`argocd` 却一个包都没有。也就是说，数据包根本没发出去。

## 真正的原因：Local Network 权限

折腾半天，问题出在 **Privacy & Security → Local Network** 上。并不是所有应用和命令行工具都能看到本地网桥，只有加入白名单的应用才被允许访问：

![macOS Local Network 权限列表](/assets/local-network.png)

浏览器本来就在列表里，所以访问成功；`curl` 则因为带有 Apple 的特殊签名，也能直接访问。其他命令行工具默认继承其宿主应用的权限——换成 macOS 自带的 Terminal，`argocd` 就通了。

## 一个更坑的细节

即便 iTerm 已经在列表里，升级到 macOS 27 之后实际权限可能并未生效，需要把开关关掉再打开（或者退出重开 iTerm）才真正放行：

```console
% argocd login argocd.dba.berryfl.asia --sso
Performing authorization_code flow login: https://argocd.dba.berryfl.asia/api/dex/auth?access_type=offline&client_id=argo-cd-cli&code_challenge=skip...
Opening system default browser for authentication
Authentication successful
'xxxx@gmail.com' logged in successfully
Context 'argocd.dba.berryfl.asia' updated
```

## 小结

macOS 虽然也是类 Unix 环境，但安全机制比 Linux 的 root 用户要严得多：并不是什么东西都对本机可见。于是就出现了这种违背直觉的现象——路由没错、服务正常，偏偏某个命令行工具就是 `no route to host`。
