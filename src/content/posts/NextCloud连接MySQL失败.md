---
title: NextCloud连接MySQL失败
published: 2024-01-10
category: 随笔
tags:
  - MySQL
---

已经在 MySQL 中建好了 NextCloud 需要的 database 和 user，但就是连不上。

最后发现是在给 user 授权的时候，应该在末尾添加 `with grant option`。也就是：

```sql
grant all privileges on *.* to 'user'@'localhost' with grant option;
```

原因是 NextCloud 连上数据库后还要再创建新的 user 才能正常运行，`with grant option` 的作用是允许 user 将自己的权限传递给新的 user。因此加上这个授权后 NextCloud 才能给新的 user 授权，才能正常运行。
