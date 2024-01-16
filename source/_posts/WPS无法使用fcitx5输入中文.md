---
title: WPS无法使用fcitx5输入中文
date: 2024-01-16 21:54:26
category:
  - 随笔
tags:
  - WPS
---

如题，在ArchLinux下使用WPS时无法使用fcitx5输入中文。

解决方案为修改启动脚本 `/usr/bin/wps`，在 `gOpt=` 下一行添加
```
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx5
export XMODIFIERS=@im=fcitx
```

参考文章 [在wps中使用fcitx5](https://www.cnblogs.com/kaleidopink/p/14015792.html)