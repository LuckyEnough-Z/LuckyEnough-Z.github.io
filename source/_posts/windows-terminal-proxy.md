---
title: 【Windows】终端配置代理
date: '2025-01-02 16:57'
tags:
  - Windows
  - 代理
  - Proxy
  - Git
  - PowerShell
categories:
  - 工具配置
abbrlink: 1aeba13e
---

## Windows cmd 设置代理

### 设置 HTTP 代理：

```
set http_proxy=http://127.0.0.1:7890 & set https_proxy=http://127.0.0.1:7890
```

### socks5代理设置：

```
set http_proxy=socks5://127.0.0.1:7890
set https_proxy=socks5://127.0.0.1:7890
```

### 取消代理：

```
set http_proxy=
set https_proxy=
```

## Windows git bash 设置代理

### 设置 HTTP 代理：

```
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890
```

### 设置 socks5代理：

```
git config --global http.proxy socks5://127.0.0.1:7890
git config --global https.proxy socks5://127.0.0.1:7890
```

### 取消代理：

```
git config --global --unset http.proxy
git config --global --unset https.proxy
```

## Windows PowerShell 设置代理

### 设置 HTTP 代理：

```
$Env:http_proxy="http://127.0.0.1:7890";$Env:https_proxy="http://127.0.0.1:7890"
```

### **代**理测试:

```
curl https://www.google.com
```
