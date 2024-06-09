---
title: WPS无法使用fcitx5输入中文
published: 2024-01-16
category: 随笔
tags:
  - WPS
---

如题，在 ArchLinux 下使用 WPS 时无法使用 fcitx5 输入中文。

解决方案为修改启动脚本 `/usr/bin/wps`，在 `gOpt=` 下一行添加

```
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx5
export XMODIFIERS=@im=fcitx
```

参考文章 [在 wps 中使用 fcitx5](https://www.cnblogs.com/kaleidopink/p/14015792.html)

~~后来才发现 Arch Wiki 中其实有提到这个问题~~
