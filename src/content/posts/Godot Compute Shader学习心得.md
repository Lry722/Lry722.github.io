---
title: Godot Compute Shader学习心得
published: 2024-04-12
category: 文章
tags:
  - Godot
  - Shader
---

最近又开始做 Godot 体素插件，产生了各种各样的并行计算需求，比如生成地形的 Mesh、材质等，因此这两天学习了一下 Compute Shader，记录一下学到的东西。

## 什么是 Compute Shader

Compute Shader 是一种特殊的 Shader，它不是用来渲染，而是用来进行通用计算，充分利用显卡的并行计算能力。

通过 Compute Shader，可以编写跨平台、跨硬件的通用 GPU 计算代码。

我使用的游戏引擎是 Godot，因此接下来均的讨论均基于 Godot 的 Compute Shader。

由于就刚学了两天，因此以下内容均为个人理解，难免有错，请自行分辨（

## 要学的东西

Godot 中要使用 Compute Shader，需要学两个方面的内容。

一部分是如何编写具体的 GLSL 代码，实现所需的功能。

另一部分是如何在 Godot 中加载写好 GLSL 代码，初始化所需的各种 gpu 资源，构建 compute list，最终提交 compute list，开始计算。

Godot 的官方示例中有一个[相关项目](https://github.com/godotengine/godot-demo-projects/tree/master/compute/texture)，可以用于学习完整的流程。

接下来基于这个项目来细说一下怎么使用 Compute Shader。

## GLSL 代码分析

首先来看 `water_compute.glsl` 这个文件，这是实际的执行计算的代码。

```glsl
#[compute]
#version 450
```

第一行表示这是一个 Compute Shader，第二行表示使用的 GLSL 版本号为 450，即该着色器代码遵循 GLSL 4.50 规范。

```glsl
layout(local_size_x = 8, local_size_y = 8, local_size_z = 1) in;
```

这行是每个 Compute Shader 都必须有的，这里的 in 是个关键字，不能叫别的东西。`local_size_x`、`local_size_y`、`local_size_z` 表示这个工作组会负责多大范围的计算。

我刚看到这时很奇怪，为什么要用 x， y ，z 来表示工作组的大小，而不是直接用一个 size 来表示呢？

实际上这里的 (8, 8, 1)表示每个`工作组`负责 8x8x1 的范围，由于这个案例中处理的目标是以像素为单位的，因此表示的其实是 8x8 个像素。

---

#### 什么是工作组

工作组是 Compute Shader 并行工作的核心概念，接下来的内容非常关键。

每个 Compute Shader 在执行时会分为若干个工作组，每个工作组中又有若干个线程，每个线程中执行的代码都是一样的。这里的 `layout` 指定的其实就是每个工作组的大小，即工作组中线程的数量。

每个工作组中的每个线程都有一个 Local ID，是一个三维向量。每个工作组本身又有一个 Group ID，也是个三维向量。在线程中将 Group ID 和 Local ID 相加，即可得到该线程的 Global ID。因此可以看出来，使用三维向量来分配线程，可以很方便地对任务进行抽象。

以上面的代码为例，如果你要处理一幅大小为 (w, h) 的图片，可以将其分为 (w/8, h/8) 个像素块，每个像素块中又分为 8x8 个像素，这样每个像素块就对应一个`工作组`，每个像素块中的 8x8 个像素就对应 8x8 个线程。在线程中，直接获取线程的 Global ID 的 x 和 y，即可得到该线程在图片中的位置。

既然 ID 都是三维向量，如果需要处理的东西是二维怎么办？只需像此处一样将 z 设为 1 即可。当然，你也可以将 y 设为 1,然后获取 Global ID 的 x 和 z 作为像素的坐标。

如果进一步抽象，其实处理的东西不一定要是图像，只要可以以小于三维的形式储存即可。

工作组的形式还有一些好处，例如一个工作组中的线程可以访问一块共享储存空间，以及可以通过 barrier 进行同步。这部分内容示例中没有用到，我也还不了解，这里就不展开讨论了。

---

接下来继续看代码

```glsl
layout(r32f, set = 0, binding = 0) uniform restrict readonly image2D current_image;
layout(r32f, set = 1, binding = 0) uniform restrict readonly image2D previous_image;
layout(r32f, set = 2, binding = 0) uniform restrict writeonly image2D output_image;
```

这里声明了三个 uniform 变量，并将它们绑定到了三个 uniform set 的 0 号位置上。uniform set 是通过一种叫做 `Descriptor Sets` 的方式管理 uniform 变量的方法。个人认为，可以大致的把每个 set 看成 array，binding 其实就是该 uniform 变量在 array 中的下标。

r32f 表示该 image 的每个像素是一个 32 位的浮点数，这是因为此案例中我们在处理的其实是水波的高度图，而不是颜色。

```glsl
layout(push_constant, std430) uniform Params {
	vec4 add_wave_point;
	vec2 texture_size;
	float damp;
	float res2;
} params;
```

这里声明了一个 uniform 块，遵循 std430 内存布局规则，并将其指定为 push constant。push constant 会在传递时直接进入 gpu 的寄存器，而不是显存，因此访问速度极快。

---

接下来的 main 函数中是实际的计算代码，我们不关心具体的计算内容，毕竟这次目的是学怎么使用 Compute Shader 而不是怎么模拟流体，因此只挑其中有关的几点进行说明。

```glsl
ivec2 uv = ivec2(gl_GlobalInvocationID.xy);

if ((uv.x > size.x) || (uv.y > size.y)) {
    return;
}
```

这里获取 uv 的方式就用到了之前提到的工作组的坐标的概念，通过获取 Global ID 即可得知正在处理的是哪个像素。

之所以要判断范围，是因为如果图片大小不是 8 的整倍数，会有线程被分配给不存在的像素，需要跳过。

```glsl
imageStore(output_image, uv, result);
```

最终当前像素的结果计算出来后，将其存到 image 对应 uv 坐标的像素中。

## Godot 调用 GLSL

写好 GLSL 代码后，要在 Godot 中进行实际的计算，还需要一些额外的工作。在这个案例中，相关内容均在 water_plane.gd 这个文件中，接下来就对这个文件进行分析。

首先看 `_initialize_compute_code` 函数，在 `_ready` 函数中会调用一次该函数。

```gdscript
rd = RenderingServer.get_rendering_device()

var shader_file = load("res://water_plane/water_compute.glsl")
var shader_spirv: RDShaderSPIRV = shader_file.get_spirv()
shader = rd.shader_create_from_spirv(shader_spirv)
pipeline = rd.compute_pipeline_create(shader)
```

这里先获取了 RenderingServer 的全局实例，然后加载了 GLSL 文件，将其转换为 SPIR-V 格式，再转换为 Shader。SPIR-V 是 Shader 的标准化中间表示，作为 Shader 代码和底层驱动间的桥梁。

然后创建了一个 compute pipeline，用于后续构建 compute list。

```gdscript
var tf : RDTextureFormat = RDTextureFormat.new()
tf.format = RenderingDevice.DATA_FORMAT_R32_SFLOAT
tf.texture_type = RenderingDevice.TEXTURE_TYPE_2D
tf.width = init_with_texture_size.x
tf.height = init_with_texture_size.y
tf.depth = 1
tf.array_layers = 1
tf.mipmaps = 1
tf.usage_bits = RenderingDevice.TEXTURE_USAGE_SAMPLING_BIT + RenderingDevice.TEXTURE_USAGE_COLOR_ATTACHMENT_BIT + RenderingDevice.TEXTURE_USAGE_STORAGE_BIT + RenderingDevice.TEXTURE_USAGE_CAN_UPDATE_BIT + RenderingDevice.TEXTURE_USAGE_CAN_COPY_TO_BIT
```

这里就是指定 Texture 格式的一堆属性，由于该案例中所有纹理格式都形同，因此只需要指定这一个格式。

`tf.format = RenderingDevice.DATA_FORMAT_R32_SFLOAT` 这句对应的是 GLSL 文件中 layout 中指定的 r32f。

```gdscript
for i in range(3):
    texture_rds[i] = rd.texture_create(tf, RDTextureView.new(), [])
    rd.texture_clear(texture_rds[i], Color(0, 0, 0, 0), 0, 1, 0, 1)
    texture_sets[i] = _create_uniform_set(texture_rds[i])
```

由于所需的三个 texture 均相同，因此使用一个循环批量创建。

---

然后是 `_render_process` 函数，在每次调用 `_process` 时也会调用一次该函数。

```gdscript
var push_constant : PackedFloat32Array = PackedFloat32Array()
push_constant.push_back(wave_point.x)
push_constant.push_back(wave_point.y)
push_constant.push_back(wave_point.z)
push_constant.push_back(wave_point.w)

push_constant.push_back(tex_size.x)
push_constant.push_back(tex_size.y)
push_constant.push_back(damp)
push_constant.push_back(0.0)
```

首先填充 push constant 的内容。

```gdscript
var x_groups = (tex_size.x - 1) / 8 + 1
var y_groups = (tex_size.y - 1) / 8 + 1
```

然后计算 x 和 y 方向上工作组的数量，因为每个像素块的大小是 8x8，因此工作组的数量自然为 size / 8。之所以还有额外的 -1 和 +1，是为了确保当 tex_size 不是 8 的整倍数时，能覆盖到所有像素。

```gdscript
var next_set = texture_sets[with_next_texture]
var current_set = texture_sets[(with_next_texture - 1) % 3]
var previous_set = texture_sets[(with_next_texture - 2) % 3]
```

这几行和具体的计算内容相关，不是重点，总之就是指定三个 uniform set 的顺序。

```gdscript
var compute_list := rd.compute_list_begin()
rd.compute_list_bind_compute_pipeline(compute_list, pipeline)
rd.compute_list_bind_uniform_set(compute_list, current_set, 0)
rd.compute_list_bind_uniform_set(compute_list, previous_set, 1)
rd.compute_list_bind_uniform_set(compute_list, next_set, 2)
rd.compute_list_set_push_constant(compute_list, push_constant.to_byte_array(), push_constant.size() * 4)
rd.compute_list_dispatch(compute_list, x_groups, y_groups, 1)
rd.compute_list_end()
```

这里开始构建 compute list。构建步骤中用到的函数均为 `compute_list_xxx` 的形式。

首先调用 `compute_list_begin`，获取到一个整数值。

之后绑定 pipeline 、 uniform set ，并传递 push constant。

最后调用 dispatch 指定工作组数量，其中 x_groups 和 y_groups 是之前计算出来的，因为处理的是二维图像，因此 z_groups 直接设为 1。

最后调用 `compute_list_end` 结束 compute list 的构建，pipeline 会根据 compute list 中的内容开始计算。

注意所有 `compute_list_xxx` 均需要传入 `compute_list_begin` 所返回的整数。

---

以上就是完整的使用 Compute Shader 的流程，最终的结果是水面的高度图，会轮流存在三个 Texture 中（因为每帧需要用到前两帧的数据进行计算）。

在 `_process` 中有这么两行：

```gdscript
if texture:
    texture.texture_rd_rid = texture_rds[next_texture]
```

这两行指定了当前帧中， `water_shader.gdshader` 应当读取哪个 Texture 作为输入。最终 `water_shader.gdshader` 会依据 Texture 中的高度信息计算法线，实现水面的涟漪效果。
