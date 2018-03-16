[<< 返回到上级](index.md)

这里将介绍OpenGL API的使用博客文章，开发OpenGL围绕下面几点展开

0. 认识OpenGL和相关扩展库
1. 基础骨架
2. GLSL基础
3. 贴图
4. 矩阵
5. 相机
6. 资源实例化
7. 漫反射光照
8. 更多光照

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
1. VAO 与 VBO：vao可看成是vbo的数组形式，每个vbo存储模型数据。另还有VIO 顶点索引模式
2. program 和 shader: program至少需要挂载顶点和片段shader。shader代码中的参数，program编译后，可以获取其索引并传值。shader里模型每一个顶点数据，要通过glEnableVertexAttribArray和glVertexAttribPointer指定  
[转到骨架](#1)  
---------------------  

## **2. GLSL基础**

## **3. 贴图**

## **4. 矩阵**

## **5. 相机**

## **6. 资源实例化**

## **7. 漫反射光照**

## **8. 更多光照**

    



