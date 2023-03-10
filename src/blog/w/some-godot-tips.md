# 一些使用 Godot 引擎的小技巧

::: warning

本页面仅用于我的个人记录，内容比较杂乱

:::

## 在 fragment shader 里使用摄像机

- 将 Godot 中的摄像机与 shader 中的摄像机关联起来

### 相关知识

- https://www.shadertoy.com/ 中有许多非常棒的 shader ，它们所呈现的各种视觉效果很让人兴奋。同时，这也是一个 shader 宝藏，很多 shader 可以被移植到一些图形程序中，例如游戏、播放器、图像处理软件等

- 通常不同的平台和游戏引擎用着不同的 shader 语言，但都大同小异。它们语法很相似，类似 C 语言，只是在结构、各别语法或 GPU API 不同而已

- 如何将 shadertoy 中非常惊艳的，基于 SDF、Ray marching 的 fragment shader 移植到 Godot 中呢？

### 相关链接

- iq 佬：https://www.iquilezles.org/www/articles/normalsSDF/normalsSDF.htm
- 推荐 pdcxs 佬的 shader 教程：https://space.bilibili.com/10707223
- shadertoy 教程 08 - 3D 摄像机设置：https://www.bilibili.com/video/BV1PS4y1j7Xg/
- Godot docs：[将 GLSL 转换为 Godot 着色器](https://docs.godotengine.org/zh_CN/stable/tutorials/shaders/converting_glsl_to_godot_shaders.html)

### 关键代码

repo: https://github.com/HK-SHAO/Plotter

- 传递摄像机的 transform 到 shader 的 uniform

```gdscript
extends Node

@export var camera: Camera3D

var material: ShaderMaterial


func _ready() -> void:
	material = get_parent().material


func _process(delta: float) -> void:

	var camera_position := camera.transform.origin
	var camera_rotation := camera.transform.basis

	material.set_shader_parameter(
		"camera_position", camera_position)

	material.set_shader_parameter(
		"camera_rotation", camera_rotation)
```

- 修复 fragment shader 中的 UV
- `uniform float ratio` 是此画布的长宽比

```glsl
uniform float ratio = 1.0;
uniform vec3 camera_position;
uniform mat3 camera_rotation;

vec2 fixedUV(vec2 uv, float r) {
	vec2 p = 2.0 * vec2(1. - uv.x, uv.y) - 1.0;
	p = r>1.0?vec2(p.x, p.y*r):vec2(p.x/r, p.y);
	return p;
}

void vertex() {
	UV = fixedUV(UV, ratio);
}
```

- 设置 `ro`、`ca`、`rd`

```glsl
vec3 ro = camera_position;
mat3 ca = camera_rotation;
vec3 rd = ca * normalize(-vec3(UV, 1.0));
```


## 修复 shader 中的 UV

- 坐标原点从屏幕中心开始，且 `UV.x` 或 `UV.y` 介于 $[-1, 1]$ 之间
- 保证视觉上的长宽比，使得图形不被压缩或拉伸
- Godot docs：[CanvasItem 着色器](https://docs.godotengine.org/zh_CN/latest/tutorials/shaders/shader_reference/canvas_item_shader.html)

### 相关知识

在 shadertoy 中，fragment shader 通常开头会先计算 UV

```glsl:no-line-numbers
vec2 uv = (2.0 * fragCoord - iResolution.xy) / min(iResolution.x, iResolution.y);
```

### 相关链接

- [UV 坐标](https://zh.wikipedia.org/wiki/UV%E5%9D%90%E6%A0%87)

### 关键代码

repo: https://github.com/HK-SHAO/Plotter

- 使用工具脚本同步 shader 中画布的长宽比 `ratio`

```gdscript
@tool

extends Node

var material: ShaderMaterial
var control: Control


func _ready() -> void:
	control = get_parent() as Control
	material = control.material
	control.resized.connect(on_size_changed)
	action()


func action() -> void:
	var ratio := control.size.y / control.size.x
	material.set_shader_parameter("ratio", ratio)


func on_size_changed() -> void:
	action()
```

- 如果画布占满窗口，可以在 shader 中这样计算 `ratio`

```glsl:no-line-numbers
float ratio = SCREEN_PIXEL_SIZE.x / SCREEN_PIXEL_SIZE.y;
```

- 修复 UV
- `uv` 即 `UV` ， `r` 即 `ratio`

```glsl
uniform float ratio = 1.0;

vec2 fixedUV(vec2 uv, float r) {
	vec2 p = 2.0*vec2(uv.x, 1.0 - uv.y) - 1.0;
	p = r>1.0?vec2(p.x, p.y*r):vec2(p.x/r, p.y);
	return p;
}
```

- 另一种已知 shadertoy 代码计算 `uv` 的方法

```glsl
vec2 fragCoord = FRAGCOORD.xy;
vec2 resolution = 1. / SCREEN_PIXEL_SIZE.xy;
vec2 uv = (2. * fragCoord - resolution.xy) / min(resolution.x, resolution.y);
```

@include(@src/shared/license.md{3-})