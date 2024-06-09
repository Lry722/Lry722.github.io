---
title: Docker访问host上的服务的方法
published: 2024-01-10
category: 随笔
tags:
  - Docker
---

今天在装 NextCloud 的时候，需要在 docker 容器内访问 host 上的 MySQL 服务。

解决方案为在启动参数中添加这一项：

```
--add-host="host.docker.internal:host-gateway"
```

如果是使用 docker-compose，则在配置文件中添加

```
extra_hosts:
  - 'host.docker.internal:host-gateway'
```

这样就可以在容器内用 host.docker.internal 表示 host 了。
