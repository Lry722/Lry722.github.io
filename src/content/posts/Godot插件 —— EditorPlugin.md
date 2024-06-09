---
title: Godot插件 —— EditorPlugin
published: 2024-01-14
mathjax: true
category: 文章
tags:
  - Godot
  - Godot 插件
---

```
此文会保持更新一段时间，基本上是学到哪更到哪
```

在制作游戏的过程中会遇到各种各样的需求，通常是为了化简某个操作，这些需求往往不能够由引擎提供的通用解决方案很好的满足。例如要编辑自定义的资源，编辑自定义的地图，或是添加一个调试窗口等。

因此，为游戏引擎的编辑器制作插件是个很常见的操作。

而众所周知，Godot 是个开源引擎，因此制作插件这件事也变得非常容易。

虽说是插件系统，但实际上整个 Godot 编辑器本身就是用这个系统搭建起来的。基本上想要做啥样插件，只需要找到引擎中相似的部分，看看那部分源码是怎么实现的就行。~~不过可能要一点 C++ 基础~~

不过话虽如此，还是要对 Godot 的插件系统一些基本的认知，不然改代码都不知道要怎么改。

接下来就介绍一下 Godot 插件系统的核心，`EditorPlugin` 类。

## 概述

我对 EditorPlugin 的总结为：

> 通过继承 EditorPlugin 并重写虚函数，创建可被编辑器识别并加载的插件 。
> EditorPlugin 提供了大量工具函数供子类使用，这些函数基本都是对编辑器相关单例的便捷访问。

## 工具函数

关于 EditorPlugin 提供的工具，大致可分为以下几类：

1. **在编辑器状态变化时发出通知**
   类中有这么四个信号：

| 信号                | 描述                                                                                                   |
| ------------------- | ------------------------------------------------------------------------------------------------------ |
| main_screen_changed | 当用户改变工作空间（**2D、3D、Script、AssetLib、其他自定义工作空间**）时发出。参数是新工作空间的名称。 |
| resource_saved      | 资源保存到磁盘时发出。参数是被保存的资源                                                               |
| scene_changed       | 更改活动场景时发出。参数是新活动场景的根节点。如果新活动场景根节点为空，则为 `null`。                  |
| scene_closed        | 用户关闭场景时发出。参数是关闭的场景的文件路径。                                                       |

2. **往编辑器中任意位置添加任意控件**
   通过形如 `add_control_to_XXX` 的函数，可以往编辑器中任意位置添加$\stackrel{Control 节点}{自定义控件}$

3. **注册基于常见功能模板的插件**
   Godot 为一些常见功能提供了模板，只要继承自模板就能方便快捷地实现相关功能，EditorPlugin 中则为每个模板提供了对应的形如 `add_XXX_plugin` 的注册函数。
   例如 `EditorInceptorPlugin` 就是检查器插件的模板，它提供了默认生成的属性编辑器。通过 EditorPlugin 的 `add_inspector_plugin` 函数，就能将自定义的 `EditorInceptorPlugin` 子类注册进检查器。具体细节看[这篇文章](https://lry722.github.io/2024/01/13/Godot%E6%8F%92%E4%BB%B6%20%E2%80%94%E2%80%94%20EditorInspectorPlugin/)。
   任何此类控件都要记得在对应的自定义 EditorPlugin 子类中进行注册，否则编辑器是认不到的。

4. **添加其他各种东西**
   除了上面提到的常用 add 函数，还有其他往编辑器中添加各种东西的 add 函数。例如用 `add_tool_menu_item` 在 `项目 > 工具` 中添加菜单项，用 `add_custom_type` 在 `创建 Node / 创建 Resource` 中添加选项，用 `add_autoload_singleton` 添加自动加载的单例等等。

   这些函数没啥规律，不需要记住，在需要的时候查文档就行。

5. **移除各种东西**
   上面提到的各种 add 函数，基本都有一个对应的 remove 函数。你甚至可以移除编辑器自带的控件，当然一般不推荐这么做。

上面这几个大类涵盖了大部分工具函数，还有其他少量杂七杂八的可以自行查阅文档。

## 虚函数

- ### notification

EditorPlugin 虚函数有很多，首先必须要实现的就是 `notification` 。

具体来说，需要在收到 `NOTIFICATION_ENTER_TREE` 时把插件自定义的各个控件添加到编辑器中，在收到 `NOTIFICATION_EXIT_TREE` 时进行移除。在 GDS 中也可以重写 `_enter_tree` 和 `_exit_tree`，两者是等价的。

- ### handles 和 edit

插件常见的一个任务就是处理某种对象（Node/Resource）,这就需要重写 handles 和 edit，两个函数都接受一个 Object。

- handles 需要返回一个布尔值，表示能否接受处理这个 Object。常见的实现方法是对 Object 调用 cast_to 进行类型转换，判断转换结果是否为 nullptr 。

- edit 则是实际接受这个 Object，例如假设该插件用于编辑某个资源，就要清除编辑控件中原先的值，填入新接受的 Object 的值，可能还要处理对原先正在编辑的资源的保存。

- ### make_visible

当插件不可见时，可能会希望清除一些东西，或是停止运行 process 以节省性能，在重新可见时再打开，此时就需要重载 make_visible。

- ### \_forward_3d_XXX 和 \_forward_canvas_XXX

前者是 3D 工作区，后者是 2D 工作区，作用相同。

| 函数                   | 描述                                                                                                                   |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| XXX_draw_over_viewport | 传入参数为中央控件，可以在上面进行绘制。通常由编辑器自动调用，也可调用 `update_overlays` 手动更新                      |
| XXX_draw_over_viewport | 和上个函数作用相同，只是通过这个绘制的东西会在最顶层。使用 set_force_draw_over_forwarding_enabled 来启用该方法         |
| XXX_gui_input          | 接受用户在中央控件的输入。参数有俩，一个为编辑器场景中的 Camera，另一个为 InputEvent。只有当活动场景不为空时才会被调用 |

- ### save_external_data

在编辑器保存项目后和关闭项目时被调用，如果这是个负责编辑 Node/Resource 的插件，可能需要询问是否保存已编辑的内容。

# 未完待续
