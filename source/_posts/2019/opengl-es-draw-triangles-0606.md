---
title: "使用OpenGL ES绘制三角形"
catalog: true
toc_nav_num: true
date: 2019-06-06 19:23:02
subtitle: "About OpenGL ES"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS

---

# 前言

OpenGL ES的学习，本来是在苹果官方文档上学习，但是该文档是针对基于有一定OpenGL ES基础的人员在iOS和Mac OS平台上的开发时的配置介绍和性能提升，并没有OpenGL ES基础的教学，可以用来后期在有一定知识情况下开发时使用手册。所以，这里从OpenGL ES的最基础开始说起，使用OpenGL ES绘制三角形。

# 自定义着色器绘制三角形

首先我们通过最原始的代码使用OpenGL ES创建一个三角形，这里需要我们自己编写着色器，着色器包含顶点着色器和片段着色器，编写着色器需要用它特有的语言--GLSL，关于GLSL的语法，参见[着色器语言](https://github.com/wshxbqq/GLSL-Card)。

## 着色器的编写

现在我们开始着色器代码的编写，首先我们要创建两个文件，一般顶点着色器使用后缀`.vsh`，片段着色器使用后缀`.fsh`（这边命名可以随意，但是这样命名比较规范）。当然，其实我们也可以不用使用文件声明着色器代码，也可以直接声明一个常量字符串，将代码写入字符串，因为使用文件最终也是读取成字符串形式然后创建着色器，但是使用文件调理更加清晰。

顶点着色器代码：
``` 
#version 300 es
layout(location = 0) in vec4 vPosition;
void main() {
    gl_Position = vPosition;
}
```
解释：
1. 第一行代码声明了着色器的版本，V3.00；
2. 第二行代码`layout(location = 0)`限定符表示这个变量的位置在顶点属性0的位置；`in vec4 vPosition`表示输出的四维向量；
3. 最后三行就是着色器的main函数，着色器执行的起点；内部就是将输入的四维向量传递给名为`gl_Position`的特殊输出变量，每个顶点着色器必须在`gl_Position`变量中输出一个位置，这个变量定义传递到管线下一个阶段的位置；

片段着色器代码：
```
#version 300 es
precision mediump float;
out vec4 fragColor;
void main() {
    fragColor = vec4(1.0, 0.0, 0.0, 1.0);
}
```
解释：
1. 第一行代码声明了着色器的版本，V3.00；
2. 第二行代码声明着色器中浮点变量的默认精度；
3. 第三行代码声明一个输出变量`fragColor`，同样是四维向量，这个值将被传输到颜色缓冲区；
4. 最后三行代码就是着色器的man函数，着色器执行的起点；内部就是给定一个默认的思维向量颜色；

## 编译和加载着色器

上面定义了着色器的源代码，这里就可以将着色器加载到OpenGL ES中。这里定义一个`loadShaderWithName:type:`函数编译和加载着色器：

``` objectivec
- (GLuint)loadShaderWithName:(NSString *)name type:(GLenum)shaderType {
    
    // 从文件中读取着色器代码
    NSString *shaderPath = [[NSBundle mainBundle] pathForResource:name ofType:shaderType == GL_VERTEX_SHADER ? @"vsh" : @"fsh"];
    NSError *error;
    NSString *shaderString = [NSString stringWithContentsOfFile:shaderPath encoding:NSUTF8StringEncoding error:&error];
    if (!shaderString) {
        NSAssert(NO, @"读取Shader失败，%@", error.localizedDescription);
        return 0;
    }
    
    // 创建shader对象
    GLuint shader = glCreateShader(shaderType);
    if (!shader) {
        return 0;
    }
    
    // 获取shader内容
    const GLchar *shaderStringUTF8 = [shaderString UTF8String];
    glShaderSource(shader, 1, &shaderStringUTF8, NULL);
    
    // 编译shader
    glCompileShader(shader);
    
    // 查询shader编译状态
    GLint compiled;
    glGetShaderiv(shader, GL_COMPILE_STATUS, &compiled);
    if (!compiled) {
        GLint infoLen;
        glGetShaderiv(shader, GL_INFO_LOG_LENGTH, &infoLen);
        if (infoLen > 1) {
            GLchar *infoLog = malloc(sizeof(GLchar) * infoLen);
            glGetShaderInfoLog(shader, infoLen, NULL, infoLog);
            NSAssert(NO, @"编译shader失败，%s", infoLog);
            free(infoLog);
        }
        glDeleteShader(shader);
        return 0;
    }
    return shader;
}
```

## 创建一个程序对象并链接着色器

不同的着色器编译为一个着色器对象之后，必须连接到一个程序对象并一起链接，才能绘制图形。程序对象可以视为最终链接的程序。

``` objectivec
- (GLuint)createProgram:(NSString *)programName {

    // 获取着色器对象
    GLuint vertexShader = [self loadShaderWithName:programName type:GL_VERTEX_SHADER];
    GLuint fragmentShader = [self loadShaderWithName:programName type:GL_FRAGMENT_SHADER];
    
    // 创建program对象
    GLuint program = glCreateProgram();
    if (!program) {
        return 0;
    }
    
    // program关联shader
    glAttachShader(program, vertexShader);
    glAttachShader(program, fragmentShader);
    
    // 链接program
    glLinkProgram(program);
    
    // 查询program状态
    GLint linked;
    glGetProgramiv(program, GL_LINK_STATUS, &linked);
    if (!linked) {
        GLint infoLen;
        glGetProgramiv(program, GL_INFO_LOG_LENGTH, &infoLen);
        if (infoLen > 1) {
            GLchar *infoLog = malloc(sizeof(GLchar) * infoLen);
            glGetProgramInfoLog(program, infoLen, NULL, infoLog);
            NSAssert(NO, @"program链接失败，%s", infoLog);
            free(infoLog);
        }
        glDeleteProgram(program);
        return 0;
    }
    return program;
}
```

## 创建渲染上下文

现在我们已经有了链接着着色器对象的程序对象，下面就是开始渲染绘制操作。首先创建渲染上下文并设置为当前上下文，这里就直接使用OpenGL ES 3.0：

``` objectivec
EAGLContext *context = [[EAGLContext alloc] initWithAPI:kEAGLRenderingAPIOpenGLES3];
[EAGLContext setCurrentContext:context];
```

## 使用CAEAGLLayer作为渲染目标

在iOS中，我们先用`CAEAGLLayer`作为渲染目标，当然后面也有其他方式，我们先讨论这个方法：

```
- (void)initialize {
    // layer的创建，并添加到父layer上
    CAEAGLLayer *layer = [[CAEAGLLayer alloc] init];
    layer.frame = self.view.bounds;
    [self.view.layer addSublayer:layer];
    
    // 创建颜色渲染缓冲区，绑定到帧缓冲区，并将layer对象的存储绑定到OpenGL ES的渲染缓冲区对象
    [self bindRenderLayer:layer context:context];
}

- (void)bindRenderLayer:(CALayer <EAGLDrawable> *)layer context:(EAGLContext *)context {
    GLuint renderbuffer;
    GLuint framebuffer;
    glGenRenderbuffers(1, &renderbuffer);
    glBindRenderbuffer(GL_RENDERBUFFER, renderbuffer);
    [context renderbufferStorage:GL_RENDERBUFFER fromDrawable:layer];
    
    glGenFramebuffers(1, &framebuffer);
    glBindFramebuffer(GL_FRAMEBUFFER, framebuffer);
    glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_RENDERBUFFER, renderbuffer);
}
```

解释：`bindRenderLayer`方法主要是将渲染缓冲区与layer绑定，将layer作为渲染缓冲区的输出层。

## 加载几何形状和绘制图元

指定绘图窗口大小：
``` objectivec
glViewport(0, 0, self.drawableWidth, self.drawableHeight);

// 获取渲染缓存宽度
- (GLint)drawableWidth {
    GLint backingWidth;
    glGetRenderbufferParameteriv(GL_RENDERBUFFER, GL_RENDERBUFFER_WIDTH, &backingWidth);
    
    return backingWidth;
}

// 获取渲染缓存高度
- (GLint)drawableHeight {
    GLint backingHeight;
    glGetRenderbufferParameteriv(GL_RENDERBUFFER, GL_RENDERBUFFER_HEIGHT, &backingHeight);
    
    return backingHeight;
}
```

创建程序对象并使用：
``` objectivec
GLuint program = [self createProgram:@"glsl"];
if (!program) {
    return;
}
glUseProgram(program);
```

清除颜色缓冲区，避免将以前的内容加载到内存中造成高消耗，为确保最佳性能，应该在每次绘图之前调用：
``` objectivec
glClearColor(0.0, 0.0, 0.0, 1.0);
glClear(GL_COLOR_BUFFER_BIT);
```

指定三角形的几何形状，由(x, y, z)3个坐标点指定：

``` objectivec
GLfloat vVertices[] = {
    0.0f, 0.5f, 0.0f,
    -0.5f, -0.5f, 0.0f,
    0.5f, -0.5f, 0.0f
};
```

绘制图元：
``` objectivec
// 启用顶点并设置顶点数据
glEnableVertexAttribArray(GLKVertexAttribPosition);
glVertexAttribPointer(GLKVertexAttribPosition, 3, GL_FLOAT, GL_FALSE, 0, vVertices);

// 开始绘制
glDrawArrays(GL_TRIANGLES, 0, 3);
```

将渲染结果呈现到屏幕
``` objectivec
[context presentRenderbuffer:GL_RENDERBUFFER];
```

最终，就会在页面上绘画出一个三角形：

![Triangle Picture](/img/article/20190606/1.png)

# 使用GLKit绘制三角形

除了上述使用自定义着色器进行绘制，苹果本身也封装了简易着色器的GLKit供我们实现简单的绘制功能，下面就来介绍使用GLKit绘制三角形。

使用GLKit绘制有两种方式：一种是使用继承`GLKView`并复写`drawRect:`方法进行渲染操作，另一种就是使用`GLKView`的代理方法进行渲染。这两种方法都需要用到`GLKBaseEffect`类，这个类内部就实现了着色器的编译链接过程，简化了整个操作。

## 继承GLKView进行绘制

基础`GLKView`，并复写`drawRect:`方法，创建`GLKBaseEffect`实例并为OpenGL ES渲染准备一个效果，即将编译后的着色器程序绑定到上下文并返回，最后进行顶点设置并绘制：

``` objectivec
@interface TriangleGLKView ()

@property (nonatomic, strong) GLKBaseEffect *baseEffect;

@end

@implementation TriangleGLKView

- (instancetype)initWithFrame:(CGRect)frame context:(nonnull EAGLContext *)context {
    self = [super initWithFrame:frame context:context];
    if (self) {
        self.baseEffect = [[GLKBaseEffect alloc] init];
        self.baseEffect.constantColor = GLKVector4Make(0.0, 1.0, 0.0, 1.0);
    }
    return self;
}


- (void)drawRect:(CGRect)rect {
    
    glClearColor(1.0, 0.0, 0.0, 1.0);
    glClear(GL_COLOR_BUFFER_BIT);
    
    // 准备渲染，将编译后的着色器程序绑定到上下文并返回
    [self.baseEffect prepareToDraw];
    
    GLfloat vVertice[] = {
        0.0, 0.5, 0.0,
        -0.5, -0.5, 0.0,
        0.5, -0.5, 0.0
    };
    
    // 启用顶点并设置顶点数据
    glEnableVertexAttribArray(GLKVertexAttribPosition);
    glVertexAttribPointer(GLKVertexAttribPosition, 3, GL_FLOAT, GL_FALSE, 0, vVertice);

    // 开始绘制
    glDrawArrays(GL_TRIANGLES, 0, 3);
}

@end
```

## 使用代理绘制

设置上下文，创建`GLKView`实例并设置代理，实现代理方法，同样使用`GLKBaseEffect`类，与继承方法对比，就是将渲染操作在代理中调用：

``` objectivec
- (void)initialize {
    EAGLContext *context = [[EAGLContext alloc] initWithAPI:kEAGLRenderingAPIOpenGLES3];
    [EAGLContext setCurrentContext:context];

    GLKView *glkView = [[GLKView alloc] initWithFrame:self.view.bounds context:context];
    glkView.backgroundColor = [UIColor clearColor];
    glkView.delegate = self;
    [self.view addSubview:glkView];
    
    self.baseEffect = [[GLKBaseEffect alloc] init];
    self.baseEffect.constantColor = GLKVector4Make(0.0, 1.0, 0.0, 1.0);
    
    [glkView display];
}


- (void)glkView:(GLKView *)view drawInRect:(CGRect)rect {
    glClearColor(1.0, 0.0, 0.0, 1.0);
    glClear(GL_COLOR_BUFFER_BIT);
    
    [self.baseEffect prepareToDraw];
    
    GLfloat vVertice[] = {
        0.0, 0.5, 0.0,
        -0.5, -0.5, 0.0,
        0.5, -0.5, 0.0
    };
    
    glEnableVertexAttribArray(GLKVertexAttribPosition);
    glVertexAttribPointer(GLKVertexAttribPosition, 3, GL_FLOAT, GL_FALSE, 0, vVertice);
    glDrawArrays(GL_TRIANGLES, 0, 3);
}
```

# 结语

简单的三角形绘制就结束了，在理解了整个渲染过程，不管是自定义着色器还是使用封装的GLKit方式渲染，包括后续的Metal渲染，都能够找到丝丝关联，在学习中更加有关联性。[Demo](https://github.com/Quincy-QC/OpenGL-ES-Study)