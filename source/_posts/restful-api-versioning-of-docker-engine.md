---
title: Docker Engine 的 RESTful API 版本管理策略
header-img: restful-api-versioning-of-docker-engine.jpeg
tags:
  - RESTful
  - Docker
permalink: restful-api-version-management-strategy-of-docker-engine
date: 2017-06-07 00:02:51
---
[Docker](https://github.com/moby/moby) 项目是一个标准的 C/S 架构，Docker Daemon（二进制为 dockerd） 作为 Server，其他诸如 Docker Client、docker-py 等则作为 Client。Daemon 提供了一套 RESTful API 供客户端使用。经过多年的发展，Docker 的版本号发生了大变革，变成了日期命名并且已经到了 17.05 版本，Docker 项目甚至改名成了 Moby。但 Docker 的 API 却一直保证了良好的兼容性，有着完善的版本管理策略和文档。

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

Client 和 Server 除了本身的 Version 外还有一个 API Version。实时上 Docker 的 API 版本和代码的版本是完全相互独立的，虽然每个代码版本都会唯一对应一个 API 版本，但 API 版本并不会随着代码版本变更。我们翻看 Docker 官网的文档可以发现罗列的从 v1.1.8 开始的各个 API 版本。

每个版本所有 API 都属同一版本，用户可以通过 `/info` 来访问，也可以通过 `/v1.28/info` 来访问 API。

## 文档定义版本

看到如此多的 API 版本和复杂的变更，我们会想，在代码层面 Docker（moby） 项目是如何标记这么多版本的 API 兼容，变更，丢弃的呢？

答案是：并没有，除了极少数 API 在代码中会根据版本有特殊处理外，兼容、变更、丢弃都没有体现在代码里。

那么这么详细的文档怎么生成的呢？

答案是：纯手写，Docker 项目并没有使用各种先进的自动文档生成工具，而是采用全部人工编写的方式。

每个版本的 API 都是用文档来定义的，Docker 项目对于文档非常重视，他们甚至有专职的文档工程师。这些文档都位于项目的 [docs/api](https://github.com/moby/moby/tree/master/docs/api) 目录下

```
$ tree docs/api
docs/api
├── v1.18.md
├── v1.19.md
├── v1.20.md
├── v1.21.md
├── v1.22.md
├── v1.23.md
├── v1.24.md
└── version-history.md

0 directories, 8 files
```

我们可以看到这里有各个版本的文档，还有一个 `version-history.md` 标记了每个版本的变更，这些文档都是手动编写的。Docker 项目有着非常严格的 Git 工作流，每个 Pull Request 如果涉及到 API 内容的变更都必须要修改文档并让文档经过文档工程师的 Review 才能合并到主线。

## 由客户端处理

既然代码里没有处理，只提供了文档，那么就得客户端来判断 API 的兼容、变更、丢弃了。

拿官方 Python 客户端 [docker-py](https://github.com/docker/docker-py/) 来说，我们看列出网络 API 的[代码](https://github.com/docker/docker-py/blob/master/docker/api/network.py)：

```python
# docker/api/network.py
class NetworkApiMixin(object):
    @minimum_version('1.21')
    def networks(self, names=None, ids=None, filters=None):
        """
        List networks. Similar to the ``docker networks ls`` command.
        Args:
            names (:py:class:`list`): List of names to filter by
            ids (:py:class:`list`): List of ids to filter by
            filters (dict): Filters to be processed on the network list.
                Available filters:
                - ``driver=[<driver-name>]`` Matches a network's driver.
                - ``label=[<key>]`` or ``label=[<key>=<value>]``.
                - ``type=["custom"|"builtin"]`` Filters networks by type.
        Returns:
            (dict): List of network objects.
        Raises:
            :py:class:`docker.errors.APIError`
                If the server returns an error.
        """

        if filters is None:
            filters = {}
        if names:
            filters['name'] = names
        if ids:
            filters['id'] = ids
        params = {'filters': utils.convert_filters(filters)}
        url = self._url("/networks")
        res = self._get(url, params=params)
        return self._result(res, json=True)
```

我们注意到 `networks` 方法有一个 `@minimum_version('1.21')` 装饰器，根据文档的记述标记了 networks 是从哪个 API 版本开始引入的。

```python
# docker/utils/decorators.py
def minimum_version(version):
    def decorator(f):
        @functools.wraps(f)
        def wrapper(self, *args, **kwargs):
            if utils.version_lt(self._version, version):
                raise errors.InvalidVersion(
                    '{0} is not available for version < {1}'.format(
                        f.__name__, version
                    )
                )
            return f(self, *args, **kwargs)
        return wrapper
    return decorator
```
在访问这些 API 的时候客户端会检测服务端的 API 版本是否满足最小版本要求，不满足则会报错。

```python
if utils.version_lt(self._version, MINIMUM_DOCKER_API_VERSION):
    warnings.warn(
        'The minimum API version supported is {}, but you are using '
        'version {}. It is recommended you either upgrade Docker '
        'Engine or use an older version of Docker SDK for '
        'Python.'.format(MINIMUM_DOCKER_API_VERSION, self._version)
    )
```
Client 类实例化的时候通过指定或者访问 Docker 的 `/version` 接口自动得到 API 版本，并且会检查版本是否满足要求。

```python
# docker/constants.py
DEFAULT_DOCKER_API_VERSION = '1.26'
MINIMUM_DOCKER_API_VERSION = '1.21'
```

客户端定义了自己兼容的最低版本.

## API 设计的原则

1. 严格撰写文档，写清楚每一个返回值和返回指定类型，标明每个可能的 HTTP Error 和相应的 Status Code。Docker Engine API 在 v1.25 之后使用 [swagger](http://swagger.io/) 来生成易读的文档页面，swagger 还能自动生成客户端示例代码
2. 一旦 API 版本定下来绝对不要修改 API
3. 设计 API 的时候要考虑兼容性，并且尽量不要去改动原来的 API。API 变化优先做法由高到低是：不改 > 增加 API > 添加旧 API 的字段 > 修改 Endpoint > 修改旧 API 的字段。增加的新字段最好给 CUD 方法加上默认值
4. 设计 API 要有一定前瞻性，不要频繁发布新版本

## 总结


Docker 的 API 管理是一个比较成功的范例，他没有使用 “先进” 的自动化 API 工具，而是用流程、规范来管理这么数量庞大的 API。在保证文档完整正确的同时也非常灵活。值得我们的借鉴。


