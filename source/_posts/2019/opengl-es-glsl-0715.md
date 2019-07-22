---
title: "OpenGL ES学习--着色语言"
catalog: true
toc_nav_num: true
date: 2019-07-15 19:09:32
subtitle: "About OpenGL ES"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS

---

> 着色器是OpenGL ES 3.0 API的一个基础核心概念，每个OpenGL ES 3.0程序都需要一个顶点着色器和一个片段着色器，以渲染有意义的图片。所以下面介绍开发着色器的着色语言-GLSL。

# 着色器版本规范

OpenGL ES 3.0顶点着色器和片段着色器的第一行总是声明着色器的版本。在之前的代码中，我是采用如下语法说明着色器使用OpenGL ES着色语言的版本3.0：
``` objc
#version 300 es
```
没有声明版本号的着色器被认定使用OpenGL ES着色语言1.00版本。着色语言1.00版本用于OpenGL ES 2.0。对于OpenGL ES 3.0，规范的作者决定匹配API和着色语言的版本号，所以版本号直接从1.00跳到3.00的原因。

# 变量和变量类型

在计算机图形中，两个基本数据组成了变换的基础：向量和矩阵。这两种数据类型也是OpenGL ES着色语言中的核心。

变量分类 | 类型 | 描述
-|-|-
标量 | float, int, uint, bool | 用于浮点、整数、无符号整数和布尔值的基于标量的数据类型
浮点向量 | float, vec2, vec3, vec4 | 有1、2、3、4个分量的基于浮点的向量类型
整数向量 | int, ivec2, ivec3, ivec4 | 有1、2、3、4个分量的基于整数的向量类型
无符号整数向量 | uint, uvec2, uvec3, uvec4 | 有1、2、3、4个分量的基于无符号整数的向量类型
布尔向量 | bool, bvec2, bvec3, bvec4 | 有1、2、3、4个分量的基于布尔的向量类型
矩阵 | mat2, mat3, mat4, max2x3, max3x2 | 2x2, 3x3, 4x4, 2x3, 3x2的基于浮点的矩阵

着色语言中的变量必须以某个类型声明。变量可以在声明时或者声明以后初始化。初始化通过使用构造器进行，构造器也用于类型转换。

# 变量构造器

OpenGL ES 着色语言在类型转换方面有非常严格的规定。变量只能赋值为相同类型的其他变量或者与相同类型的变量进行运算。
标量：
``` objc
float myFloat = 1.0;
bool myBool = true; 

myFloat = Float(myBool);    // bool -> float
myBool = bool(myFloat);     // float -> bool
```

向量：
``` objc
vec4 myVec4 = vec4(1.0);        // myVec4 = {1.0, 1.0, 1.0, 1.0}
vec3 myVec3 = vec3(1.0, 0.0, 0.5);

vec3 temp = vec3(myVec3);
vec2 myVec2 = vec2(myVec3);     // myVec2 = {myVec3.x, myVec3.y}

myVec4 = vec4(myVec2, temp);   myVec4 = {myVec2.v, myVec2.y, temp.x, temp.y}
```

矩阵：
``` objc
// 矩阵以列优先顺序存储
mat3 myMat3 = mat3(1.0, 2.0, 3.0,
                   4.0, 5.0, 6.0,
                   7.0, 8.0, 9.0)
// 生成的矩阵是：
[1.0 4.0 7.0
 2.0 5.0 8.0
 3.0 6.0 9.0]
```

# 向量和矩阵分量

向量的单独分量可以用两种方式访问：使用“.”运算符或者通过数组下标。根据组成向量的分量数量，每个分量可以通过使用{x, y, z, w}、{r, g, b, a}或者{s, t, p, q}组合访问。三种不同命名方案的原因是向量可以互换地表示数学上的向量、颜色、和纹理坐标。

``` objc
vec3 myVec3 = vec3(0.0, 1.0, 2.0)
vec3 temp;

temp = myVec3.xyz;      // temp = {0.0, 1.0, 2.0}
temp = myVec3.xxx;      // temp = {0.0, 0.0, 0.0}
temp = myVec3.zyx;      // temp = {2.0, 1.0, 0.0   }
```

除了使用 "." 操作符之外，还可以使用数组下标操作。在使用数组下标操作时，元素 [0] 对应的是 x，元素 [1] 对应 y，以此类推。值得注意的是，在 OpenGL ES 2.0 中的某些情况下，数组下标不支持使用非常数的整型表达式（如使用整型变量索引），这是因为对于向量的动态索引操作，某些硬件设备处理起来很困难。在 OpenGL ES 2.0 中仅对 uniform 类型的变量支持这种动态索引。

矩阵可以看成是由一些向量组成。例如，mat2可以看做是两个vec2，mat3可以看做是3个vec3，等等。对于矩阵，单独的列可以用数组下标运算符“[]”选择，然后每个向量可以通过向量访问行为来访问。

``` objc
mat4 myMat4 = mat4(1.0);
vec4 col0 = myMat4[0];      // 矩阵中的向量
float m1_1 = myMat4[1][1];  // 从矩阵[1][1]位置获取元素
```

# 常量

可以将任何基本类型声明为常数变量。常数变量是着色器中不变的值。声明常量时，在声明中加入const限定符。

``` objc
const float zero = 0.0;
const vec4 red = vec4(1.0, 0.0, 0.0, 1.0);
```
声明为const的变量是只读的，不能在源代码中修改。

# 结构

着色语言也提供了声明结构的语法：

``` objc
struct fogStruct {
    vec4 color;
    float start;
    float end;
} fogVar;

fogVar = fogStruct(vec4(0.0, 1.0, 0.0, 0.0),    // color
                   0.5,                         // start
                   2.0);                        // end

vec4 color = fogVar.color;
float start = fogVar.start;
float end = fogVar.end;
```

# 数组

着色语言同样支持数组：

``` objc
float floatArray[4];
vec4 vecArray[2];

float a[4] = float[](1.0, 2.0, 3.0, 4.0);
float b[4] = float[4](1.0, 2.0, 3.0, 4.0);
vec2 c[2] = vec[2](vec2(1.0), vec2(1.0));
```

# 运算符

绝大多数运算符与C语言中一致。运算符只能出现在有相同基本类型的变量之间。像乘这样的运算符可以在浮点、向量和矩阵之间进行运算。

# 函数

函数的声明方法和C语言相同，最明显的不同之处在于函数参数的传递方法。着色语言提供特殊的限定符，定义函数是否可以修改可变参数：

限定符 | 描述
-|-
in | 默认限定符，指定参数按值传送，函数不能被修改
inout | 规定变量按照引用传入函数，如果该值被修改，它将在函数退出后变化
out | 表示该变量的值不被传入函数，但是在函数返回时被修改

对于着色语言中的函数还需注意一点：函数不能递归。这一限制的原因是，某些实现通过把函数代码真正地内嵌到位GPU生成的最终程序来实施函数调用，着色语言有意地构造为允许这种内嵌式实现，以支持没有堆栈的GPU。

# 插值限定符

顶点着色器的输出和片段着色器的输入，在没有任何限定符时，默认的插值行为是执行平滑着色。也就是说，来自顶点着色器的输出变量在图元中线性插值，片段着色器接收线性插值之后的数值作为输入。我们可以明确请求平滑着色：
``` objc
// Vertex shader output
smooth out vec3 v_color;

// Fragment shader input
smooth in vec3 v_color
```

还有另一种插值方法--平面着色。在平面着色中，图元中的值没有进行插值，而是将其中一个顶点视为驱动顶点，该顶点的值被用于图元中的所有片段：
``` objc
// Vertex shader output
flat out vec3 v_color;

// Fragment shader input
flat in vec3 v_color;
```

最后，可以用`centroid`关键字在插值器中添加另一个限定符，质心采样（centroid sampling）。本质上，使用多重采样渲染时，`centroid`关键字可用于强制插值发生在被渲染图元内部（否则，在图元的边缘可能出现伪像）：
``` objc
// Vertex shader output
smooth centroid out vec3 v_color;

// Fragment shader input
smooth centroid in vec3 v_color;
```

# 精度限定值

精度限定值使着色器作者可以指定着色器变量的计算精度。变量可以为低、中或者高精度。这些限定符用于提示编译器允许在较低的范围和精度上执行变量计算。在较低的精度上，有些OpenGL ES实现在运行着色器时可能更快，或者电源效率更高。当然，这种效率是以精度为代价的，在没有正确使用精度限定符时可能造成伪像。
> OpenGL ES规范中没有规定底层硬件必须支持多种精度，所以某个OpenGL ES实现在最高精度上进行所有运算并简单忽略限定符时完全正常的。不过，在某些实现上，使用较低精度可能带来好处。
精度限定符可以用于指定任何基于浮点或者整数的变量精度。指定精度的关键字是lowp、mediump、highp。

``` objc
highp vec4 position;
varying lowp vec4 color;
mediump float specularExp;
```

除了精度限定符之外，还有默认精度的概念。也就是说，如果变量声明时没有使用精度限定符，它将拥有该类型的默认精度：

``` objc
precision highp float;
precision mediump int;
```

在顶点着色器中，如果没有指定默认精度，则int和float的默认精度都为highp。也就是说，顶点着色器中所有没用精度限定符声明的变量都是使用最高精度。片段着色器的规则与此不同。在片段着色器中，浮点值没有默认的精度值：每个着色器必须声明一个默认的float精度，或者为每个float变量指定精度。

# 不变性

OpenGL ES着色语言中引入的`invariant`关键字可以用于任何可变的顶点着色器输出。它的必要性在于：着色器需要编译，而编译器可能进行导致指令重新排序的优化，这种指令重排意味着两个着色器之间的等价计算不能保证产生完全相同的结果。这种不一致性在多遍着色器特效时可能成为问题，在这种情况下，相同的对象用Alpha混合绘制在自身上房。如果用于计算输出位置的数值的精度不完全一样，精度差异就会导致伪像。这个问题通常表现为“深度冲突”（Z fighting），每个像素的Z（深度）精度差异导致不同遍着色相互之间有微小的偏移。
```
#version 300 es
uniform mat4 u_viewProjMatrix;
layout(location = 0) in vec4 a_vertex;
invariant gl_Position;
void main() {
    // will be the same value in all shaders with the same viewProjMatrix and vertex
    gl_Position = u_viewProMatrix * a_vertex;
}
```
也可以用`#pragma STDGL invariant(all)`执行让所有变量全部不变。

> 警告：因为编译器需要保证不变性，所以可能限制它所做的优化。因此，`invariant`限定符应该只在必要时使用；否则可能导致性能下降。由于这个原因，全局启用不变性的`#pragma`指令会应该在不变性对于所有变量都必需的时候使用。还要注意，虽然不变性表示在指定GPU上的计算会得到相同的结果，但是并不意味着计算在任何OpenGL ES实现之间不变。

# 总结

这篇文章主要介绍了OpenGL ES着色语言的一些特性。