---
title: "OpenGL ES学习--着色器与程序"
catalog: true
toc_nav_num: true
date: 2019-07-06 17:30:55
subtitle: "About OpenGL ES"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS

---

> 上一篇文章介绍了绘制一个三角形的简单程序，在例子中，我们创建了两个着色器对象（顶点着色器和片段着色器）和一个程序对象，以渲染三角形。

# 着色器和程序

着色器对象是包含单个着色器对象，源代码提供给着色器对象，然后着色器被编译为一个目标形式。编译后，着色器对象可以连接到一个程序对象。程序对象可以连接多个着色器对象。在OpenGL ES中，每个程序对象都必须连接一个顶点着色器和一个片段着色器。程序对象最终被链接为用于渲染的最后”可执行程序“。
获得链接后的着色器对象的一般过程包括6个步骤：
1. 创建一个顶点着色器对象和一个片段着色器对象；
2. 将源代码连接到每个着色器对象；
3. 编译着色器对象；
4. 创建一个程序对象；
5. 将编译后的着色器对象连接到程序对象；
6. 链接程序对象。

## 创建和编译一个着色器

着色器的创建与删除：
``` objc
/**
 创建着色器

 @param type#> 类型可以是顶点着色器 GL_VERTEX_SHADER 或者是片段着色器 GL_FRAGMENT_SHADER description#>
 @return 着色器对象
 */
glCreateShader(GLenum type);

/**
 删除着色器

 @param shader#> 着色器对象 description#>
 @return void
 */
glDeleteShader(GLuint shader)
```
注意，如果一个着色器连接到程序对象，那么调用删除方法并不会立刻删除着色器，而是将着色器标记，在着色器不再连接到任何程序对象时，它的内存对象被释放。

创建着色器对象后，下一步是提供着色器源代码：
``` objc
/**
 提供着色器源代码

 @param shader#> 着色器对象 description#>
 @param count#> 着色器源字符串的数量 description#>
 @param string#> 指向保存数量为count的着色器源字符串的数组指针 description#>
 @param length#> 指向字符串长度 description#>
 @return void
 */
glShaderSource(GLuint shader, GLsizei count, const GLchar *const *string, const GLint *length)
```

指定着色器代码之后，下一步是编译着色器：
``` objc
/**
 着色器编译

 @param shader#> 着色器对象 description#>
 @return void
 */
glCompileShader(GLuint shader)
```

编译之后可以通过`glGetShderiv`查询有关着色器对象的信息：
``` objc
/**
 获取编译器相关信息

 @param shader#> 着色器对象 description#>
 @param pname#> 查询参数，可以是：GL_COMPILE_STATUS，GL_DELETE_STATUS，GL_INFO_LOG_LENGTH，GL_SHADER_SOURCE_LENGTH，GL_SHADER_TYPE description#>
 @param params#> 查询结果存储位置的指针 description#>
 @return void
 */
glGetShaderiv(GLuint shader, GLenum pname, GLint *params)
```

通过`glGetShaderiv`传入`GL_INFO_LOG_LENGTH`参数获取日志缓冲区大小，然后用`glGetShaderInfoLog`检索信息日志：
``` objc
/**
 检索信息日志

 @param shader#> 着色器对象 description#>
 @param bufsize#> 保存信息日志的缓冲大小 description#>
 @param length#> 写入信息日志的长度，可以为NULL description#>
 @param infolog#> 指向保存信息日志的字符缓冲区指针 description#>
 @return void
 */
glGetShaderInfoLog(GLuint shader, GLsizei bufsize, GLsizei *length, GLchar *infolog)
```

通过上述步骤就可以加载一个着色器。

## 创建和链接程序

下一步就是创建一个程序对象。程序对象是一个容器对象，可以将着色器与之连接，并链接一个最终可执行程序。
创建一个程序对象：
``` objc
/**
 创建程序对象

 @param void void
 @return 程序对象
 */
glCreateProgram()
```

删除一个程序对象：
``` objc
/**
 删除一个程序对象

 @param program#> 程序对象 description#>
 @return void
 */
glDeleteProgram(GLuint program)
```

创建程序对象之后，下一步就是将着色器与之连接：
``` objc
/**
 连接着色器和程序

 @param program#> 程序对象 description#>
 @param shader#> 着色器对象 description#>
 @return void
 */
glAttachShader(GLuint program, GLuint shader)
```
> 着色器可以在任何时候连接--连接到程序之前不一定需要编译，甚至可以没有源代码。唯一的要求是，每个程序对象必须有且仅有一个顶点着色器和片段着色器与之连接。

断开程序和着色器：
``` objc
/**
 断开程序和着色器

 @param program#> 程序对象 description#>
 @param shader#> 着色器对象 description#>
 @return void
 */
glDetachShader(GLuint program, GLuint shader)
```

最后链接程序对象：
``` objc
/**
 链接程序

 @param program#> 程序对象 description#>
 @return void
 */
glLinkProgram(GLuint program)
```

这个链接操作负责生产最终的可执行程序。链接程序将确保顶点着色器写入片段着色器使用的所有顶点着色器输出变量（并用相同类型声明），链接程序还将确保任何在顶点和片段着色器中声明的统一变量和统一变量缓冲区的类型相符，此外，链接程序将确保最终的程序符合棘突实现的限制（例如，属性、统一变量或者输入输出着色器变量的数量）。一般来说，链接阶段是生成在硬件上运行的最终硬件指令的时候。

链接程序之后，检查链接状态：
``` objc
/**
 检查链接程序状态

 @param program#> 程序对象 description#>
 @param pname#> 获取信息的参数，可以是：GL_ACTIVE_(ATTRIBUTES, ATTRIBUTE_MAX_LENGTH, UNIFORM_BLOCK, UNIFORM_BLOCK_MAX_LENGTH, UNIFORMS, UNFORM_MAX_LENGTH), GL_ATTACHED_SHADERS, GL_DELETE_STATUS, GL_INFO_LOG_LENGTH, GL_LINK_STATUS, GL_PROGRAM_BINARY_RETRIEVABLE_HINT, GL_TRANSFORM_FEEDBACK_(BUFFER_MODE, VARYINGS, VARYING_MAX_LENGTH), GL_VALIDATE_STATUS description#>
 @param params#> 指向查询结果整数存储位置的指针 description#>
 @return void
 */
glGetProgramiv(GLuint program, GLenum pname, GLint *params)
```

同样可以通过`glGetProgramiv`函数传入`GL_INFO_LOG_LENGTH`参数获取信息日志缓冲区大小，查询链接程序日志：
``` objc
/**
 检索链接程序日志

 @param program#> 程序对象 description#>
 @param bufsize#> 存储信息日志的缓冲区大小 description#>
 @param length#> 写入信息日志长度，可以为NULL description#>
 @param infolog#> 指向存储信息日志的字符缓冲区指针 description#>
 @return void
 */
glGetProgramInfoLog(GLuint program, GLsizei bufsize, GLsizei *length, GLchar *infolog)
```

一旦成功链接程序，我们就几乎为使用它渲染做好了准备，但是，我们还需要检查程序是否有效。也就是说，成功链接不能保证执行的某些方面，如应用程序可能没有把有效的纹理绑定到采样器，这时候我们需要再次验证程序当前状态：
``` objc
/**
 检查程序能以当前状态执行

 @param program#> 程序对象 description#>
 @return void
 */
glValidateProgram(GLuint program)
```
调用`glValidProgram`函数后，通过之前的`glGetProgramiv`传入`GL_VALIDATE_STATUS`参数检查，信息日志也将更新。

在完成创建程序对象，连接着色器，链接以及获取信息日志确认无误，在渲染之前，我们还要对程序对象做一件事，就是将其设置为活动程序：
``` objc
/**
 设置为活动程序

 @param program#> 程序对象 description#>
 @return void
 */
glUseProgram(GLuint program)
```

# 统一变量和属性

一旦链接了程序对象，就可以在对象上进行许多查询。首先，我们可能需要找出程序中的活动统一变量（uniform）。

统一变量被组合成两类统一变量块。第一类是命名统一变量块，统一变量的值由所谓的统一变量缓冲区对象支持：
``` objc
uniform TransformBlock {
    mat4 matViewProj;
    mat3 matNormal;
    mat3 matTexGen
}
```

第二类是默认的统一变量块，用于在命名统一变量块之外的声明的统一变量：
```
uniform mat4 matViewProj;
uniform mat3 matNormal;
uniform mat3 matTexGen;
```

如果统一变量在顶点着色器和片段着色器均有声明，则声明的类型必须相同，且在两个着色器中的值也需相同。在链接阶段，链接程序将为程序中与默认统一变量块相关的活动统一变量指定位置。这些位置是应用程序用于加载统一变量的标识符。链接程序还将为与命名统一变量块相关的活动统一变量分配偏移和跨距（对于数组和矩阵类型的统一变量）。

## 获取和设置统一变量

要查询程序中活动统一变量的列表，首先通过`glGetProgramiv`函数传入`GL_ACTIVE_UNIFORMS`获取程序中活动统一变量的数量，传入`GL_ACTIVE_UNIFORM_MAX_LENGTH`参数获取统一变量最长字符数量，知道这两个参数之后可以使用`glGetActiveUniform`和`glGetActiveUniformsiv`找出每个统一变量的细节：
``` objc
GLint uniformsCount;
GLint maxUniformLength;
glGetProgramiv(program, GL_ACTIVE_UNIFORMS, &uniformsCount);
glGetProgramiv(program, GL_ACTIVE_UNIFORM_MAX_LENGTH, &maxUniformLength);
for (int i = 0; i < uniformsCount; i++) {
    GLsizei length;
    GLint size;
    GLenum type;
    GLchar *name = malloc(sizeof(GLchar) * maxUniformLength);
    glGetActiveUniform(program, i, maxUniformLength, &length, &size, &type, name);
    NSLog(@"length: %d, size: %d, type: %u, name: %s", length, size, type, name);
}
```

使用`glGetActiveUniform`，可以确定几乎所有统一变量的属性。通过统一变量的名称，我们就可以找到它的位置：
``` objc
/**
 获取统一变量在程序中的位置

 @param program#> 程序对象 description#>
 @param name#> 统一变量名称 description#>
 @return 位置
 */
glGetUniformLocation(GLuint program, const GLchar *name)
```

获取到位置下标后，我们就可以加载统一变量的值，各个类型都有对应不同的函数，举几个例子：
```
void glUniformlf(GLint location, GLfloat x);
void glUniformfv(GLint location, GLsizei count, const GLfloat* value);
...
```

>  `glUniform*`调用不以程序对象句柄作为参数，原因是，`glUniform*`总是在于`glUseProgram`绑定的当前程序上操作。统一变量值本身保存在程序对象中。也就是说，一旦在程序对象中设置一个统一变量的值，即使我们让另一个程序处于活动状态，该值仍然保留在原来的程序对象中。从这个意义上，我们可以说统一变量值是程序对象局部所有。

## 统一变量缓冲区对象

可以使用缓冲区对象存储变量数据，从而在程序中的着色器之间甚至程序之间共享统一变量，这种缓冲区被称为统一变量缓冲区对象 UBO（Uniform Buffer Object）。使用统一变量缓冲区对象可以在更新大的统一变量块时降低API开销，此外，这种方法增加了统一变量的可用存储。
要更新统一变量缓冲区对象的统一变量数据，我们可以使用`glBufferData`、`glBufferSubData`、`glMapBufferRange`、`glUnmapBuffer`等命令修改缓冲区对象的内容，而不是使用`glUniform*`命令。

UBO必须配合Uniform Block（命名统一变量块）一起使用，在显存中创建缓存对象（Buffer），在buffer中存储统一变量数据，将buffer与指定的point（不得大于`GL_MAX_UNIFORM_BUFFER_BINDINGS`，通过`glGet`函数查询）绑定，将统一变量缓冲区的索引和point绑定，这样通过point将变量和缓存链接。

[UBO原理图](/img/article/20190706/1.png)

首先检索统一变量块索引：
``` objc
/**
 检索统一变量块索引

 @param program#> 程序对象 description#>
 @param uniformBlockName#> 统一变量块名称 description#>
 @return 索引
 */
glGetUniformBlockIndex(GLuint program, const GLchar *uniformBlockName)
```

可以通过`glGetActiveUniformBlockName`获取统一变量块名：
``` objc
/**
 获取统一变量块名

 @param program#> 程序对象 description#>
 @param uniformBlockIndex#> 统一变量块索引 description#>
 @param bufSize#> 名称数组中的字符数 description#>
 @param length#> 可为NULL，否则可以写入字符数 description#>
 @param uniformBlockName#> 写入统一变量块名称 description#>
 @return 块名
 */
glGetActiveUniformBlockName(GLuint program, GLuint uniformBlockIndex, GLsizei bufSize, GLsizei *length, GLchar *uniformBlockName)
```

通过`glGetActiveUniformBlockiv`获取统一变量块的许多属性：
``` objc
/**
 获取统一变量块的属性

 @param program#> 程序对象 description#>
 @param uniformBlockIndex#> 需要查询的统一变量块索引 description#>
 @param pname#> 查询属性，可以是：GL_UNIFORM_BLOCK_(BINDING, DATA_SIZE, NAME_LENGTH, ACTIVE_UNIFORMS, ACTIVE_UNIFORM_INDICES, REFERENCED_BY_VERTEX_SHADER, REFERENCED_BY_FRAGMENT_SHADER) description#>
 @param params#> 写入pname指定的结果 description#>
 @return void
 */
glGetActiveUniformBlockiv(GLuint program, GLuint uniformBlockIndex, GLenum pname, GLint *params)
```

一旦有了统一变量块索引，就可以将该索引与程序中的统一变量块绑定点关联：
``` objc
/**
 将统一变量块索引与程序中的统一变量块绑定点关联

 @param program#> 程序对象 description#>
 @param uniformBlockIndex#> 统一变量块索引 description#>
 @param uniformBlockBinding#> 统一变量缓冲区对象绑定点 description#>
 @return void
 */
glUniformBlockBinding(GLuint program, GLuint uniformBlockIndex, GLuint uniformBlockBinding)
```

最后可以将统一变量缓冲区对象绑定到`GL_UNIFORM_BUFFER`目标和程序中的统一变量块绑定点：
``` objc
/**
 将统一变量缓冲区对象绑定到目标和程序中的统一变量块绑定点

 @param target#> 必须是GL_UNIFORM_BUFFER或GL_TRANSFORM_FEEDBACK_BUFFER description#>
 @param index#> 绑定索引 description#>
 @param buffer#> 缓冲区对象句柄 description#>
 @return void
 */
glBindBufferBase(GLenum target, GLuint index, GLuint buffer)

/**
 将统一变量缓冲区对象绑定到目标和程序中的统一变量块绑定点

 @param target#> 必须是GL_UNIFORM_BUFFER或GL_TRANSFORM_FEEDBACK_BUFFER description#>
 @param index#> 绑定索引 description#>
 @param buffer#> b缓冲区对象句柄 description#>
 @param offset#> 以字节数计算的缓冲区对象其实偏移 description#>
 @param size#> 可以从缓冲区对象读取或者写入缓冲对象的数据量 description#>
 @return void
 */
glBindBufferRange(GLenum target, GLuint index, GLuint buffer, GLintptr offset, GLsizeiptr size)
```

下面举个例子：
对于着色器代码，除非我们使用std140统一变量块布局（默认），否则需要查询程序对象得到字节偏移和跨距，以在统一变量缓冲区对象中设置统一变量数据。std140布局保证使用由OpengGL ES 3.0规范定义的明确布局规范进行特定包装。因此，使用std140，我们就可以在不同的OpengGL ES 3.0实现之间共享统一变量块。
着色器代码：
``` objc
layout (std140) uniform LightBlock {
    vec3 lightDirection;
    vec4 lightPosition;
}
```

设置UBO代码：
``` objc
GLuint blockId, bufferId;
GLint blockSize;
GLuint bindingPoint = 1;
GLfloat lightData[] = {
    1.0f, 0.0f, 0.0f, 0.0f,
    0.0f, 0.0f, 0.0f, 1.0f
};

blockId = glGetUniformBlockIndex(program, "LightBlock");

glUniformBlockBinding(program, blockId, bindingPoint);

glGetActiveUniformBlockiv(program, blockId, GL_UNIFORM_BLOCK_DATA_SIZE, &blockSize);

glGenBuffers(1, &bufferId);
glBindBuffer(GL_UNIFORM_BUFFER, bufferId);
glBufferData(GL_UNIFORM_BUFFER, blockSize, lightData, GL_DYNAMIC_DRAW);

glBindBufferBase(GL_UNIFORM_BUFFER, bindingPoint, bufferId);
```

修改UBO代码：
``` objc
GLfloat updateData[] = {
    1.0f, 0.0f, 0.0f, 0.0f
};

glBindBuffer(GL_UNIFORM_BUFFER, bufferId);

const GLchar *names[] = {"lightDirection"};
GLuint indices;
GLint offset;
GLint size;
glGetUniformIndices(program, 1, names, &indices);
glGetActiveUniformsiv(program, 1, &indices, GL_UNIFORM_OFFSET, &offset);
glGetActiveUniformsiv(program, 1, &indices, GL_UNIFORM_SIZE, &size);
glBufferSubData(GL_UNIFORM_BUFFER, offset, size, updateData);
```

# 程序二进制码

程序二进制码是完全编译和链接的程序的二进制表现形式。它们很有用，因为可以保存到文件系统供以后使用，从而避免在线编译的代价。我们也可以是使用程序二进制码，这样就没有必要在实现中分发着色器源代码。
我们可以在成功编译和链接程序之后，检索程序二进制码：
``` objc
/**
 检索程序二进制码
 
 @param program#> 程序对象 description#>
 @param bufSize#> 可以写入binary的最大字节数 description#>
 @param length#> 二进制数据字节数 description#>
 @param binaryFormat#> 供应商专用二进制格式标识 description#>
 @param binary#> 着色器编译器生成的二进制数据指针 description#>
 @return void
 */
glGetProgramBinary(GLuint program, GLsizei bufSize, GLsizei *length, GLenum *binaryFormat, GLvoid *binary)
```

检索之后，可以将其保存到文件系统，或者将程序二进制码读回OpenGL ES实现：
``` objc
/**
 将二进制码读回OpenGL ES实现

 @param program#> 程序对象 description#>
 @param binaryFormat#> 供应商专用二进制格式标识 description#>
 @param binary#> 着色器编译器生成的二进制数据指针 description#>
 @param length#> 二进制数据的字节数 description#>
 @return void
 */
glProgramBinary(GLuint program, GLenum binaryFormat, const GLvoid *binary, GLsizei length)
```

为了确保存储的程序二进制码仍然兼容，在调用`glProgramBinary`之后，可以通过`glProgramiv`查询`GL_LINK_STATUS`。如果二进制码不兼容，我们需要重新编译着色器源码。

# 总结

这篇文章主要介绍了创建、编译和链接着色器到程序的方法，着色器对象和程序对象组成了OpenGL ES 3.0中的基本对象。同时介绍了查询着色器和程序信息以及加载统一变量的方法等。