---
title: GDExtension构建流程
published: 2024-01-10
category: 文章
tags:
  - Godot
  - GDExtension
---

要构建 GDExtension，请确保你的电脑上已有：

- Godot 4 可执行文件

- C++ 编译器

- SCons 作为构建工具

## 1. 目录结构

```
工程目录/
|
+--Godot工程目录/       # 用Godot在这个目录下新建工程
|
+--godot-cpp/           # 从godot-cpp仓库clone
|
+--src/                 # 你写的GDExtension的源码
```

先创建工程目录，然后创建三个子目录，`工程目录`和`Godot工程目录`的名字可根据需求任意改。

## 2. 构建 C++ 绑定

在`工程目录`下执行：

```bash
git init
git submodule add -b 4.2 https://github.com/godotengine/godot-cpp
cd godot-cpp
git submodule update --init
```

其中 4.2 为写这篇文章时的最新稳定版，实际操作时可以换成新版本。

然后，在 `godot-cpp` 目录下执行：

```bash
godot --dump-extension-api
scons platform=系统名 custom_api_file=extension_api.json
# 系统名替换成实际系统名，windows就写windwos，linux就写linux
```

此时会开始编译，可能要等待一段时间。

## 3. 创建 Godot 工程

用 Godot 在`Godot工程目录`下创建一个空的工程，创建完后可以直接退出 Godot。

## 4. 编写 Extension 代码

上面这些准备完成后，可以在 src 目录下开始实际编写 Extension 的代码，以下为示例代码。

- ### gdexample.h

```cpp
#ifndef GDEXAMPLE_H
#define GDEXAMPLE_H

#include <godot_cpp/classes/sprite2d.hpp>

namespace godot
{

	class GDExample : public Sprite2D
	{
		GDCLASS(GDExample, Sprite2D)

	private:
		double time_passed;
		double time_emit;
		double amplitude;
		double speed;

	protected:
		static void _bind_methods();

	public:
		GDExample();
		~GDExample();

		void set_amplitude(const double p_amplitude);
		double get_amplitude() const;
		void set_speed(const double p_speed);
		double get_speed() const;

		void _process(double delta) override;
	};

}

#endif
```

- ### gdexample.cpp

```cpp
#include "gdexample.h"
#include <godot_cpp/core/class_db.hpp>

using namespace godot;

void GDExample::_bind_methods()
{
	ClassDB::bind_method(D_METHOD("get_amplitude"), &GDExample::get_amplitude);
	ClassDB::bind_method(D_METHOD("set_amplitude", "p_amplitude"), &GDExample::set_amplitude);
	ClassDB::add_property("GDExample", PropertyInfo(Variant::FLOAT, "amplitude"), "set_amplitude", "get_amplitude");

	ClassDB::bind_method(D_METHOD("get_speed"), &GDExample::get_speed);
	ClassDB::bind_method(D_METHOD("set_speed", "p_speed"), &GDExample::set_speed);
	ClassDB::add_property("GDExample", PropertyInfo(Variant::FLOAT, "speed", PROPERTY_HINT_RANGE, "0,20,0.01"), "set_speed", "get_speed");

	ADD_SIGNAL(MethodInfo("position_changed", PropertyInfo(Variant::OBJECT, "node"), PropertyInfo(Variant::VECTOR2, "new_pos")));
}

GDExample::GDExample()
{
	time_passed = 0.0;
	amplitude = 10.0;
	speed = 1.0;
}

GDExample::~GDExample()
{

}

void GDExample::_process(double delta)
{
	time_passed += speed * delta;

	Vector2 new_position = Vector2(
		amplitude + (amplitude * sin(time_passed * 2.0)),
		amplitude + (amplitude * cos(time_passed * 1.5)));

	set_position(new_position);

	time_emit += delta;
	if (time_emit > 1.0)
	{
		emit_signal("position_changed", this, new_position);

		time_emit = 0.0;
	}
}

void GDExample::set_amplitude(const double p_amplitude)
{
	amplitude = p_amplitude;
}

double GDExample::get_amplitude() const
{
	return amplitude;
}

void GDExample::set_speed(const double p_speed)
{
	speed = p_speed;
}

double GDExample::get_speed() const
{
	return speed;
}
```

还需要两个文件，`register_types.h` 和 `register_types.cpp` 。刚刚新建的这个类 Godot 并不知道，这俩文件用于将新建的类注册进 Godot。

- ### register_types.h

```cpp
#ifndef GDEXAMPLE_REGISTER_TYPES_H
#define GDEXAMPLE_REGISTER_TYPES_H

#include <godot_cpp/core/class_db.hpp>

using namespace godot;

void initialize_example_module(ModuleInitializationLevel p_level);
void uninitialize_example_module(ModuleInitializationLevel p_level);

#endif
```

- ### register_types.cpp

```cpp
#include "register_types.h"

#include "gdexample.h"

#include <gdextension_interface.h>
#include <godot_cpp/core/defs.hpp>
#include <godot_cpp/godot.hpp>

using namespace godot;

void initialize_example_module(ModuleInitializationLevel p_level)
{
	if (p_level != MODULE_INITIALIZATION_LEVEL_SCENE)
	{
		return;
	}

	ClassDB::register_class<GDExample>();
}

void uninitialize_example_module(ModuleInitializationLevel p_level)
{
	if (p_level != MODULE_INITIALIZATION_LEVEL_SCENE)
	{
		return;
	}
}

extern "C"
{
	// 初始化
	GDExtensionBool GDE_EXPORT example_library_init(GDExtensionInterfaceGetProcAddress p_get_proc_address, const GDExtensionClassLibraryPtr p_library, GDExtensionInitialization *r_initialization)
	{
		godot::GDExtensionBinding::InitObject init_obj(p_get_proc_address, p_library, r_initialization);

		init_obj.register_initializer(initialize_example_module);
		init_obj.register_terminator(uninitialize_example_module);
		init_obj.set_minimum_library_initialization_level(MODULE_INITIALIZATION_LEVEL_SCENE);

		return init_obj.init();
	}
}
```

## 5. 构建写好的 Extension

手工编写 SCons 用于构建的 SConstruct 文件并不容易，作为示例项目，只需使用准备好的[这个硬编码的 SConstruct 文件](https://docs.godotengine.org/zh-cn/4.x/_downloads/45a3f5e351266601b5e7663dc077fe12/SConstruct)。

将 SConstruct 文件放在`工程目录`下，然后在`工程目录`下执行 `scons`：

```bash
scons platform=系统名
# 系统名替换成实际系统名，windows就写windwos，linux就写linux
```

编译完在 `Godot工程目录` 下会多出一个 `bin` 目录，编译好的 Extension 就在里面。

## 6. 配置 Extension

在刚刚多出来的 `bin` 目录下新建一个名为 `gdexample.gdextension` 的文件，实际使用时将文件名中的 `gdexample` 改名为你的 Extension 名。该文件用于告知 Godot 要如何加载 Extension ，内容如下：

```
[configuration]

# 调用哪个函数来注册你的 Extension ，取值取决于你在 register_types.cpp 中定义的初始化函数
entry_symbol = "example_library_init"
# 允许加载该 Extension 的最低 Godot 版本
compatibility_minimum = "4.2"
# 开启热重载
reloadable = true

[libraries]

# 这一部分用于告知 Godot 要加载哪些动态库文件
# 以下列出了所有系统和cpu架构的可能选择，只需要保留你电脑对应的两行即可
# 例如我是linux系统，x86_64架构，则只需要保留以下两行
# linux.debug.x86_64 = "res://bin/libgdexample.linux.template_debug.x86_64.so"
# linux.release.x86_64 = "res://bin/libgdexample.linux.template_release.x86_64.so"
# 文件名中的libgdexample在实际使用时要根据实际库文件名做修改

macos.debug = "res://bin/libgdexample.macos.template_debug.framework"
macos.release = "res://bin/libgdexample.macos.template_release.framework"
windows.debug.x86_32 = "res://bin/libgdexample.windows.template_debug.x86_32.dll"
windows.release.x86_32 = "res://bin/libgdexample.windows.template_release.x86_32.dll"
windows.debug.x86_64 = "res://bin/libgdexample.windows.template_debug.x86_64.dll"
windows.release.x86_64 = "res://bin/libgdexample.windows.template_release.x86_64.dll"
linux.debug.x86_64 = "res://bin/libgdexample.linux.template_debug.x86_64.so"
linux.release.x86_64 = "res://bin/libgdexample.linux.template_release.x86_64.so"
linux.debug.arm64 = "res://bin/libgdexample.linux.template_debug.arm64.so"
linux.release.arm64 = "res://bin/libgdexample.linux.template_release.arm64.so"
linux.debug.rv64 = "res://bin/libgdexample.linux.template_debug.rv64.so"
linux.release.rv64 = "res://bin/libgdexample.linux.template_release.rv64.so"
android.debug.x86_64 = "res://bin/libgdexample.android.template_debug.x86_64.so"
android.release.x86_64 = "res://bin/libgdexample.android.template_release.x86_64.so"
android.debug.arm64 = "res://bin/libgdexample.android.template_debug.arm64.so"
android.release.arm64 = "res://bin/libgdexample.android.template_release.arm64.so"

[dependencies]

# 这一部分用来引入第三方库的动态库文件的依赖
# 因为这个示例项目没用到第三方库，因此为空

[icons]

# 这一部分用来给你 Extension 中的 Node 设置图标

GDExample = "res://icon.svg"
```

## 7. 开始使用你的 Extension

在做好上面一切准备后，`工程目录`下应该有如下内容：

```
工程目录/
|
+--Godot工程目录/
|   |
|   +--bin/
|       |
|       +--gdexample.gdextension
|
+--godot-cpp/
|
+--src/
|   |
|   +--register_types.cpp
|   +--register_types.h
|   +--gdexample.cpp
|   +--gdexample.h
```

用你的 Godot 打开工程，尝试新建一个 GDExample 节点吧！

如果一切无误，你可以看到该节点有 speed 和 amplitude 两个属性，以及一个 position_changed 信号。

# 下一步

以上内容展示了构建 GDExtension 的基础部分，现在你可以尝试编写更复杂脚本来控制 Godot 中的节点了。

每当你对 Extension 做出了修改，只需要在`工程目录`下重新[构建写好的 Extension](#5-构建写好的-Extension)，再回到编辑器中，刚刚的改动就会被加载到编辑器中了。
