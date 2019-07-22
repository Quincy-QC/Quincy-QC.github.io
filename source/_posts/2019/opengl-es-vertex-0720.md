---
title: "OpenGL ES学习--顶点属性、顶点数组和缓冲区对象"
catalog: true
toc_nav_num: true
date: 2019-07-20 18:28:11
subtitle: "About OpenGL ES"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS

---

> 顶点数据也称作顶点属性，指定每个顶点的数据。这种逐顶点数据可以为每个顶点指定，也可以用于所有顶点的常量。例如，如果想要绘制一个固定颜色的三角形，可以指定一个常量值，用于三角形的全部3个顶点。但是，组成三角形的3个顶点的位置不同，所以我们必须指定一个顶点数组来存储3个位置值。

# 指定顶点属性数据

顶点属性数据可以用一个顶点数组对每个顶点指定，也可以将一个常量值用于一个图元的所有顶点。

所有OpenGL ES 3.0实现必须支持最少16个顶点属性，可以通过`glGet*`函数查询特定支持的顶点属性的准确数量：

``` objc
GLint maxVertexAttribs; 
glGetIntegerv(GL_MAX_VERTEX_ATTRIBS, &maxVertexAttribs);
```

## 常量顶点属性

常量顶点属性对于一个图元的所有顶点都相同，所以对一个图元的所有顶点只需指定一个值。

``` objc
// 加载indx指定的通用顶点属性(x, 0.0, 0.0, 1.0)
glVertexAttrib1f(GLuint indx, GLfloat x);
glVertexAttrib1fv(GLuint indx, const GLfloat *values);

// 加载indx指定的通用顶点属性(x, y, 0.0, 1.0)
glVertexAttrib2f(GLuint indx, GLfloat x, GLfloat y);
glVertexAttrib2fv(GLuint indx, const GLfloat *values);

// 加载indx指定的通用顶点属性(x, y, z, 1.0)
glVertexAttrib3f(GLuint indx, GLfloat x, GLfloat y, GLfloat z);
glVertexAttrib3fv(GLuint indx, const GLfloat *values);

// 加载indx指定的通用顶点属性(x, y, z, w)
glVertexAttrib4f(GLuint indx, GLfloat x, GLfloat y, GLfloat z, GLfloat w);
glVertexAttrib4fv(GLuint indx, const GLfloat *values);
```

## 顶点数组

顶点数组指定每个顶点的属性，是保存在应用程序地址空间的缓冲区。它们作为顶点缓冲对象的基础，提供指定顶点属性数据的一种高效、灵活的手段。

``` objc
/**
 指定顶点属性

 @param indx#> 指定通用顶点属性索引 description#>
 @param size#> 顶点数组中为索引引用的顶点属性所指定的分量数量，有效值为1~4 description#>
 @param type#> 数据格式 description#>
 @param normalized#> 用于表示非浮点数据格式类型在转换为浮点值时是否应该规范化 description#>
 @param stride#> 每个顶点由size指定的顶点属性分量顺序存储。stride指定顶点索引I和I+1表示的顶点数据之间的位移。如果stride为0，则每个顶点的属性数据顺序存储。如果stride大于0，则使用该值作为获取下一个索引表示的顶点数据的跨距 description#>
 @param ptr#> 如果使用客户端顶点数组，则是保存顶点属性数据的缓冲区指针。如果使用顶点缓冲区对象，则表示该缓冲区内的偏移量 description#>
 @return void
 */
glVertexAttribPointer(GLuint indx, GLint size, GLenum type, GLboolean normalized, GLsizei stride, const GLvoid *ptr);

/**
 指定顶点属性，数据格式类型被当做整数对待

 @param index#> 指定通用顶点属性索引 description#>
 @param size#> 顶点数组中为索引引用的顶点属性所指定的分量数量，有效值为1~4 description#>
 @param type#> 数据格式 description#>
 @param stride#> 每个顶点由size指定的顶点属性分量顺序存储。stride指定顶点索引I和I+1表示的顶点数据之间的位移。如果stride为0，则每个顶点的属性数据顺序存储。如果stride大于0，则使用该值作为获取下一个索引表示的顶点数据的跨距 description#>
 @param pointer#> 如果使用客户端顶点数组，则是保存顶点属性数据的缓冲区指针。如果使用顶点缓冲区对象，则表示该缓冲区内的偏移量 description#>
 @return void
 */
glVertexAttribIPointer(GLuint index, GLint size, GLenum type, GLsizei stride, const GLvoid *pointer);
```

分配和存储顶点属性数据有两种常用的方法：
- 在一个缓冲区中存储顶点属性--这种方法称为结构数组。结构表示顶点的所有属性，每个顶点有一个属性的数组。
- 在单独的缓冲区中保存每个顶点属性--这种方法称为数组结构。

### 哪种存储顶点属性的方法高效

在大部分情况下，答案是结构数组。原因是，每个顶点的属性数据可以顺序方法读取，这最有可能造成高效的内存访问模式。使用结构数组的缺点在应用程序需要修改特定属性时变得很明显。如果顶点属性数据的一个自己需要修改，这将造成顶点缓冲区的跨距更新。当顶点缓冲区以缓冲区对象的形式提供时，需要重新加载整个顶点属性缓冲区。可以通过将动态的顶点属性保存在单独的缓冲区来避免这种效率低下的情况。

### 顶点属性使用哪种数据格式

`glVertexAttribPointer`中的`type`参数指定的顶点属性格式不仅影响顶点属性数据的图形内存存储需求，而且影响整体性能，这是渲染帧所需内存带宽的一个函数。数据空间占用越小，需要的内存带宽越小。OpenGL ES 3.0支持名为`GL_HALF_FLOAT`的16位浮点顶点格式，建议尽可能使用。纹理坐标、法线、副法线、切向量等都是使用`GL_HALF_FLOAT`存储每个分量的候选。颜色可以存储为`GL_UNSIGNED_BYTE`，每个顶点颜色具有4个分量。顶点位置可以存储为`GL_FLOAT`。

### `glVertexAttribPointer`中的规范化标识如何工作

在用于顶点着色器之前，顶点属性在内部保存为单精度浮点数，如果数据类型表示顶点属性不是浮点数，顶点属性将在用于顶点着色器之前转换为单精度浮点数。规范化标志控制非浮点顶点属性数据到单精度浮点值的转换。如果规范化标识为假，则顶点数据被直接转换为浮点数。这类似于将非浮点类型的变量转换为浮点变量。例子：

``` objc
GLfloat f;
GLbyte b;
f = (GLfloat)b;     // f represents values in the range [-128.0, 127.0]
```

如果规范化标志位真，且如果数据类型为`GL_BYTE`、`GL_SHORT`或者`GL_FIXED`，则顶点数据被映射到[-1.0, 1.0]范围内，如果数据类型为`GL_UNSIGNED_BYTE`或`GL_UNSIGNED_SHORT`，则被映射到[0.0, 1.0]范围内。

下表说明了设置规范化标志时非浮点数据类型的转换，表中第二列的`c`值指的是第一列中指定格式的一个值：

顶点数据格式 | 转换为浮点数
- | -
GL_BYTE | max(c/(2<sup>7</sup>-1), -1.0)
GL_UNSIGNED_BYTE | c/(2<sup>8</sup>-1)
GL_SHORT | max(c/(2<sup>16</sup>-1), -1.0)
GL_UNSIGNED_SHORT | c/(2<sup>16</sup>-1)
GL_FIXED | c/(2<sup>16</sup>)
GL_FLOAT | c
GL_HALF_FLOAT_OES | c

在顶点着色器中，也有可能按照整数的形式访问整数型顶点属性数据，而不是将其转换为浮点数，可以使用`glVertexAttribIPointer`函数。

### 在常量顶点属性和顶点数组之间选择

应用程序可以让OpenGL ES使用常量数据或者来自顶点数组的数据。`glEnableVertexAttribArray`和`glDisableVertexAttribArray`命令分别用于启用和禁用通用顶点属性数组。如果某个通用属性索引的顶点属性数组被禁用，将使用为该索引指定的常量顶点属性数据。

``` objc
// glsl.vsh
#version 300 es
layout(location = 0) in vec4 a_color;
layout(location = 1) in vec4 a_position;
out vec4 v_color;
void mian() {
    v_color = a_color;
    gl_Position = a_position;
}
```

``` objc
// glsl.fsh
#version 300 es
precision mudiump float;
in vec4 v_color;
out vec4 o_fragColor;
void main() {
    o_fragColor = v_color;
}
```

``` objc
GLfloat color[4] = {1.0f, 0.0f, 0.0f, 1.0f};
GLfloat vertexPos[9] = {0.0f, 0.5f, 0.0f,
                        -0.5f, -0.5f, 0.0f,
                        0.5f, -0.5f, 0.0f};
glVertexAttrib4fv(0, color);
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 0, vertexPos);
glEnableVertexAttribArray(1);
glDrawArrays(GL_TRIANGLES, 0, 3);
glDisableVertexAttribArray(1);
```

代码示例中使用的顶点属性`color`是一个常量，用`glVertexAttrib4fv`指定，不启用顶点属性数组0。`vertexPos`属性用`glVertexAttribPointer`以一个顶点数组指定，并用`glEnableVertexAttribArray`启用数组。`color`值对于所绘制的三角形的所有顶点均相同，而`vertexPos`属性对于三角形的各个顶点可以不同。

# 在顶点着色器中声明顶点属性变量

在顶点着色器中，变量通过使用`in`限定符声明为顶点属性。属性变量也可以包含一个布局限定符，提供属性索引：

``` objc
layout(location = 0) in vec4 a_position;
layout(location = 1) in vec2 a_texcoord;
layout(location - 2) in vec3 a_normal;
```

属性可以在顶点着色器内部声明--但是如果没有使用，就不会被认为是活动属性，从而不会被计入限制。如果在顶点着色器中使用的属性数量大于`GL_MAX_VERTEX_ATTRIBS`，这个顶点着色器就无法链接。

当程序成功链接，我们就需要找出连接到该程序的顶点着色器使用的活动顶点属性数量。注意：这一步骤只需要在我们对属性不使用布局限定符时才有必要。
首先通过`glGetProgramiv`函数传入`GL_ACTIVE_ATTRIBUTE`获取活动的属性数量，传入`GL_ACTIVE_ATTRIBUTE_MAX_LENGTH`获取属性的最长字符数量，最后使用`glGetActiveAttrib`函数查询顶点着色器各属性索引点对应的属性变量名称：

``` objc
GLint attribsCount;
GLint attribMaxLength;
glGetProgramiv(program, GL_ACTIVE_ATTRIBUTES, &attribsCount);
glGetProgramiv(program, GL_ACTIVE_ATTRIBUTE_MAX_LENGTH, &attribMaxLength);
for (int i = 0; i < attribsCount; i++) {
    GLsizei length;
    GLint size;
    GLenum type;
    GLchar *name = malloc(attribMaxLength * sizeof(GLchar));
    glGetActiveAttrib(program, i, attribMaxLength, &length, &size, &type, name);
    NSLog(@"length: %d, size: %d, type: %u, name: %s", length, size, type, name);
}
```

## 将顶点属性绑定到顶点着色器中的属性变量

在OpenGL ES 3.0中，可以使用3种方法将通用顶点属性索引映射到顶点着色器中的一个属性变量名称：
- 索引可以在顶点着色器源代码中用`layout(location = N)`限定符指定（推荐）；
- OpenGL ES 3.0将通用顶点属性索引绑定到属性名称；
- 应用程序可以将顶点属性索引绑定到属性名称；

将属性绑定到一个位置的最简单方法是简单地使用`layout(location = N)`限定符，这种方法需要的代码最少。但是，在某些情况下，其他两个选项可能更合适。`glBindAttribLocation`命令可用于将通用顶点属性索引绑定到顶点着色器中的一个属性变量。这种绑定在下一次程序链接时生效--不会改变当前链接的程序中使用的绑定。

``` objc
/**
 将通用顶点属性索引绑定到属性变量

 @param program#> 程序对象 description#>
 @param index#> 通用顶点属性索引 description#>
 @param name#> 属性变量名称 description#>
 @return void
 */
glBindAttribLocation(GLuint program, GLuint index, const GLchar *name);
```

如果之前绑定了`name`，则它所指定的绑定被索引代替。`glBindAttribLocation`甚至可以在顶点着色器连接到程序对象之前调用，因此，这个调用可以用于绑定任何属性名称。不存在的属性名称或者在连接到程序对象的顶点着色器中不活动的属性将被忽略。

另一个选项是让OpenGL ES 3.0将属性变量名称绑定到一个通用顶点属性索引。这种绑定在程序链接时进行，在链接阶段，OpenGL ES 3.0实现为每个属性变量执行如下操作：对于每个属性变量，检查是否已经通过`glBindAttribLocation`。如果指定了一个绑定，则使用指定的对应属性索引。否则，OpenGL ES实现将分配一个通用顶点属性索引。
这种分配特定于实现；也就是说，在一个OpenGL ES 3.0实现中和在另一个实现中可能不同。应用程序可以使用`glGetAttribLocation`命令查询分配的绑定。

``` objc
/**
 查询属性索引

 @param program#> 程序对象 description#>
 @param name#> 属性变量名称 description#>
 @return 属性索引
 */
glGetAttribLocation(GLuint program, const GLchar *name);
```

`glGetAttribLocation`返回`program`定义的程序对象最后一次链接时绑定到属性变量`name`的通用属性索引。如果`name`不是一个活动属性变量，或者`program`不是一个有效的程序对象，或者没有链接成功，则返回-1，表示无效的属性索引。

# 顶点缓冲区对象

使用顶点数组指定的顶点数据保存在客户内存中。在进行`glDrawArrays`或者`glDrawElements`等绘图调用时，这些数据必须从客户内存复制到图形内存。但是，如果我们没有必要在每次绘图调用时都复制顶点数据，而是在图形内存中缓存这些数据，那就好得多了。这种方法可以显著地改进渲染性能，也会降低内存带宽和电力消耗序曲，对于手持设备相当重要。这是顶点缓冲区对象发挥的地方。顶点缓冲区对象使OpenGL ES 3.0应用程序可以在高性能的图形内存中分配和缓存顶点数据，并从这个内存进行渲染，从而避免在每次绘制图元的时候重新发送数据。不仅是顶点数据，描述图元顶点索引、作为`glDrawElements`参数传递的元素索引也可以缓存。

OpenGL ES 3.0支持两类缓冲区对象，用于指定顶点和图元数据：数组缓冲区对象和元素数组缓冲区对象。`GL_ARRAY_BUFFER`标志指定的数组缓冲区对象用于创建保存顶点数据的缓冲区对象。`GL_ELEMENT_ARRAY_BUFFER`标志指定的元素数组缓冲区对象用于创建保存图元索引的缓冲区对象。

> 为了得到最佳性能，推荐OpenGL ES 3.0应用程序对顶点属性数据和元素索引使用顶点缓冲区对象。

 在使用缓冲对象渲染之前，需要分配缓冲区对象并将顶点数据和元素索引上传到相应的缓冲区对象。

``` objc
// vertex_t是自定义顶点数据类型
void initVertexBufferObjects(vertex_t *vertexBuffer,
                             GLushort *indices,
                             GLuint numVertices,
                             GLuint numIndices,
                             GLuint *vboIds) {
    glGenBuffers(2, vboIds);
    
    glBindBuffer(GL_ARRAY_BUFFER, vboIds[0]);
    glBufferData(GL_ARRAY_BUFFER, numVertices * sizeof(vertex_t), vertexBuffer, GL_STATIC_DRAW);
    
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, vboIds[1]);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, numIndices * sizeof(GLushort), indices, GL_STATIC_DRAW);
}
```

上述代码创建两个缓冲区对象：一个用于保存实际的顶点属性数据，另一个用于保存组成图元的元素索引。数组缓冲区对象用于保存一个或者多个图元的顶点属性数据，元素数组缓冲区对象保存一个或者多个图元的索引。

``` objc
/**
 创建缓冲区对象

 @param n#> 返回缓冲区对象名称数量 description#>
 @param buffers#> 指向n个条目的数组指针，该数组是分配的缓冲区对象返回的位置 description#>
 @return void
 */
glGenBuffers(GLsizei n, GLuint *buffers);

/**
 指定当前缓冲区对象

 @param target#> GL_ARRAY_BUFFER, GL_ELEMENT_ARRAY_BUFFER, GL_COPY_READ_BUFFER, GL_COPY_WRITE_BUFFER, GL_PIXEL_PACK_BUFFER, GL_PIXEL_UNPACK_BUFFER, GL_TRANSFROM_FEEDBACK_BUFFER, GL_UNIFORM_BUFFER description#>
 @param buffer#> 分配给目标当做当前对象的缓冲区对象 description#>
 @return void
 */
glBindBuffer(GLenum target, GLuint buffer);

/**
 顶点数组数据或者元素数组数据存储创建和初始化

 @param target#> GL_ARRAY_BUFFER, GL_ELEMENT_ARRAY_BUFFER, GL_COPY_READ_BUFFER, GL_COPY_WRITE_BUFFER, GL_PIXEL_PACK_BUFFER, GL_PIXEL_UNPACK_BUFFER, GL_TRANSFROM_FEEDBACK_BUFFER, GL_UNIFORM_BUFFER description#>
 @param size#> 缓冲区数据存储大小，以字节数表示 description#>
 @param data#> 应用程序提供的缓冲区数据的指针，可为NULL值，表示保留的数据存储不进行初始化。如果是一个有效的指针，则其内容被复制到分配的数据存储 description#>
 @param usage#> 详见下表 description#>
 @return void
 */
glBufferData(GLenum target, GLsizeiptr size, const GLvoid *data, GLenum usage);

/**
 顶点数组数据或者元素数组数据存储初始化或者更新

 @param target#> GL_ARRAY_BUFFER, GL_ELEMENT_ARRAY_BUFFER, GL_COPY_READ_BUFFER, GL_COPY_WRITE_BUFFER, GL_PIXEL_PACK_BUFFER, GL_PIXEL_UNPACK_BUFFER, GL_TRANSFROM_FEEDBACK_BUFFER, GL_UNIFORM_BUFFER description#>
 @param offset#> 缓冲区数据存储中的偏移 description#>
 @param size#> 被修改的数据存储字节数 description#>
 @param data#> 需要被复制到缓冲区对象数据存储的客户数据指针 description#>
 @return void
 */
glBufferSubData(GLenum target, GLintptr offset, GLsizeiptr size, const GLvoid *data)
```

缓冲区使用枚举值 | 描述
- | -
GL_STATIC_DRAW | 缓冲区对象数据将被修改一次，使用多次，以绘制图元或者指定图像
GL_STATIC_READ | 缓冲区对象数据将被修改一次，使用多次，以从OpenGL ES读回数据。从OpenGL ES读回的数据将从应用程序中查询
GL_STATIC_COPY | 缓冲区对象数据将被修改一次，使用多次，以从OpenGL ES读回数据。从OpenGL ES读回的数据将直接用作绘制图元或者修改图像的信息来源
GL_DYNAMIC_DRAW | 缓冲区对象数据将被重复修改，使用多次，以绘制图元或者指定图像
GL_DYNAMIC_READ | 缓冲区对象数据将被重复修改，使用多次，以从OpenGL ES读回数据。从OpenGL ES读回的数据中将从应用程序中查询
GL_DYNAMIC_COPY | 缓冲区对象数据将被重复修改，使用多次，以从OpenGL ES读回数据。从OpenGL ES读回的数据将直接用作绘制图元或者修改图像的信息来源
GL_STREAM_DRAW | 缓冲区对象数据将被修改一次，只使用少数几次，以绘制图元或者指定图像
GL_STREAM_READ | 缓冲区对象数据将被修改一次，只使用少数几次，以从OpenGL ES读回数据。从OpenGL ES读回的数据将从应用程序中查询
GL_STREAM_COPY | 缓冲区对象数据将被修改一次，只使用少数几次，以从OpenGL ES读回数据。从OpenGL ES读回的数据将直接用作绘制图元或者修改图像的信息来源

在用`glBufferData`或者`glBufferSubData`初始化或者更新缓冲区对象数据存储之后，客户数据存储不再需要，可以释放。对于静态的几何形状，应用程序可以释放客户数据存储，减少应用程序消耗的系统内存。对于动态几何形状，这可能无法做到。

下面是个使用和不使用缓冲区对象进行的图元绘制：

``` objc
// glsl.vsh
#version 300 es
layout(location = 0) in vec4 vPosition;
layout(location = 1) in vec4 vColor;
uniform float offsetLoc;
out vec4 v_color;

void main() {
    v_color = vColor;
    gl_Position = vec4(vPosition.x + offsetLoc, vPosition.yzw);
}
```

``` objc
// glsl.fsh
#version 300 es
precision mediump float;
in vec4 v_color;
out vec4 fragColor;

void main() {
    fragColor = v_color;
}
```

``` objc
void DrawPrimitiveWithoutVBOs() {
    GLint vertex_position_size = 3;
    GLint vertex_color_size = 4;
    GLushort indices[3] = {0, 1, 2};
    GLfloat vertices[] = {
        -0.5f, 0.5f, 0.0f,
        1.0f, 0.0f, 0.0f, 1.0f,
        -1.0f, -0.5f, 0.0f,
        0.0f, 1.0f, 0.0f, 1.0f,
        0.0f, -0.5f, 0.0f,
        0.0f, 0.0f, 1.0f, 1.0f
    };
    GLfloat *vtxBuf = vertices;
    GLsizei vtxStride = sizeof(GLfloat) * (vertex_color_size + vertex_position_size);
    
    glBindBuffer(GL_ARRAY_BUFFER, 0);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);
    
    glEnableVertexAttribArray(0);
    glEnableVertexAttribArray(1);
    
    glVertexAttribPointer(0, vertex_position_size, GL_FLOAT, GL_FALSE, vtxStride, vtxBuf);
    vtxBuf += vertex_position_size;
    glVertexAttribPointer(1, vertex_color_size, GL_FLOAT, GL_FALSE, vtxStride, vtxBuf);
    
    glDrawElements(GL_TRIANGLES, 3, GL_UNSIGNED_SHORT, indices);
    
    glDisableVertexAttribArray(0);
    glDisableVertexAttribArray(1);
}

void DrawPrimitiveWithVBOs() {
    GLvoid *offset = 0;
    GLuint vboIds[2];
    GLint vertex_position_size = 3;
    GLint vertex_color_size = 4;
    GLushort indices[3] = {0, 1, 2};
    GLint vtxStride = sizeof(GLfloat) * (vertex_position_size + vertex_color_size);
    GLfloat vertices[] = {
        -0.5f, 0.5f, 0.0f,
        1.0f, 0.0f, 0.0f, 1.0f,
        -1.0f, -0.5f, 0.0f,
        0.0f, 1.0f, 0.0f, 1.0f,
        0.0f, -0.5f, 0.0f,
        0.0f, 0.0f, 1.0f, 1.0f
    };
    GLfloat *vtxBuf = vertices;

    glGenBuffers(2, vboIds);
    glBindBuffer(GL_ARRAY_BUFFER, vboIds[0]);
    glBufferData(GL_ARRAY_BUFFER, vtxStride * 3, vtxBuf, GL_STATIC_DRAW);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, vboIds[1]);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(GLushort) * 3, indices, GL_STATIC_DRAW);
    
    glEnableVertexAttribArray(0);
    glEnableVertexAttribArray(1);
    
    glVertexAttribPointer(0, vertex_position_size, GL_FLOAT, GL_FALSE, vtxStride, offset);
    offset += vertex_position_size * sizeof(GLfloat);
    glVertexAttribPointer(1, vertex_color_size, GL_FLOAT, GL_FALSE, vtxStride, offset);
    
    glDrawElements(GL_TRIANGLES, 3, GL_UNSIGNED_SHORT, 0);
    
    glDisableVertexAttribArray(0);
    glDisableVertexAttribArray(1);
    
    glBindBuffer(GL_ARRAY_BUFFER, 0);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);
}

// 调用
DrawPrimitiveWithoutVBOs();
glUniform1f(0, 1.0f);
DrawPrimitiveWithVBOs();
```

上述代码中，我们对一个顶点的所有属性使用相同的缓冲区对象。使用`GL_ARRAY_BUFFER`缓冲区对象时，`glVertexAttribPointer`的`pointer`参数从指向实际数据的一个指针变成用`glBufferData`分配的顶点缓冲区存储中以字节表示的偏移量。类似地，如果使用有效的`GL_ELEMENT_ARRAY_BUFFER`对象，则`glDrawElements`中的`indices`参数从指向实际元素索引的指针变成用`glBufferData`分配的元素索引缓冲区存储中以字节表示的偏移量。

上述代码描述的存储顶点属性的是结构数组方法，也可以对每个顶点属性使用一个缓冲区对象--数组结构方法：

``` objc
void DrawPrimitiveWithVBOs() {
    GLuint vboIds[3];
    GLint vertex_position_size = 3;
    GLint vertex_color_size = 4;
    GLfloat position[] = {
        -0.5f, 0.5f, 0.0f,
        -1.0f, -0.5f, 0.0f,
        0.0f, -0.5f, 0.0f
    };
    GLfloat color[] = {
        1.0f, 0.0f, 0.0f, 1.0f,
        0.0f, 1.0f, 0.0f, 1.0f,
        0.0f, 0.0f, 1.0f, 1.0f
    };
    GLushort indices[3] = {0, 1, 2};

    glGenBuffers(3, vboIds);
    glBindBuffer(GL_ARRAY_BUFFER, vboIds[0]);
    glBufferData(GL_ARRAY_BUFFER, sizeof(GLfloat) * vertex_position_size * 3, position, GL_STATIC_DRAW);
    glBindBuffer(GL_ARRAY_BUFFER, vboIds[1]);
    glBufferData(GL_ARRAY_BUFFER, sizeof(GLfloat) * vertex_color_size * 4, color, GL_STATIC_DRAW);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, vboIds[2]);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(GLushort) * 3, indices, GL_STATIC_DRAW);
    
    glBindBuffer(GL_ARRAY_BUFFER, vboIds[0]);
    glEnableVertexAttribArray(0);
    glVertexAttribPointer(0, vertex_position_size, GL_FLOAT, GL_FALSE, sizeof(GLfloat) * vertex_position_size, 0);
    glBindBuffer(GL_ARRAY_BUFFER, vboIds[1]);
    glEnableVertexAttribArray(1);
    glVertexAttribPointer(1, vertex_color_size, GL_FLOAT, GL_FALSE, sizeof(GLfloat) * vertex_color_size, 0);
    glDrawElements(GL_TRIANGLES, 3, GL_UNSIGNED_SHORT, 0);
    
    glDisableVertexAttribArray(0);
    glDisableVertexAttribArray(1);
    
    glBindBuffer(GL_ARRAY_BUFFER, 0);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);
}
```

在应用程序结束缓冲对象的使用之后，删除它们：

``` objc
/**
 删除缓冲区对象

 @param n#> 删除的缓冲区对象数量 description#>
 @param buffers#> 包含要删除的缓冲区对象的有n个元素的数组 description#>
 @return void
 */
glDeleteBuffers(GLsizei n, const GLuint *buffers);
```

一旦缓冲区对象被删除，它就可以作为新的缓冲区对象重用，存储顶点属性或者不同图元的元素索引。

使用顶点缓冲区对象非常容易，比起顶点数组，所需要的额外工作非常少。考虑到这种功能提供的性能提升，支持顶点缓冲区对象的少量额外工作是值得的。

# 顶点数组对象

上面我们已经介绍了顶点属性加载的两种不同方式：使用客户顶点数组和使用顶点缓冲区对象。顶点缓冲区对象优于客户顶点数组，因为它们能够减少GPU和CPU之间复制的数据量，从而获得更好的性能。在OpenGL ES 3.0中引入了一个新特性，使顶点数组的使用更加高效：顶点数组对象（VAO）。正如上面代码所示，使用顶点缓冲区对象设置绘图操作可能需要多次调用`glBindBuffer`、`glVertexAttribPointer`和`glEnableVertexAttribArray`。为了更快地在顶点数组配置直接切换，OpenGL ES 3.0推出了顶点数组对象。VAO提供包含在顶点数组/顶点缓冲区对象配置之间切换所需要的所有状态的单一对象。

实际上，OpenGL ES 3.0中总是有一个活动的顶点数组对象。目前上述所有例子都在默认的顶点数组对象上操作（默认VAO的ID为0）。要创建新的顶点数组对象，可以使用`glGenVertexArrays`函数：

``` objc
/**
 创建顶点数组对象

 @param n#> 要返回的顶点数组对象名称的数量 description#>
 @param arrays#> 指向一个n个元素的数组的指针，该数组是分配的顶点数组对象返回的位置 description#>
 @return void
 */
glGenVertexArrays(GLsizei n, GLuint *arrays);
```

一旦创建，就可以用`glBindVertexArray`绑定顶点数组对象供以后使用：

``` objc
/**
 绑定顶点数组对象

 @param array#> 被指定为当前顶点数组对象的对象 description#>
 @return void
 */
glBindVertexArray(GLuint array);
```

每个VAO都包含一个完整的状态向量，描述所有顶点缓冲区绑定和启用的顶点客户状态。绑定VAO时，它的状态向量提供顶点缓冲区状态的当前设置。用`glBindVertexArray`绑定顶点数组对象后，更改顶点数组状态的后续调用（`glBindBuffer`、`glVertexAttribPointer`、`glEnableVertexAttribArray`、`glDisableVertexAttribArray`）将影响新的VAO。

这样，应用程序可以通过绑定一个已经设置状态的顶点数组对象快速地在顶点数组配置之间切换。所有变化可以在一个函数调用中完成，没有必要多次调用以更改顶点数组状态。例子：

``` objc
void DrawPrimitiveWithVAOs() {
    GLuint vboIds[2];
    GLint vertex_position_size = 3;
    GLint vertex_color_size = 4;
    GLfloat vertices[] = {
        0.0f, 0.5f, 0.0f,
        1.0f, 0.0f, 0.0f, 1.0f,
        -0.5f, -0.5f, 0.0f,
        0.0f, 1.0f, 0.0f, 1.0f,
        0.5f, -0.5f, 0.0f,
        0.0f, 0.0f, 1.0f, 1.0f
    };
    GLuint vaoId;
    GLushort indices[3] = { 0, 1, 2 };
    glGenBuffers(2, vboIds);
    glBindBuffer(GL_ARRAY_BUFFER, vboIds[0]);
    glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, vboIds[1]);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW);
    
    glGenVertexArrays(1, &vaoId);
    glBindVertexArray(vaoId);
    
    glBindBuffer(GL_ARRAY_BUFFER, vboIds[0]);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, vboIds[1]);
    
    glEnableVertexAttribArray(0);
    glEnableVertexAttribArray(1);
    
    glVertexAttribPointer(0, vertex_position _size, GL_FLOAT, GL_FALSE, sizeof(GLfloat) * (vertex_position_size + vertex_color_size), 0);
    glVertexAttribPointer(1, vertex_color_size, GL_FLOAT, GL_FALSE, sizeof(GLfloat) * (vertex_position_size + vertex_color_size), (const void *)(sizeof(GLfloat) * vertex_position_size));
        
    glBindVertexArray(0);

    // 绘制
    glBindVertexArray(vaoId);
    glDrawElements(GL_TRIANGLES, 3, GL_UNSIGNED_SHORT, 0);
}
```

当应用程序结束一个或者多个顶点数组对象的使用时，可以用`glDeleteVertexArrays`删除它们：

``` objc
/**
 删除顶点数组对象

 @param n#> 要删除的顶点数组对象的数量 description#>
 @param arrays#> 包含需要删除的顶点数组对象的有n个元素的数组 description#>
 @return void
 */
glDeleteVertexArrays(GLsizei n, const GLuint *arrays);
```

# 映射缓冲区对象

到目前为止，我们已经使用了`glBufferData`或`glBufferSubData`将数据加载到缓冲区对象。应用程序也可以将缓冲区对象数据存储映射到应用程序的地址空间（也可以解除映射）。应用程序映射缓冲区而不使用`glBufferData`或者`glBufferSubData`加载数据有几个理由：
- 映射缓冲区可以减少应用程序的内存占用，因为可能只需要存储数据的一个副本；
- 在使用共享内存的架构上，映射缓冲区返回GPU存储缓冲区的地址空间的直接指针。通过映射缓冲区，应用程序可以避免复制步骤，从而实现更好的更新性能。

`glMapBufferRange`命令返回指向所有或者一部分（范围）缓冲区对象数据存储的指针。这个指针可以供应应用程序使用，以读取或者更新缓冲区对象的内容。`glUnmapBuffer`命令用于指示更新已经完成和释放映射的指针：

``` objc
/**
 返回指向所有或者部分缓冲区对象数据存储的指针

 @param target#> GL_ARRAY_BUFFER, GL_ELEMENT_ARRAY_BUFFER, GL_COPY_READ_BUFFER, GL_COPY_WRITE_BUFFER, GL_PIXEL_PACK_BUFFER, GL_PIXEL_UNPACK_BUFFER, GL_TRANSFORM_FEEDBACK_BUFFER, GL_UNIFORM_BUFFER description#>
 @param offset#> 缓冲区数据存储中的偏移量，以字节数计算 description#>
 @param length#> 需要映射的缓冲区数据的字节数 description#>
 @param access#> 详见下表 description#>
 @return GLvoid *
 */
glMapBufferRange(GLenum target, GLintptr offset, GLsizeiptr length, GLbitfield access);
```

访问标志 | 描述
- | -
GL_MAP_READ_BIT | 应用程序将从返回的指针读取
GL_MAP_WRITE_BIT | 应用程序将写入返回的指针

此外，应用程序可以包含如下可选访问标志：

访问标志 | 描述
- | -
GL_MAP_INVALIDATAE_RANGE_BIT | 表示指定范围内的缓冲区内容可以在返回指针之前由驱动程序放弃。这个标志不能与GL_MAP_READ_BIT组合使用
GL_MAP_INVALIDATE_BUFFER_BIT | 表示整个缓冲区的内容可以在返回指针之前由驱动程序放弃。这个标志不能与GL_MAP_READ_BIT组合使用
GL_MAP_FLUSH_EXPLICIT_BIT | 表示应用程序将明确地用glFlushMappedBufferRange刷新对映射范围子范围的操作。这个标志不能与GL_MAP_WRITE_BIT组合使用
GL_MAP_UNSYNCHRONIZED_BIT | 表示驱动程序在返回缓冲区范围的指针之前不需要等待缓冲对象上的未决操作。如果有未决的操作，则未决操作的结果和缓冲区对象上的任何未来操作都变为未定义

`glMapBufferRange`返回请求的缓冲区数据存储范围的指针。如果出现错误或者发出无效的请求，该函数将返回NULL。`glUnmapBuffer`命令取消之前的缓冲区映射：

``` objc
/**
 取消缓冲区映射

 @param target#> 必须设置为GL_ARRAY_BUFFER description#>
 @return GLboolean
 */
glUnmapBuffer(GLenum target);
```

如果取消映射操作成功，则`glUnmapBuffer`返回`GL_TRUE`。`glMapBufferRange`返回的指针在成功执行取消映射之后不再可以使用。如果顶点缓冲区对象数据存储中的数据在缓冲区映射之后已经破坏，`glUnmapBuffer`将返回`GL_FALSE`，这可能是因为屏幕分辨率的变化、OpenGL ES上下文使用多个屏幕或者导致映射内存被抛弃的内存不足事件所导致。

写入映射缓冲区对象例子：
``` objc
GLfloat *vtxMappedBuf;
GLushort *idxMappedBuf;
GLfloat vertices[] = {
    0.0f, 0.5f, 0.0f,
    1.0f, 0.0f, 0.0f, 1.0f,
    -0.5f, -0.5f, 0.0f,
    0.0f, 1.0f, 0.0f, 1.0f,
    0.5f, -0.5f, 0.0f,
    0.0f, 0.0f, 1.0f, 1.0f
};
GLushort indices[3] = { 0, 1, 2 };

glGenBuffers(2, vboIds);
glBindBuffer(GL_ARRAY_BUFFER, vboIds[0]);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), NULL, GL_STATIC_DRAW);
vtxMappedBuf = (GLfloat *)glMapBufferRange(GL_ARRAY_BUFFER, 0, sizeof(vertices), GL_MAP_WRITE_BIT | GL_MAP_INVALIDATE_BUFFER_BIT);
if (vtxMappedBuf == NULL) {
    NSLog(@"Error mapping vertex buffer object");
    return;
}
memcpy(vtxMappedBuf, vertices, sizeof(vertices));
if (glUnmapBuffer(GL_ARRAY_BUFFER) == GL_FALSE) {
    NSLog(@"Error unmapping array buffer object");
    return;
}

glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, vboIds[1]);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), NULL, GL_STATIC_DRAW);
idxMappedBuf = (GLushort *)glMapBufferRange(GL_ELEMENT_ARRAY_BUFFER, 0, sizeof(indices), GL_MAP_WRITE_BIT | GL_MAP_INVALIDATE_BUFFER_BIT);
if (idxMappedBuf == NULL) {
    NSLog(@"Error mapping element buffer object");
    return;
}
memcpy(idxMappedBuf, indices, sizeof(indices));
if (glUnmapBuffer(GL_ELEMENT_ARRAY_BUFFER) == GL_FALSE) {
    NSLog(@"Error unmapping element buffer object");
    return;
}
```

## 刷新映射的缓冲区

应用程序可能希望用`glMapBufferRange`来映射缓冲区对象的一个范围或者全部，但是只更新映射范围的不同子区域。为了避免调用`glUnmapBuffer`时刷新整个映射范围的潜在性能损失，应用程序可以用`GL_MAP_FLUSH_EXPLICIT_BIT`访问标志（和`GL_MAP_WRITE_BIT`组合）映射。当应用程序完成映射范围一部分的更新时，可以用`glFlushMappedBufferRange`指出这个事实：

``` objc
/**
 指出应用程序完成映射范围的部分更新

 @param target#> GL_ARRAY_BUFFER, GL_ELEMENT_ARRAY_BUFFER, GL_COPY_READ_BUFFER, GL_COPY_WRITE_BUFFER, GL_PIXEL_PACK_BUFFER, GL_PIXEL_UNPACK_BUFFER, GL_TRANSFORM_FEEDBACK_BUFFER, GL_UNIFORM_BUFFER description#>
 @param offset#> 从映射缓冲区起始点的偏移量，以字节数表示 description#>
 @param length#> 从便宜点开始刷新的缓冲区字节数 description#>
 @return void
 */
glFlushMappedBufferRange(GLenum target, GLintptr offset, GLsizeiptr length);
```

如果应用程序用`GL_MAP_FLUSH_EXPLICIT_BIT`映射，但是没有明确用`glFlushMappedBufferRange`刷新修改后的区域，它的内容将是未定义的。

# 复制缓冲区对象

上面我们已经说明如何用`glBufferData`、`glBufferSubData`和`glMapBufferRange`加载缓冲区对象。所有这些技术都涉及从应用程序到设备的数据传输。OpenGL ES 3.0还可以从一个缓冲区对象将数据完全复制到设备：

``` objc
/**
 从一个缓冲区对象将数据完全复制到设备

 @param readTarget#> 读取的缓冲区对象目标 description#>
 @param writeTarget#> 写入的缓冲区对象目标 description#>
 @param readOffset#> 需要复制的读缓冲数据中的偏移量，以字节表示 description#>
 @param writeOffset#> 需要复制的写缓冲数据中的偏移量，以字节表示 description#>
 @param size#> 从读缓冲区数据复制到写缓冲区数据的字节数 description#>
 @return void
 */
glCopyBufferSubData(GLenum readTarget, GLenum writeTarget, GLintptr readOffset, GLintptr writeOffset, GLsizeiptr size);
```

调用`glCopyBufferSubData`将从绑定到`readtarget`的缓冲区复制指定的字节到`writetarget`。缓冲区绑定根据每个目标的最后一次`glBindBuffer`确定调用。任何类型的缓冲区对象（数组、元素数组、变换反馈等）都可以绑定到`GL_COPY_READ_BUFFER`或`GL_COPY_WRITE_BUFFER`目标。这两个目标是一种方便的措施，使得应用程序在执行缓冲区间的复制时不必改变任何真正的缓冲区绑定。

# 总结

这篇文章主要介绍了指定顶点属性和数据的方法。