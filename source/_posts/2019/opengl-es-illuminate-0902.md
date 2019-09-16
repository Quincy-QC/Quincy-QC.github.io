---
title: "OpenGL ES--光照"
catalog: true
toc_nav_num: true
date: 2019-09-02 17:30:54
subtitle: "About OpenGL ES"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS

---

# 颜色

现实世界中有无数种颜色，每一个物体都有它们自己的颜色。我们需要使用（有限的）数值来模拟真实世界中（无限）的颜色，所以并不是所有现实世界中的颜色都可以用数值来表示的。然而我们仍能通过数值来表现出非常多的颜色，甚至可能都不会注意到与现实的颜色有任何的差异。颜色可以数字化的由红色（Red）、绿色（Green）和蓝色（Blue）三个分量组成，它们通常被缩写为RGB。仅仅用这三个值就可以组合出任意一种颜色。
我们在显示生活中看到某一物体的颜色并不是这个物体真正拥有的颜色，而是它所反射的（Reflected）颜色。换句话说，那些不能被物体所吸收（Absorb）的颜色（被拒绝的颜色）就是我们能够感知到的物体的颜色。例如，太阳光能被看见的白光其实是由许多不同的颜色组合而成的。如果我们将白光照在一个蓝色的玩具上，这个蓝色的玩具会吸收白光中除了蓝色以外的所有子颜色，不被吸收的蓝色光被反射到我们的眼中，让这个玩具看起来是蓝色的。
这些颜色反射的定律被直接地运用在图形领域。当我们在OpenGL中创建一个光源时，我们希望给光源一个颜色。我们将光源设置为白色。当我们把光源的颜色与物体的颜色值相乘，所得到的就是这个物体所反射的颜色（也就是我们所感知的颜色）。在图形学中，我们将这两个颜色向量作分量相乘，结果就是最终的颜色向量：

``` objc
vec3 lightColor = vec3(1.0f, 1.0f, 1.0f);
vec3 toyColor = vec3(1.0f, 0.5f, 0.3f);
vec3 result = lightColor * toyColor;    // = (1.0f, 0.5f, 0.3f)
```

我们可以看到玩具的颜色吸收了白色光源中很大一部分的颜色，但它根据自身的颜色值对红、绿、蓝三个分量都做出了一定的反射。这也表现了现实中颜色的工作原理。由此，我们可以定义物体的颜色为物体从一个光源反射各个颜色分量的大小。如果我们使用绿色光源：

``` objc
vec3 lightColor = vec3(0.0f, 1.0f, 0.0f);
vec3 toyColor = vec3(1.0f, 0.5f, 0.3f);
vec3 result = lightColor * toyColor;    // = (0.0f, 0.5f, 0.0f)
```

可以看到，并没有红色和蓝色的光让玩具吸收或者反射。这个玩具吸收了光线中一半的绿色值，但仍然反射了一半的绿色值。玩具现在看上去是深绿色的。

# 创建一个光照场景

首先我们需要一个物体来作为被投光的对象--立方体箱子。还有一个立方体箱子来代表光源在3D场景中的位置。这里使用两套着色器，因为灯的颜色不应该受到物体光照计算结果的影响，所以需要将它与物体分离，不受其他颜色变化的影响。

着色器代码：

``` objc
NSString *objectVertexShaderString = @" \
#version 300 es \
layout(location = 0) in vec3 a_position; \
uniform mat4 modelMatrix; \
uniform mat4 viewMatrix; \
uniform mat4 projectionMatrix; \
void main() { \
gl_Position = projectionMatrix * viewMatrix * modelMatrix * vec4(a_position, 1.0); \
}";
NSString *objectFragmentShaderString = @"\
#version 300 es \
precision mediump float; \
uniform vec3 objectColor; \
out vec4 fragColor; \
void main() { \
fragColor = vec4(objectColor, 1.0); \
}";

NSString *lightVertexShaderString = @" \
#version 300 es \
layout(location = 0) in vec3 a_position; \
uniform mat4 modelMatrix; \
uniform mat4 viewMatrix; \
uniform mat4 projectionMatrix; \
void main() { \
gl_Position = projectionMatrix * viewMatrix * modelMatrix * vec4(a_position, 1.0); \
}";
NSString *lightFragmentShaderString = @"\
#version 300 es \
precision mediump float; \
out vec4 fragColor; \
void main() { \
fragColor = vec4(1.0); \
}";
```

绘制代码：

``` objc
- (void)setupVertices {
    ...
    顶点配置，设置缓冲
    ...
    // 物体顶点数组对象
    glGenVertexArrays(1, &vao);
    glBindVertexArray(vao);
    glEnableVertexAttribArray(0);
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, sizeof(GLfloat)*6, 0);
    glEnableVertexAttribArray(1);
    glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, sizeof(GLfloat)*6, (GLvoid *)(sizeof(GLfloat)*3));
    glBindVertexArray(0);

    // 灯顶点数组对象
    glGenVertexArrays(1, &lightvao);
    glBindVertexArray(lightvao);
    glEnableVertexAttribArray(0);
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, sizeof(GLfloat)*6, 0);
    glBindVertexArray(0);
}

- (void)glkView:(GLKView *)view drawInRect:(CGRect)rect {
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
    glClearColor(0.0f, 0.0f, 0.0f, 1.0f);
    
    glUseProgram(program1);
    
    glBindVertexArray(vao);
    glDrawArrays(GL_TRIANGLES, 0, 36);
    
    float aspect = self.view.frame.size.width / self.view.frame.size.height;
    GLKMatrix4 modelMatrix = GLKMatrix4MakeTranslation(-0.3f, 0.2f, 1.0f);
    GLKMatrix4 viewMatrix = GLKMatrix4MakeLookAt(-3.5f, 2.0f, 4.0f, 0.5f, 0.0f, 0.0f, 0.0f, 1.0f, 0.0f);
    GLKMatrix4 projectionMatrx = GLKMatrix4MakePerspective(GLKMathDegreesToRadians(45), aspect, 0.1, 100.0);
    glUniformMatrix4fv(modelLocation, 1, GL_FALSE, modelMatrix.m);
    glUniformMatrix4fv(viewLocation, 1, GL_FALSE, viewMatrix.m);
    glUniformMatrix4fv(projectionLocation, 1, GL_FALSE, projectionMatrx.m);
    glUniform3f(objectLocation, 1.0f, 0.5f, 0.3f);  // objct color
    
    glUseProgram(program2);
    glBindVertexArray(lightvao);
    glDrawArrays(GL_TRIANGLES, 0, 36);
    
    GLKMatrix4 translateMatrix2 = GLKMatrix4MakeTranslation(3.0f, 6.6f, 2.5f);
    GLKMatrix4 scaleMatrix2 = GLKMatrix4MakeScale(0.2f, 0.2f, 0.2f);
    GLKMatrix4 modelMatrix2 = GLKMatrix4Multiply(scaleMatrix2, translateMatrix2);
    glUniformMatrix4fv(modelLocation2, 1, GL_FALSE, modelMatrix2.m);
    glUniformMatrix4fv(viewLocation2, 1, GL_FALSE, viewMatrix.m);
    glUniformMatrix4fv(projectionLocation2, 1, GL_FALSE, projectionMatrx.m);
}
```

这样我们就能有一个干净的光照实验场地，如下图：

![干净的光照实验场地](/img/article/20190902/1.png)

# 基础光照

现实世界的光照是极其复杂的，而且会受到诸多因素的影响，这是我们有限的计算能力所无法比拟的。因此OpenGL的光照使用的是简化的模型，对现实的情况进行近似，这样处理起来会更容易一些，而且看起来也差不多一样。这些光照模型都是基于我们对光的物理特性的理解。其中一个模型被称为冯氏光照模型（Phong Lighting Model）。冯氏光照模型的主要结构由3个分量组成：环境（Ambient）、漫反射（Diffuse）和镜面（Specular）光照。

![基础光照](/img/article/20190902/2.png)

- 环境光照（Ambient Lighting）：即使在黑暗的情况下，世界上通常也仍然有一些光亮，所以物体几乎永远不会是完全黑暗的。为了模拟这个，我们会使用一个环境光照常量，它会永远给物体一些颜色。
- 漫反射光照（Diffuse Lighting）：模拟光源对物体的方向性影响（Directional Impact）。它是冯氏光照模型中视觉上最显著的分量。物体的某一部分越是正对着光源，他就会越亮。
- 镜面光照（Specular Lighting）：模拟有光泽物体上面出现的亮点。镜面光照的颜色相比于物体的颜色会更倾向于光的颜色。

# 环境光照

光通常都不是来自于同一个光源，而是来自于我们周围分散的很多光源，即使它们可能并不是那么显而易见。光的一个属性是，它可以向很多方向发散并反弹，从而能够到达不是非常直接临近的点。所以，光能够在其它的表面上反射，对一个物体产生间接的影响。考虑到这种情况的算法叫做全局照明（Global Illumination）算法，但是这种算法既开销高昂又及其复杂。

由于我们现在对那种又复杂又开销高昂的算法不是很感兴趣，所以我们将会先使用一个简化的全局照明模型，即环境光照。我们使用一个很小的常量（光照）颜色，添加到物体片段的最终颜色中，这样子的话即使场景中没有直接的光源也能看起来存在有一些发散的光。

把环境光照添加到场景里非常简单。我们用光的颜色乘以一个很小的常量环境因子，再乘以物体的颜色，然后将最终结果作为片段的颜色：

``` objc
NSString *objectFragmentShaderString = @"\
#version 300 es \
precision mediump float; \
uniform vec3 objectColor; \
uniform vec3 lightColor; \
out vec4 fragColor; \
void main() { \
float ambientStrength = 0.1; \
vec3 ambient = ambientStrength * lightColor; \
vec3 result = ambient * objectColor; \
fragColor = vec4(result, 1.0); \
}";
```

效果如下图，冯氏光照的第一个阶段已经应用到我们的物体上。这个物体非常暗，但由于应用了环境光照（注意光源立方体没受影响是因为我们对它使用了另一个着色器），也不完全是黑的。

![环境光照](/img/article/20190902/3.png)

# 漫反射光照

环境光照本身不能提供最有趣的结果，但是漫反射光照就能开始对物体产生显著的视觉影响了。漫反射光照物体上与光线方向越接近的片段能从光源处获得更多的亮度。

![漫反射光照原理图](/img/article/20190902/4.png)

图左上方有一个光源，它所发出的光线落在物体的一个片段上。我们需要测量这个光线是以什么角度接触到这个片段的。如果光线垂直于物体表面，这束光对物体的影响会最大化（更亮）。为了测量光线和片段的角度，我们使用一个叫做法向量的东西，它是垂直于片段表面的一个向量（黄色箭头），这两个向量之间的角度很容易就能够通过点乘计算出来。

两个单位向量的夹角越小，它们点乘的结果越倾向于1。当两个向量的夹角为90度的时候，点乘会变为0。这同样适用于θ，θ越大，光对片段颜色的影响就越小。

> 为了得到两个向量夹角的余弦值，我们使用的是单位向量（长度为1的向量），所以我们需要确保所有的向量都是标准化的，否则点乘返回的就不仅仅是余弦值了。

点乘返回一个标量，我们可以用它计算光线对片段颜色的影响。不同片段朝向光源的方向的不同，这些片段被照亮的情况也不同。

所以，计算漫反射光照需要：
- 法向量：一个垂直于顶点表面的向量。
- 定向的光线：作为光源的位置与片段的位置之间向量差的方向向量。为了计算这个光线，我们需要光的位置向量和片段的位置向量。

# 法向量

法向量是一个垂直于顶点表面的（单位）向量。由于顶点本事并没有表面（它只是空间中一个独立的点），我们利用它周围的顶点来计算出这个顶点的表面。我们能够使用一个小技巧，使用叉乘对立方体所有的顶点计算法向量，但是由于3D立方体不是一个复杂的形状，所以我们可以简单地把发现数据手工添加到顶点数据中。如下：

``` objc
glFloat vertices[] = {
    -0.5f, -0.5f, -0.5f,  0.0f,  0.0f, -1.0f,
    0.5f, -0.5f, -0.5f,  0.0f,  0.0f, -1.0f,
    0.5f,  0.5f, -0.5f,  0.0f,  0.0f, -1.0f,
    0.5f,  0.5f, -0.5f,  0.0f,  0.0f, -1.0f,
    -0.5f,  0.5f, -0.5f,  0.0f,  0.0f, -1.0f,
    -0.5f, -0.5f, -0.5f,  0.0f,  0.0f, -1.0f,
    
    -0.5f, -0.5f,  0.5f,  0.0f,  0.0f, 1.0f,
    0.5f, -0.5f,  0.5f,  0.0f,  0.0f, 1.0f,
    0.5f,  0.5f,  0.5f,  0.0f,  0.0f, 1.0f,
    0.5f,  0.5f,  0.5f,  0.0f,  0.0f, 1.0f,
    -0.5f,  0.5f,  0.5f,  0.0f,  0.0f, 1.0f,
    -0.5f, -0.5f,  0.5f,  0.0f,  0.0f, 1.0f,
    
    -0.5f,  0.5f,  0.5f, -1.0f,  0.0f,  0.0f,
    -0.5f,  0.5f, -0.5f, -1.0f,  0.0f,  0.0f,
    -0.5f, -0.5f, -0.5f, -1.0f,  0.0f,  0.0f,
    -0.5f, -0.5f, -0.5f, -1.0f,  0.0f,  0.0f,
    -0.5f, -0.5f,  0.5f, -1.0f,  0.0f,  0.0f,
    -0.5f,  0.5f,  0.5f, -1.0f,  0.0f,  0.0f,
    
    0.5f,  0.5f,  0.5f,  1.0f,  0.0f,  0.0f,
    0.5f,  0.5f, -0.5f,  1.0f,  0.0f,  0.0f,
    0.5f, -0.5f, -0.5f,  1.0f,  0.0f,  0.0f,
    0.5f, -0.5f, -0.5f,  1.0f,  0.0f,  0.0f,
    0.5f, -0.5f,  0.5f,  1.0f,  0.0f,  0.0f,
    0.5f,  0.5f,  0.5f,  1.0f,  0.0f,  0.0f,
    
    -0.5f, -0.5f, -0.5f,  0.0f, -1.0f,  0.0f,
    0.5f, -0.5f, -0.5f,  0.0f, -1.0f,  0.0f,
    0.5f, -0.5f,  0.5f,  0.0f, -1.0f,  0.0f,
    0.5f, -0.5f,  0.5f,  0.0f, -1.0f,  0.0f,
    -0.5f, -0.5f,  0.5f,  0.0f, -1.0f,  0.0f,
    -0.5f, -0.5f, -0.5f,  0.0f, -1.0f,  0.0f,
    
    -0.5f,  0.5f, -0.5f,  0.0f,  1.0f,  0.0f,
    0.5f,  0.5f, -0.5f,  0.0f,  1.0f,  0.0f,
    0.5f,  0.5f,  0.5f,  0.0f,  1.0f,  0.0f,
    0.5f,  0.5f,  0.5f,  0.0f,  1.0f,  0.0f,
    -0.5f,  0.5f,  0.5f,  0.0f,  1.0f,  0.0f,
    -0.5f,  0.5f, -0.5f,  0.0f,  1.0f,  0.0f
};
```

更新后的光照顶点着色器：

``` objc
NSString *vertexShaderString = @" \
#version 300 es \
layout(location = 0) in vec3 a_position; \
layout(location = 1) in vec3 a_normal; \
uniform mat4 modelMatrix; \
uniform mat4 viewMatrix; \
uniform mat4 projectionMatrix; \
out vec3 v_normal; \
void main() { \
v_normal = a_normal; \
gl_Position = projectionMatrix * viewMatrix * modelMatrix * vec4(a_position, 1.0); \
}";
```

# 计算漫反射光照

我们现在对每个顶点都有了法向量，但是我们仍需要光源的位置向量和片段的位置向量。由于光源的位置是一个静态变量，我们可以简单地在片段着色器中把它声明为`uniform`：

``` objc
uniform vec3 lightPos;
```

最后，我们还需要片段的位置。我们会在世界空间进行所有的光照计算，因此我们需要一个在世界空间中的顶点位置。我们可以通过把顶点位置属性乘以模型矩阵（不是观察和投影矩阵）来把它变换到世界空间坐标。这个在顶点着色器中很容易完成，所以我们声明一个输出变量，来计算它的世界空间坐标：

``` objc
NSString *vertexShaderString = @" \
#version 300 es \
layout(location = 0) in vec3 a_position; \
layout(location = 1) in vec3 a_normal; \
uniform mat4 modelMatrix; \
uniform mat4 viewMatrix; \
uniform mat4 projectionMatrix; \
out vec3 v_normal; \
out vec3 FragPos; \
void main() { \
FragPos = vec3(modelMatrix * vec4(a_position, 1.0)); \
v_normal = a_normal; \
gl_Position = projectionMatrix * viewMatrix * modelMatrix * vec4(a_position, 1.0); \
}";
```

最后，在片段着色器中添加对应的输入变量`FragPos`。

我们需要做的第一件事是计算光源和片段位置之间的方向向量。光的方向向量是光源位置向量与片段位置向量之间的向量差。我们同样希望确保所有相关向量最后都转换为单位向量，所以需要把法线和最终的方向向量都进行标准化：

``` objc
vec3 norm = normalize(v_normal);
vec3 lightDir = normalize(lightPos - FragPos);
```

> 当计算光照时我们通常不关心一个向量的模长或它的位置，只关心它们的方向。所以，几乎所有的计算都使用单位向量完成，因为这简化了大部分的计算（比如点乘）。所以，几乎所有的计算都使用单位向量完成，因为这简化了大部分的计算（比如点乘）。所以当进行光照计算时，确保总是对相关向量进行标准化，来保证它们是真正地单位向量。忘记对向量进行标准化是一个非常常见的错误。

下一步，我们对`norm`和`lightDir`向量进行点乘，计算光源对当前片段实际的漫反射影响。结果值再乘以光的颜色，得到漫反射分量。两个向量之间的角度越大，漫反射分量就会越小：

``` objc
float diff = max(dot(norm, lightDir), 0.0);
vec3 diffuse = diff * lightColor;
```

如果两个向量之间的角度大于90度，点乘的结果就会变成负数，这样会导致漫反射分量变为负数。为此，我们使用`max`函数返回两个参数之间较大的参数，从而保证漫反射分量不会变成负数。负数颜色的光照是没有定义的，所以最好避免它。

现在我们有了环境光分量和漫反射分量，我们把它们相加，然后把结果乘以物体的颜色，来获得片段最后的输出颜色：

``` objc
vec3 result = (ambient + diffuse) * objectColor;
FragColor = vec4(result, 1.0);
```

最终的片段着色器代码如下：

``` objc
NSString *fragmentShaderString = @"\
#version 300 es \
precision mediump float; \
uniform vec3 objectColor; \
uniform vec3 lightColor; \
uniform vec3 lightPos; \
in vec3 FragPos; \
in vec3 v_normal; \
out vec4 fragColor; \
void main() { \
float ambientStrength = 0.1; \
vec3 ambient = ambientStrength * lightColor; \
vec3 norm = normalize(v_normal); \
vec3 lightDir = normalize(lightPos - FragPos); \
float diff = max(dot(norm, lightDir), 0.0); \
vec3 diffuse = diff * lightColor; \
vec3 result = (ambient + diffuse) * objectColor; \
fragColor = vec4(result, 1.0); \
}";
```

编译之后，效果图如下：

![漫反射光照](/img/article/20190902/5.png)

# 还有...

现在我们已经把法向量从顶点着色器传到了片段着色器。可以，目前片段着色器里的计算都是在世界空间坐标系中进行的。所以，我们应该也需要把法向量转换为世界空间坐标，但这不是简单地把它乘以一个模型矩阵就能搞定的。

首先，法向量只是一个方向向量，不能表达空间中的特定位置。同时，法向量没有齐次坐标（顶点位置中的w分量）。这意味着，位移不应该影响到法向量。一次，如果我们打算把法向量乘以一个模型矩阵，我们就要从矩阵中移除位移部分，只选用模型矩阵左上角3x3的矩阵（注意，我们也可以把法向量的w分量设置为0，再乘以4x4矩阵；这同样可以移除位移）。对于法向量，我们只希望对它实施缩放和旋转变换。

其次，如果模型矩阵执行了不等比缩放，顶点的改变会导致法向量不再垂直于表面了。因此，我们不能用这样的模型矩阵来变换法向量。如下图应用了不等比缩放的模型矩阵对法向量的影响：

![不等比缩放的模型矩阵对法向量的影响](/img/article/20190902/6.png)

每当我们应用一个不等比缩放时（注意，等比缩放不会破坏法线，因为法线的方向没被改变，仅仅改变了法线的长度，而这很容易通过标准化来修复），法向量就不会再垂直于对应的表面了，这样光照就会被破坏。

修复这个行为的诀窍是使用一个为法向量专门定制的模型矩阵。这个矩阵称之为法线矩阵，它使用了一些线性代数的操作来移除对法向量错误缩放的影响。

法线矩阵被定义为「模型矩阵左上角的逆矩阵的转置矩阵」。注意，大部分的资源都会将法线矩阵定义为应用到模型-观察矩阵上的操作，但是由于我们只在世界空间中进行操作（不是在观察空间），我们只使用模型矩阵。

在顶点着色器中，我们可以使用`inverse`和`transpose`函数自己生成这个法线矩阵，这两个函数对所有类型矩阵都有效。注意，我们还要把被处理过的矩阵强制转换为3x3矩阵，来保证它失去了位移属性以及能够乘以`vec3`的法向量：

``` objc
v_normal = mat3(transpose(inverse(model)))* a_normal;
```

在漫反射光照部分，光照表现并没有问题，这是因为我们没有对物体本身执行任何缩放操作，所以并不是必须要使用一个发现矩阵，仅仅让模型矩阵乘以法线也可以。可是，如果进行了不等比缩放，使用法线矩阵乘以法向量就是必不可少的了。

> 即使是对于着色器来说，逆矩阵也是一个开销比较大的运算，因此，只要可能就应该避免在着色器中进行逆矩阵运算，它们必须为场景中的每个顶点都进行这样的处理。用作学习目的这样做是可以的，但是对于一个对效率有要求的应用来说，在绘制之前最好用CPU计算出法线矩阵，然后通过uniform把值传递给着色器（像模型矩阵一样）。

# 镜面光照

现在我们还需要把镜面高光加进来，这样冯氏光照才能算完整。

和漫反射光照一样，镜面光照也是根据光的方向向量和物体的法向量来决定的，但是它也依赖于观察方向。镜面光照是基于光的反射特性。如果我们想象物体表面像一面镜子一样，那么无论我们从哪里去看那个表面所反射的光，镜面光照都会达到最大化：

![镜面光照原理图](/img/article/20190902/7.png)

我们通过反射法向量周围光的方向来计算反射向量。然后我们计算反射向量和视线方向的角度差，如果夹角越小，那么镜面光的影响就会越大。它的作用效果就是，当我们去看光被物体所反射的那个方向的时候，我们会看到一个高光。

观察向量是镜面光照附加的一个变量，我们可以使用观察者世界空间位置和片段的位置来计算它。之后，我们计算镜面光强度，用它乘以光源的颜色，再将它加上环境光和漫反射分量。

为了得到观察者的世界空间坐标，我们简单地使用摄像机对象的位置坐标代替（它当然就是观察者）。所以我们把另一个`uniform`添加到片段着色器，把相应的摄像机位置坐标传给片段着色器：

``` objc
uniform vec3 viewPos;
```

现在我们已经获得所有需要的变量，可以计算高光强度了。首先，我们定义一个镜面强度变量，给镜面高光一个中等亮度颜色，让它不要产生过度的影响：

``` objc
float specularStrength = 0.5;
```

如果我们把它设置为1.0f，我们会得到一个非常亮的镜面光分量。下一步，我们计算视线方向向量，和对应的沿着法线轴的反射向量：

``` objc
vec3 viewDir = normalize(viewPos - FragPos);
vec3 reflectDir = reflect(-lightDir, norm);
```

需要注意的是我们对`lightDir`向量进行了取反。`reflect`函数要求第一个向量是从光源指向片段位置的向量，但是`lightDir`刚好与其相反，是从片段指向光源。为了保证我们得到正确的`reflect`向量，我们通过对`lightDir`向量取反来获得相反的方向。第二个参数要求是一个法向量，所以我们提供的是已经标准化的`norm`向量。

剩下要做的是计算镜面分量：

``` objc
float spec = pow(max(dot(viewDir, reflectDir), 0.0), 32.0);
vec3 specular = specularStrength * spec * lightColor;
```

我们先计算视线方向与反射方向的点乘（并确保它不是负值），然后取它的32次幂。这个32是高光的反光度。一个物体的反光度越高，反射光的能力越强，散射得越少，高光点就会越小。

![不同反光度的视觉影响](/img/article/20190902/8.png)

剩下的最后一件事是把它加到环境光分量和漫反射分量里，再用结果乘以物体的颜色：

``` objc
vec3 result = (ambient + diffuse + specular) * objectColor;
FragColor = vec4(result, 1.0);
```

最终的片段着色器代码如下：

``` objc
NSString *fragmentShaderString = @"\
#version 300 es \
precision mediump float; \
uniform vec3 objectColor; \
uniform vec3 lightColor; \
uniform vec3 lightPos; \
uniform vec3 viewPos; \
in vec3 FragPos; \
in vec3 v_normal; \
out vec4 fragColor; \
void main() { \
float ambientStrength = 0.1; \
vec3 ambient = ambientStrength * lightColor; \
vec3 norm = normalize(v_normal); \
vec3 lightDir = normalize(lightPos - FragPos); \
float diff = max(dot(norm, lightDir), 0.0); \
vec3 diffuse = diff * lightColor; \
float specularStrength = 0.5; \
vec3 viewDir = normalize(viewPos - FragPos); \
vec3 reflectDir = reflect(-lightDir, norm); \
float spec = pow(max(dot(viewDir, reflectDir), 0.0), 32.0); \
vec3 specular = specularStrength * spec * lightColor; \
vec3 result = (ambient + diffuse + specular) * objectColor; \
fragColor = vec4(result, 1.0); \
}";
```

最终的效果图如下：

![镜面光照](/img/article/20190902/9.png)

> 这里我没做出效果，可能是没有找到对应的观察点。。。

# 总结

这篇文章主要针对颜色、光照进行剖析和示例展示。