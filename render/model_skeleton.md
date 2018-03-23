[<< 返回到主页](index.md)

**这里将介绍模型几何数据的博客文章**  

场景中的模型为静态和动态，均可以带Morpher变形动画。
在游戏中，这两种不同的模型在硬件渲染下，有不同的处理方式。对于静态模型，数据在glBufferData参数设置我GL_STATIC_DRAW，而动态模型设置为GL_DYNAMIC_DRAW。对于静态模型，直接将数据传输到显卡，一次写入，后期渲染性能会很快。而动态模型，变化的数据部分需要每次都传输，较静态模型要慢。   

接下来介绍的模型文件采用自定义的一套, 采用二进制格式。  



## **认识一些概念**  

1. 三角面存储格式  
    1. 顶点索引模式  
    2. 三角面模式  
    3. 三角带模式  
2. 数据压缩： float转short (比如float 0.1f转为 0.1f * 65526 = 6553，最后6553 / 65526 = 0.1xf)对数据压缩  
3. 一个模型数据由多个子模型构成  
4. 骨骼动画对比静态模型由骨骼初识数据和关键帧数据，后期动画播放时，骨骼根据时间从关键帧插值得到变换信息，再根据顶点与骨骼的权重绑定关系，更新位置等变化数据，然后将这部分数据传输到显卡进行一次渲染。  
5. 在骨骼动画中，会存在其他绑定点，如武器的挂点。一般这种骨骼是不动的  
6. 在做骨骼影响蒙皮时，骨骼的数据和蒙皮的数据都是相对自身坐标系的，在游戏场景中需要将骨骼的初始绑定矩阵求逆矩阵，再做后面的计算。  
7. 一般对于一个物体，所有的动画共用一套骨骼  
8. 骨骼位移可带可不带  
9. 每个顶点信息包含  
    1. position 位置 3个float  
    2. normal   法线 3个float  
    3. color    颜色 4个float  
    4. tangent  切线 3个float  
    5. binormal 次发现 3个float  
    6. blend weight  权重值  4个float  
    7. blend indices 骨骼索引值 4个float   
    8. TEXCOORD0   默认贴图坐标 2个float  
    9. TEXCOORD1   光照贴图坐标 2个float  
    ...  
10. 关注morpher动画和物体挂点  

## **与物体相关的数据组织结构**  

如果定义每个子物体为MeshObject，则数据定义为：  
```
enum MeshObjectOptionEnum
{
    MOO_LARGE_MESH          = 0x0080,
    MOO_COLLIDER            = 0x0040,
    MOO_CONTROLLABLE        = 0x0020,
    MOO_VERTEX_ANIMATION    = 0x0010,
    MOO_NO_COLLISION        = 0x0008,
    MOO_UVDIVIDED           = 0x0004,
    MOO_DENSITY             = 0x0002,
    MOO_INVISIBLE           = 0x0001,
};
enum
{
    MOT_UNKNOWN             = 0,
    MOT_TRIANGLES           = 1,
    MOT_INDEXED_PRIMITIVES  = 2,
    MOT_TRIANGLE_STRIPS     = 3,
};
enum
{
    DST_DEFAULT             = 0,
    DST_SHORT               = 1,
    DST_FLOAT               = 2,
};

// 版本号
__u16 mCurrentVersion;
// 一个掩码，有两个标识，低8位表示MeshObjectOptionEnum，8-12位表示数据是否压缩，高4位表示子模型是顶点索引模式，三角带还是三角面模式存储
__u16 m_mask;
// 模型名称
char m_meshObjectName[];
// 一个id，指父节点id
__u32 m_meshObjectHandle;
// 此模型id
__u32 m_parentHandle;

// 顶点个数
__u32 m_vertexCount;  
float m_offset[3];
// 缩放系数，同uv一样
double m_vertexScale; 
// 顶点数据可能被压缩 
unsigned char *m_vertexPosArray;  
// 如果是顶点索引模式，这里m_faceIndexArray != 0
__u16 *m_faceIndexArray;
// 面索引数量
__u32 m_faceIndexCount;
// 是否做了平滑处理
unsigned char m_smoothGroup;

// 法线数组
char *m_normalArray;
// 切向量数组
char *m_tangentArray;
// 次发现数组
char *m_binormalArray;
// 顶点颜色数组
__u32 *m_vertexColorArray;
// uv组
HexUVGroups *m_uvGroups;
// 材质名
char m_materialName[];
// 材质id
__u32 m_materialId; 
// 是否透明设置
unsigned char m_opacity;
```

其中HexUVGroups:  
```
HexUVGroup **m_UVGroupList; // uv组，默认美术会提供一套  
__u32 m_UVGroupSize; // 若大于65535，则为较大的模型
unsigned char m_internalSize; // uv组个数
__u16 mCurrentVersion;// 版本号
```

HexUVGroup信息为：  
```     
enum
{
    UVT_DEFAULT     = 0,        //default, equal uv for base-texture
    UVT_NORMAL_MAP  = 1,        //uv for normal/gloss map
    UVT_LIGHT_MAP   = 2,        //uv for light-map
    UVT_RESERVE     = 3,        //maxium texture layer 
};

__u16 mCurrentVersion; // 版本号
unsigned char m_UVDataType; // 压缩为short，非压缩为float
unsigned char m_UVType; // 默认贴图，光照贴图，法线贴图
double m_UVScale; // 压缩后，缩放系数  
unsigned char *m_UVArray;// uv数据，根据m_UVDataType读取，每两个为一个坐标
__u32 m_UVCount; // uv的个数
```

上面针对的是几何数据信息，接下来描述骨骼结构和骨骼绑定信息。   
物体的一个挂点结构HexNodeDummyObject：  
```
// 挂点名
char m_nodeName[]; 
// 挂点id
__u32 m_nodeHandle;
// 挂点空间信息
float m_transform[7];
```

物体的一个挂点数组HexNodeDummy：  
```
// 挂点数
__u16 m_dummyCount;
// 挂点数组
HexNodeDummyObject **m_dummyArray;
```

骨骼带有层级关系，用一棵树来描述HexNodeTree：  
```
// 版本号
__u16 mCurrentVersion;
// 骨骼名称
char m_nodeName[];
// id
__u32 m_nodeHandle;
// 初始值，平移3个float，旋转4个float
float m_transform[7];
// 子节点数
__u16 m_numChildren;
// 子节点数组
HexNodeTree **m_children;
// 组id(对骨骼分组)
__u16 m_groupId;
```

接下来说明每一个顶点绑定到骨骼的数据HexSkeletonBindingNode：  
```
struct HexNodeWeight
{
    // 骨骼id
    __u32 m_target;
    // 权重值
    float m_weight;
}
// 顶点受骨骼影响的数量(最多为4个)
unsigned char m_weightCount;
// 权重数组
HexNodeWeight **m_nodeWeightArray;
// 当前版本号
__u16 mCurrentVersion;
```

每一个子模型对应的绑定数据HexSkeletonPiece：
```
// 版本号
__u16 mCurrentVersion;
// id值，对应子模型
__u32 m_pieceHandle;
// 对应子模型顶点数
__u16 m_bindingNodeCount;
// 子模型顶点绑定数组
HexSkeletonBindingNode **m_bindingNodeArray;
```

每一个模型的绑定数据HexSkeletonBinding：  
```
// 名称
char m_skeletonName[];
// 对应子模型个数
__u16 m_bindingPieceCount;
// 子模型数组
HexSkeletonPiece **m_bindingPieceNodeArray;
// 当前版本号
__u16 mCurrentVersion;
```

接下来分析骨骼动画数据：   
单独一根骨骼在一个动画上的关键帧数据HexSkeletonNodeAnimation：  
```
// 版本号
__u16 mCurrentVersion;
// 父id
__u32 m_parent;
// 帧数
__u16 m_frameCount;
// 7 * m_frameCount 关键帧数组
float *m_posQuatArray;
// 骨骼名
char m_boneName[64];
```

所有骨骼在一个动画上的数据HexSkeletonAnimation：  
```
// 动画名
char m_animationName[ANIM_NAME_SIZE + 1];
// 关键时间点序列 m_frameCount 个 float
float *m_frameArray;
// 帧数
__u16 m_frameCount;
// 帧率
__u16 m_frameRate;
// 所有骨骼的关键帧数据
HexSkeletonNodeAnimation **m_nodeAnimationArray;
// 骨骼数量
__u16 m_animationNodeCount;
// 版本号
__u16 m_version;
```

多个动画集合HexSkeletonAnimations：  
```
// 动画数量
unsigned char m_animationCount;
// 动画数据数组
HexSkeletonAnimation **m_animationArray;
// 版本号
__u16 mCurrentVersion;
```

变形Morpher动画数据分析  
每一个点的数据HexVertexMorphObject：  
```
// 数据类型
unsigned char m_vertexDataType; 
// 顶点维度
__u16 m_vertexCount;
// 顶点
unsigned char *m_vertexPosArray;
// 法线
char *m_normalArray;
// 偏移值
float m_offset[3];
// 缩放值
double m_scale;
// 对应object索引
__u32 m_meshObjectIndex;
```

一个顶点的morpher序列HexVertexMorphAnimationFrame：  
```
// 所有点的变形位置数组
HexVertexMorphObject **m_vertexMorphObjectList;
// 序列数量
__u16 m_vertexMorphObjectCount;
```

一个morpher动画所有顶点的序列HexVertexMorphAnimation：  
```
// 动画名
char m_animationName[];
// 顶点序列数组
HexVertexMorphAnimationFrame **m_vertexMorphFrameList;
// 顶点数
__u16 m_vertexMorphFrameCount;
// 帧率
__u16 m_frameRate;
// 时间序列
float *m_frameArray;
```

所有morpher动画HexVertexMorphAnimations：  
```
// 动画数量
unsigned char m_animationCount;
// 动画数组
HexVertexMorphAnimation **m_animationArray;
```

## **渲染引擎对物体数据封装**  

