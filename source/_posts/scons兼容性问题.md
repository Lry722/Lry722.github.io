---
title: scons兼容性问题
date: 2024-01-03 16:51:26
category:
  - 文章
tags:
  - python
  - godot
  - scons
  - 编译
---

今天在研究 GDExtension，使用 scons 带 custom_api_file 参数编译 Godot 的时候遇到了如下报错：

```
AttributeError: 'str' object has no attribute 'decode':
File "D:\Godot\GDExtension\gdextension_cpp_example\godot_cpp\SConstruct", line 124:
env_base = Environment(tools=custom_tools)
File "C:\Users\95851\AppData\Local\Programs\Python\Python39\lib\site-packages\SCons\Environment.py", line 1244:
apply_tools(self, tools, toolpath)
File "C:\Users\95851\AppData\Local\Programs\Python\Python39\lib\site-packages\SCons\Environment.py", line 119:
_ = env.Tool(tool)
File "C:\Users\95851\AppData\Local\Programs\Python\Python39\lib\site-packages\SCons\Environment.py", line 2004:
tool(self)
File "C:\Users\95851\AppData\Local\Programs\Python\Python39\lib\site-packages\SCons\Tool_init_.py", line 265:
self.generate(env, *args, **kw)
File "C:\Users\95851\AppData\Local\Programs\Python\Python39\lib\site-packages\SCons\Tool\default.py", line 35:
for t in SCons.Tool.tool_list(env['PLATFORM'], env):
File "C:\Users\95851\AppData\Local\Programs\Python\Python39\lib\site-packages\SCons\Tool_init_.py", line 769:
c_compiler = FindTool(c_compilers, env) or c_compilers[0]
File "C:\Users\95851\AppData\Local\Programs\Python\Python39\lib\site-packages\SCons\Tool_init_.py", line 672:
if t.exists(env):
File "C:\Users\95851\AppData\Local\Programs\Python\Python39\lib\site-packages\SCons\Tool\msvc.py", line 330:
return msvc_setup_env_tool(env, tool=tool_name)
File "C:\Users\95851\AppData\Local\Programs\Python\Python39\lib\site-packages\SCons\Tool\MSCommon\vc.py", line 1588:
MSVC.SetupEnvDefault.register_tool(env, tool, msvc_exists)
File "C:\Users\95851\AppData\Local\Programs\Python\Python39\lib\site-packages\SCons\Tool\MSCommon\MSVC\SetupEnvDefault.py", line 77:
_initialize(env, msvc_exists_func)
File "C:\Users\95851\AppData\Local\Programs\Python\Python39\lib\site-packages\SCons\Tool\MSCommon\MSVC\SetupEnvDefault.py", line 72:
_Data.msvc_installed = msvc_exists_func(env)
File "C:\Users\95851\AppData\Local\Programs\Python\Python39\lib\site-packages\SCons\Tool\MSCommon\vc.py", line 1545:
vcs = get_installed_vcs(env)
File "C:\Users\95851\AppData\Local\Programs\Python\Python39\lib\site-packages\SCons\Tool\MSCommon\vc.py", line 1135:
VC_DIR = find_vc_pdir(env, ver)
File "C:\Users\95851\AppData\Local\Programs\Python\Python39\lib\site-packages\SCons\Tool\MSCommon\vc.py", line 888:
comps = find_vc_pdir_vswhere(msvc_version, env)
File "C:\Users\95851\AppData\Local\Programs\Python\Python39\lib\site-packages\SCons\Tool\MSCommon\vc.py", line 848:
lines = cp.stdout.decode("mbcs").splitlines()
```

问 AI 后得知，原因是在 Python 3 中，subprocess.Popen 的 stdout 默认返回的是解码后的字符串，而不是原始的字节流。而在上述代码片段中，它尝试对已经解码过的字符串再次进行解码。

解决方案为在 `subprocess.Popen` 的调用参数中加上 `encoding=None` 来获取未经解码的字节串。

共有两处地方需要修改，根据报错找到后改掉即可正常使用。
