---
layout: post
title: "一个 IP 挂多个域名：Nginx 反向代理配置"
date: 2026-09-25
categories: ops nginx
tags: nginx reverse-proxy websocket devops
---

## 简述

多个域名解析到同一个 IP，Nginx 按请求头里的 `Host` 区分目标域名，转发到对应后端。一台机器、一个 Nginx、多个 `server` 块。

## 定义

- `Host` 头：HTTP 请求里标明目标域名的字段，Nginx 拿它当路由依据
- `server_name`：`server` 块声明自己匹配哪个域名，与请求的 `Host` 对照
- `proxy_pass`：反向代理的目标后端地址，命中后把请求转过去

## 演示

一个 IP，三个域名，三个后端：

```
oi.nginx.chat    → 127.0.0.1:3000   # 前端，含 WebSocket
api.nginx.chat   → 127.0.0.1:4000   # 接口
admin.nginx.chat → 127.0.0.1:5000   # 后台
```

三个域名指向同一 IP，Nginx 按 `Host` 头分流：

```
request (Host: oi.nginx.chat)    → server_name 匹配 → proxy_pass → 127.0.0.1:3000
request (Host: api.nginx.chat)   → server_name 匹配 → proxy_pass → 127.0.0.1:4000
request (Host: admin.nginx.chat) → server_name 匹配 → proxy_pass → 127.0.0.1:5000
```

## 对比

一 IP 跑一服务是最直觉的做法，多域名方案的价值在场景差异上，不在形容词。

| 维度 | 一 IP 一域名 | 一 IP 多域名 |
| --- | --- | --- |
| 触发条件 | 每个服务独占公网 IP | 多个域名可解析到同一 IP |
| 路由依据 | IP 本身 | `Host` 头 + `server_name` |
| 资源成本 | IP 数量随服务线性增长 | 复用单个 IP |
| 失败表现 | 端口冲突要换 IP 或端口 | 域名没配对会命中 `default_server`，返回错站或 404 |

## 分层拆解

### 第一层：域名匹配

解决「这个请求到底找谁」。
触发条件：只要有多域名需求，这一层就生效。
案例：`oi.nginx.chat` 命中第一个块，`api.nginx.chat` 命中第二个，靠的是 `server_name` 与 `Host` 的严格对照。

### 第二层：反向代理转发

解决「匹配之后把请求送到哪个后端」。
触发条件：命中 `server` 块之后。
案例：命中 `api.nginx.chat` 后，`proxy_pass http://127.0.0.1:4000` 把请求转给 4000 端口，同时透传 `Host`、`X-Real-IP`、`X-Forwarded-For`，让后端拿到原始域名和客户端真实 IP。

### 第三层：WebSocket 升级

解决「HTTP 握手能不能升级成长连接」。
触发条件：路径是 `/ws/` 这类 WebSocket 端点。
案例：`oi.nginx.chat` 的 `/ws/` 多设三条，缺一条握手就失败：

```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

### 第四层：兜底与边界

解决「没匹配到怎么办、长连接会不会被掐」。
触发条件：请求域名不在任何 `server_name` 里，或 WebSocket 空闲过久。
案例：未配置的域名命中第一个、或标了 `default_server` 的块；长连接空闲时 Nginx 默认读超时会断掉它，需要显式调大。

## 完整配置

```nginx
# oi.nginx.chat —— 前端，含 WebSocket
server {
    listen 80;
    server_name oi.nginx.chat;

    location /ws/ {
        proxy_pass http://127.0.0.1:3000;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_buffering off;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

# api.nginx.chat —— 接口
server {
    listen 80;
    server_name api.nginx.chat;

    location / {
        proxy_pass http://127.0.0.1:4000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

# admin.nginx.chat —— 后台
server {
    listen 80;
    server_name admin.nginx.chat;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

三个块里重复的 header 透传可以抽成包含文件用 `include` 引入，这里保留原样是为了让每个块各自完整可读。

## 关键指令

| 指令 | 功能 |
| --- | --- |
| `server_name` | 声明当前块匹配的域名，支持通配符 |
| `listen 80;` | 监听端口，HTTP 默认 80 |
| `proxy_pass` | 反向代理的后端目标地址 |
| `proxy_set_header Host $host` | 把原始域名带给后端，后端常据此再区分 |
| `proxy_http_version 1.1` | 升到 HTTP/1.1，WebSocket 的前置条件 |
| `Upgrade` + `Connection "upgrade"` | 完成协议升级握手，缺一 WebSocket 不通 |
| `proxy_buffering off` | 关掉响应缓冲，实时推送不被攒着 |

## 规则

- 所有域名必须在 DNS 里解析到同一个 IP。
- 未配置的域名会命中第一个，或标了 `default_server` 的 `server` 块。
- WebSocket 三条指令缺一不可：`proxy_http_version 1.1`、`Upgrade`、`Connection "upgrade"`。
- 生产环境要再叠一层 HTTPS 与证书。
- WebSocket 长连接空闲被默认读超时掐断时，调大 `proxy_read_timeout`，别只查握手头。

## 排查

- 返回错站或 404：对照请求域名与 `server_name`，多数是 DNS 没指向这台机器或不一致。
- WebSocket 连不上：先查三条升级指令，再确认目标端口真在监听。
- 转发后连上又断开：查 `proxy_read_timeout`，这是超时不是握手。
