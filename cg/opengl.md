[<< 返回到上级](index.md)

这里将介绍OpenGL API的使用博客文章，开发OpenGL围绕下面几点展开

[0. 认识OpenGL和相关扩展库](#0)  
[1. 基础骨架](#1)  
[2. GLSL基础](#2)  
[3. 贴图](#3)  
[4. 矩阵](#4)  
[5. 相机](#5)  
[6. 资源实例化](#6)  
[7. 漫反射光照](#7)  
[8. 更多光照](#8)  

<span id="0"></span>
## **0. 认识OpenGL和相关扩展库**

OpenGL是跨平台的开源图形库。当你在使用OpenGL之前，需要对其进行初始化。由于跨平台， 则初始化没有一个固定的标准。  
OpenGL的初始化分两个阶段：

第一个阶段：  
    创建一个OpenGL上下文环境。
    这个上下文环境存储了所有与OpenGL相关的状态（OpenGL是一个状态机）。
    上下文位于操作系统中某个进程中，一个进程可以创建多个上下文，每一个上下文都可以描绘一个不同的可视界面，就像应用程序中的窗口。  

第二个阶段：  
    定位所有需要在OpenGL中使用的函数。
    因为OpenGL只是一个标准，具体的实现是由驱动开发商针对特定显卡实现的。  
    由于OpenGL驱动版本众多，它大多数函数的位置都无法在编译时确定下来，需要在运行时查询。
    任务就落在了开发者身上，开发者需要在运行时获取函数地址并将其保存在一个函数指针中供以后使用，取得地址的方法因平台而异。  

基于上面两个阶段，使用OpenGL开发时，引入两个扩展库GLFW和GLEW：

GLFW：创建OpenGL窗口。类似有GLUT，FREEGLUT。  
    GLFW有哪些优势？glut太老了，最后一个版本还是90年代的。freeglut完全兼容glut，算是glut的代替品，功能齐全，但是bug太多，稳定性也不好(据说没使用过)，GLFW应运而生    

GLEW：对OpenGL在不同平台下做一层封装，开发者只需调用即可。从而不依赖于操作系统，不依赖具体的显卡实现，达到跨平台的效果。  

接下里的代码释义依赖于GL，GLEW，GLFW库  

[跳转到开头](#0)  
--------------  

<span id="1"></span>
## **1. 基础骨架**
#### 1.1 窗口创建与上下文设置
代码片段
```
    glfwInit();
    GLFWwindow* gWindow = glfwCreateWindow((int)SCREEN_SIZE.x, (int)SCREEN_SIZE.y, "OpenGL Tutorial", NULL, NULL);
    glfwMakeContextCurrent(gWindow);
```
1.1.1 首先初始化glfw
1.1.2 创建窗口，参数为 窗口尺寸，窗口标题等
1.1.3 设置上下文，OpenGL渲染输出到指定的窗口

#### 1.2 定位OpenGL函数
代码片段
```
    glewInit();
```
a. 初始化glew，确定所需要的OpenGL函数

#### 1.3 初始化模型等资源
绘制一个简单的三角形，其初始化过程如下：  
**1.3.1 初始化 Program gProgram** 

**1.3.1.1 初始化Shader**
```
GLint shader_object = glCreateShader(shaderType);
const char* code = shaderCode.c_str();
glShaderSource(shader_object, 1, (const GLchar**)&code, NULL);
glCompileShader(shader_object);
GLint status;
glGetShaderiv(shader_object, GL_COMPILE_STATUS, &status);
```
a. 根据shaderType(顶点着色器还是片段着色器)创建shader id  
b. 根据shaderCode着色器文件路径，将代码指定到 shader_object 里  
c. 对着色器代码编译  
d. 检测着色器编译状态  

**1.3.1.2 初始化 Program**
```
GLint _object = glCreateProgram();
glAttachShader(_object, vert_shader_object);
glAttachShader(_object, frag_shader_object);
glLinkProgram(_object);
glDetachShader(_object, vert_shader_object);
glDetachShader(_object, frag_shader_object);
GLint status;
glGetProgramiv(_object, GL_LINK_STATUS, &status);
```
a. 创建 Program id  
b. 顶点和片段着色器挂载到 program上  
c. 将着色器链接到一起  
d. 卸载着色器  
e. 检测状态，判断有无错误  

**1.3.2 初始化Triangle(GLuint gVAO = 0; GLuint gVBO = 0;)**
```
glGenVertexArrays(1, &gVAO);
glBindVertexArray(gVAO);
glGenBuffers(1, &gVBO);
glBindBuffer(GL_ARRAY_BUFFER, gVBO);
GLfloat vertexData[] = {
    //  X     Y     Z
     0.0f, 0.8f, 0.0f,
    -0.8f,-0.8f, 0.0f,
     0.8f,-0.8f, 0.0f,
};
glBufferData(GL_ARRAY_BUFFER, sizeof(vertexData), vertexData, GL_STATIC_DRAW);
glEnableVertexAttribArray(gProgram->attrib("vert"));
glVertexAttribPointer(gProgram->attrib("vert"), 3, GL_FLOAT, GL_FALSE, 0, NULL);
glBindBuffer(GL_ARRAY_BUFFER, 0);
glBindVertexArray(0);
```
a. 指定可用的gVAO  
b. 绑定gVAO  
c. 指定可用的gVBO  
d. 绑定gVBO，且为GL_ARRAY_BUFFER  
e. 三角形顶点数据  
f. 将gVBO大小设置为sizeof(vertexData)  ，并将顶点数据拷贝到gVBO显存，且是GL_STATIC_DRAW静态  
g. 开启vert位置，并指定该位置所需数据类型以及个数  
h. 重置vbo  
i. 重置vao  

**1.3.3 渲染循环**
```
while(!glfwWindowShouldClose(gWindow))
{
    glfwPollEvents();
    Render();
}
```
a. 处理输入事件，鼠标，键盘等  
b. 渲染物体  

**1.3.3.1 渲染Render()**
```
glClearColor(0, 0, 0, 1);
glClear(GL_COLOR_BUFFER_BIT);
glUseProgram(gProgram->object());    
glBindVertexArray(gVAO);
glDrawArrays(GL_TRIANGLES, 0, 3);
glBindVertexArray(0);
glUseProgram(0);
glfwSwapBuffers(gWindow);
```
a. 设置窗口背景色  
b. 清空颜色缓冲区  
c. 使用 program  
d. 绑定gVAO  
e. 提交绘制指定  
f. 重置vao  
g. 重置program  
h. 刷新帧缓冲区到窗口  

**1.4 骨架小结**
1. 初始化glfw和glew
2. 渲染循环
    1. 处理事件
    2. 渲染所有物体
        1. Program use(事先创建好)
        2. VAO绑定 (根据模型数据事先设定好)
        3. 提交绘制指令
        4. 重置vao
        5. 重置program
        6. 重复上述步骤，只有所有物体都被处理
    3. 刷新缓冲区

**1.5 核心函数**
1. glClear()
2. glUseProgram()
    1. glCreateProgram()
    2. glAttachShader()
        1. glCreateShader()
        2. glShaderSource()
        3. glCompileShader()
        4. glGetShaderiv()
    3. glLinkProgram()
    4. glDetachShader()
    5. glGetProgramiv()
    6. glGetAttribLocation()
    7. glGetUniformLocation()
3. glBindVertexArray()
    1. glGenVertexArrays()
    2. glGenBuffers()
    3. glBindBuffer()
    4. glBufferData()
    5. glEnableVertexAttribArray()
    6. glVertexAttribPointer()
4. glDrawArrays()
5. glfwSwapBuffers()

**1.6 一般说明**
1. VAO 与 VBO：vao可看成是vbo的数组形式，每个vbo存储模型数据。另还有VIO 顶点索引模式。绑定vbo后，需要指定如何去解析数据块。
2. program 和 shader: program至少需要挂载顶点和片段shader。shader代码中的参数，program编译后，可以获取其索引并传值。shader里模型每一个顶点数据，要通过glEnableVertexAttribArray和glVertexAttribPointer指定，另外glVertexAttrib可以设置  

[转到骨架](#1)   

<span id="2"></span>  
## **2. GLSL基础**    

作为OpenGL着色器语言，glsl代码格式类似于c代码。在此，主要是简单介绍一下glsl，以及在OpenGL如何使用以及需要注意的地方  

着色器有几种，根据OpenGL文档，主要是Vertex Shader顶点着色器， Tessellation Shader曲面细分着色器，Geometry Shader几何着色器，Fragment Shader着色器。其中曲面细分和几何着色器是可选项，顶点着色器和片段着色器是必须项，一般我们只用到这两个就可以。  

GLSL版本之间限定符不一样(1.3以前使用attribute和varying，1.4以后用in，out或者inout)，需要留意版本问题。

**2.1 向量**    
    [|i|u|b]vec[2|3|4]  
    浮点、整数、无符号整数、布尔向量，维度为2维，3维，4维  
    取值可以使用xyzw, rgba, stpq  
    支持调换操作，如color.rgba = color.bgra  

**2.2 矩阵**  
    矩阵就是一个由向量组成的数组.  
    mat(ixj) = mat[2|3|4]x[2|3|4]  
    *矩阵存储分列式和行式存储*  
    矩阵数据获取，mat3[i].xyz  
    矩阵乘以向量，即对向量做变换 p = mat \* v  
    矩阵乘以矩阵，表示累积变换的最终变换，需满足 前一个的列数与后一个的行数值相等  


**2.3 限定符**  

介绍常用的几个限定符  
|限定符|描述|
|:----:|:----:|
|highp，mediump，lowp，invariant|精度限定|
|const|声明变量或函数的参数为只读类型|
|in，attribute|只能存在于vertex shader中,用于保存顶点或法线数据,它可以在数据缓冲区中读取数据|
|out，varying|主要负责在vertex 和 fragment 之间传递变量|
|inout|复制到函数中并在返回时复制出来|
|uniform|在运行时shader无法改变uniform变量,程序传递给shader变换矩阵,材质,光照参数等|

**2.4 流控制与纹理**  

特殊的有discard，使用discard会退出片段着色器，不执行后面的片段着色操作  
sampler2D 外部传入贴图，一般是uniform，程序传入  

**2.5 内置变量**  

内置变量以及内置常量  
|变量名|描述|
|:----:|:----:|
|gl_Position|放置顶点坐标信息vec4|
|gl_PointSize|gl_PointSize 需要绘制点的大小(只在gl.POINTS模式下有效)float|
|gl_FragCoord|片元在framebuffer画面的相对位置vec4|
|gl_FrontFacing|标志当前图元是不是正面图元的一部分bool|
|gl_PointCoord|经过插值计算后的纹理坐标,点的范围是0.0到1.0 vec2|
|gl_FragColor|设置当前片点的颜色 vec4 RGBA color|
|gl_FragData[n]|设置当前片点的颜色,使用glDrawBuffers数据数组vec4 RGBA color|
|gl_MaxVertexAttribs|>=8，int,表示在vertex shader中可用的最大attributes数|
|gl_MaxVertexUniformVectors|vertex shader最大uniform vectors数|
|gl_MaxVaryingVectors|vertex shader最大varying vectors数|
|gl_MaxVertexTextureImageUnits|vertex shader最大纹理单元数|
|gl_MaxCombinedTextureImageUnits |vertex shader最多支持多少个纹理单元|
|gl_MaxTextureImageUnits |片元着色器中能访问的最大纹理单元数|
|gl_MaxFragmentUniformVectors |片元着色器中可用的最大uniform vectors数|
|gl_MaxDrawBuffers|可用的drawBuffers数|
|gl_DepthRange|表明全局深度范围|

**2.6 内置函数**  
内置变量以及内置常量  
|函数类型|
|:----:|
|通用函数|
|角度&三角函数|
|指数函数|
|几何函数|
|矩阵函数|
|向量函数|
|纹理查询函数|

**2.7 例子**  
下面的shader如果你可以一眼看懂,说明你已经对glsl语言基本掌握了  
Vertex Shader:  
```
uniform mat4 mvp_matrix; //透视矩阵 * 视图矩阵 * 模型变换矩阵
uniform mat3 normal_matrix; //法线变换矩阵(用于物体变换后法线跟着变换)
uniform vec3 ec_light_dir; //光照方向
attribute vec4 a_vertex; // 顶点坐标
attribute vec3 a_normal; //顶点法线
attribute vec2 a_texcoord; //纹理坐标
varying float v_diffuse; //法线与入射光的夹角
varying vec2 v_texcoord; //2d纹理坐标
void main(void)
{
 //归一化法线
 vec3 ec_normal = normalize(normal_matrix * a_normal);
 //v_diffuse 是法线与光照的夹角.根据向量点乘法则,当两向量长度为1是 乘积即cosθ值
 v_diffuse = max(dot(ec_light_dir, ec_normal), 0.0);
 v_texcoord = a_texcoord;
 gl_Position = mvp_matrix * a_vertex;
}
```

Fragment Shader:  
```
precision mediump float;
uniform sampler2D t_reflectance;
uniform vec4 i_ambient;
varying float v_diffuse;
varying vec2 v_texcoord;
void main (void)
{
 vec4 color = texture2D(t_reflectance, v_texcoord);
 //这里分解开来是 color*vec3(1,1,1)*v_diffuse + color*i_ambient
 //色*光*夹角cos + 色*环境光
 gl_FragColor = color*(vec4(v_diffuse) + i_ambient);
}
```

GLSL 1.4版本以后，去除了 varying和attribute，使用out和in代替  

**总结：**  
1. glUniform() 设置 uniform参数值
2. glEnableVertexAttribArray 和 glVertexAttribPointer 函数指定 in或者attribute的数据，shader再对每个点的数据按照指定的字节数去获取，从而设置好点的参数值  
3. varying在vertex shader 和 fragment shader之间共享传值(这里需要注意：fragment是在vertex指定的数值上，做插值得到。因为fragment针对光栅化后的像素来计算，而vertex只是针对点，还没经过光栅化)  
4. layout(location = 0)可以设定顶点数据的片段位置，然后通过程序设置 

[跳转到前面](#2)

## **3. 贴图**

现在往基础骨架中，添加一张贴图。  
**3.1 初始化贴图信息**  
3.1.1 bmp图像加载
```
    int width, height, channels;
    unsigned char* pixels = stbi_load(filePath.c_str(), &width, &height, &channels, 0);
    if(!pixels) throw std::runtime_error(stbi_failure_reason());
    
    Bitmap bmp(width, height, (Format)channels, pixels);
    stbi_image_free(pixels);
    bmp.flipVertically();
```

3.1.2 封装Texture贴图信息  
```
Texture(const Bitmap& bitmap, GLint minMagFiler, GLint wrapMode) :
    _originalWidth((GLfloat)bitmap.width()),
    _originalHeight((GLfloat)bitmap.height())
    {
        glGenTextures(1, &_object);
        glBindTexture(GL_TEXTURE_2D, _object);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, minMagFiler);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, minMagFiler);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, wrapMode);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, wrapMode);
        glTexImage2D(GL_TEXTURE_2D,
                     0, 
                     TextureFormatForBitmapFormat(bitmap.format()),
                     (GLsizei)bitmap.width(), 
                     (GLsizei)bitmap.height(),
                     0, 
                     TextureFormatForBitmapFormat(bitmap.format()), 
                     GL_UNSIGNED_BYTE, 
                     bitmap.pixelBuffer());
        glBindTexture(GL_TEXTURE_2D, 0);      
    }

```
1. 创建贴图id  
2. 绑定id  
3. 设置Filter参数   
    1. GL_TEXTURE_MIN_FILTER  
    2. GL_TEXTURE_MAG_FILTER  
4. 设置Wrap参数  
    1. GL_TEXTURE_WRAP_S  
    2. GL_TEXTURE_WRAP_T  
5. 设置贴图数据  
6. 重置贴图id绑定为0  

主要是针对GL_TEXTURE_2D 2D纹理。  

Filter参数：针对纹理尺寸和图形尺寸而言    
1. GL_TEXTURE_MAG_FILTER(纹理尺寸小于图形尺寸)：GL_NEAREST 和 GL_LINEAR  
2. GL_TEXTURE_MIN_FILTER(纹理尺寸大于等于图形尺寸)：GL_NEAREST和GL_LINEAR以及MIPMAP相关选项  
GL_NEAREST：计算快，但没有过度的效果，锯齿明显  
GL_LINEAR：计算慢，插值计算，较为平滑  

Wrap参数：针对纹理坐标值而言，主语是超过了纹理坐标范围(小于0或者大于1)  
1. GL_TEXTURE_WRAP_S：水平方向，u，x  
2. GL_TEXTURE_WRAP_T：垂直方向，v，y 
GL_REPEAT：超过的部分进行重复，可以将图形分成多个格子的形式  
GL_CLAMP或GL_CLAMP_TO_EDGE：简单的截断，小于0取0，大于1取1  

Mipmaps：level等级  
当纹理尺寸大于图形尺寸时的一种技术手段。当模型很远时，z值较大，选择较小的贴图，因为不用关心细节。mipmap图片可以有OpenGL自动创建保存，需要的时候进行加载。对应level参数  

**3.2 贴图使用并渲染**   
```
gProgram->use();
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, gTexture->object());
gProgram->setUniform("tex", 0);// glActiveTexture GL_TEXTURE0对应
```
1. 执行当前program  
2. 激活0号贴图  
3. 绑定贴图id  
4. 将贴图信息传入shader sampler2D参数  
注意：  
glEnable(GL_TEXTURE_2D);开启  
uniform sampler2D：shader里纹理采样器设置  
支持多重纹理(原始贴图，光照贴图，法线贴图等)  
可能会这样写glUniform1i(texLoc, BaseID,但这做法是错的，GLSL的sample2D只接受纹理单元的索引号，GL_TEXTURE0+i，还有一个要注意的地方就是要用glUniform1i函数，而不要用glUniform1ui()  

## **4. 矩阵**

```
uniform mat4 projection;
uniform mat4 camera;
uniform mat4 model;
```
在shader里，程序进行设置model矩阵。若在循环内，修改模型矩阵，每帧都更新最新的model矩阵，则在窗口能看到图形的变化。  
顶点着色器计算时，常用的就是模型本地顶点进行mvp变换到齐次裁剪空间，渲染管线进行后期的裁剪，透视除法，图元装配等等计算    

## **5. 相机**
```
uniform mat4 projection;
uniform mat4 camera;
uniform mat4 model;
```
在shader里，程序修改相机camera矩阵，则可以观察到 场景里不同的位置。  
1. 鼠标move对相机的旋转操作时，对鼠标的增量进行累加，然后计算旋转矩阵(该表欧拉角)  
2. 鼠标滚轮拉近拉远(fov改变)  
3. 键盘WASD对相机进行平移操作(局部坐标系XOY平面平移)  

## **6. 资源实例化**

**6.1 基础数据封装**  
现在结合前面所讲，对一个模型进行简单封装：  
```
struct ModelAsset {
    Program* shaders;
    Texture* texture;
    GLuint vbo;
    GLuint vao;
    GLenum drawType;
    GLint drawStart;
    GLint drawCount;
}
```
1. shaders为创建program，并且链接着色器  
2. texture为创建纹理贴图，指定图片资源信息  
3. vao，vbo指定模型数据信息(顶点坐标，纹理坐标等)  
4. drawType, drawStart,drawCount为glDrawArrays提供参数信息  

```
struct ModelInstance {
    ModelAsset* asset;
    glm::mat4 transform;
};
```
1. 在上面的结构体中，添加一个变换矩阵(此处理解为局部坐标系与世界坐标系之间的联系)  

**6.2 管理模型并渲染**  
```
std::list<ModelInstance> gInstances;
CreateInstances();
void RenderInstance(const ModelInstance& inst);
```
1. 保存创建的所有实例  
2. 创建所有对象并保存  
3. 渲染所有实例  

此处只是对之前所讲的一个扩展，能够处理多个物体。当存在多个物体时，会出现遮挡等情形，必要的时候需要开启深度测试或者融合处理达到透明  

## **7. 光照**

上诉说完，还剩下一个光照来模拟更真实的世界  
```
struct Light {
    vec3 position;
    vec3 intensities;
};
```
1. 灯光位置  
2. 灯光颜色  

此处只提供光的位置，而没有方向    
其他光照灯型为：平行光，点光源，聚光灯，区域光，环境光等  

光照计算一般在着色器里进行，并且模型的数据中要包含顶点法线，计算过程：  
1. 将顶点映射到世界坐标系 vertPosition  
2. 计算点到灯光位置的方向 surfaceToLight = light.position - vertPosition  
3. 将顶点法线映射到世界坐标系normal  
4. 计算normal与surfaceToLight的cos值，得到入射角  
5. 根据入射角度与灯光强度计算反映在这个点上的光强，然后与基础贴图颜色做乘积得到最终的颜色。颜色透明通道默认为基础贴图的alpha值  

## **8. 更多光照**

按照光源分类进行封装，然后将参数传递给着色器处理即可。  
光照计算可在顶点作色器处理，也可以在片段着色器处理  
因为作为参数传入，则不涉及到光照的OpenGL函数。老版本的OpenGL函数，有单独介绍光照的函数API。  



