---
title: Docker Engine 的 RESTful API 版本管理策略
header-img: restful-api-versioning.jpeg
tags:
  - RESTful
  - Docker
permalink: restful-api-version-management-policy-for-docker-engine
date: 2017-06-07 00:02:51
---
[Docker](https://github.com/moby/moby) 项目是一个标准的 C/S 架构，Docker Daemon（二进制为 dockerd） 作为 Server，其他诸如 Docker Client、docker-py 等则作为 Client。Daemon 提供了一套标准化的 RESTful API 供客户端使用。经过多年的发展，Docker 的版本号发生了大变革，变成了日期命名并且已经到了 17.05 版本，Docker 项目甚至改名成了 Moby。但 Docker 的 API 却一直保证了良好的兼容性，有着完善的版本管理策略和文档。

本文将就 Docker Engine 的 API 版本管理策略展开讨论。

## 代码版本和 API 版本相互独立

在命令行中执行 `docker version` 我们可以看到

```bash
$ docker version

Client:
 Version:      17.04.0-ce
 API version:  1.28
 Go version:   go1.7.5
 Git commit:   4845c56
 Built:        Wed Apr  5 06:06:36 2017
 OS/Arch:      darwin/amd64

Server:
 Version:      17.04.0-ce
 API version:  1.28 (minimum version 1.12)
 Go version:   go1.7.5
 Git commit:   4845c56
 Built:        Tue Apr  4 00:37:25 2017
 OS/Arch:      linux/amd64
 Experimental: true
```

Client 和 Server 除了本身的 Version 外还有一个 API Version，