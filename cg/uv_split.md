[<< 返回到上页](../index.md)

**这里将介绍UV划分的博客文章**  

3D模型中三角面对应到2D纹理贴图uv坐标值的计算，中间存在一个3D到2D降维的过程  
1. 怎样尽可能避免信息丢失？  
2. 对于模型中，顶点的相连关系怎样确保在uv坐标系下对应？  
3. 如何尽可能的占据纹理贴图空间？  

在计算模型的uv坐标时，整体的策略是对模型的三角面进行XOY，XOZ，YOZ平面投影，选择投影面积最大的轴平面投影并去掉重要性小的分量值，得到一个2D坐标。  

对投影后的2D坐标，进行按共享点分组，即可获得一片一片的网格数据。  

对分组后的uv坐标值，寻找最小的矩形包围盒，根据这个包围盒在纹理图片上找到对应的区域；找到对应的区域后，进行坐标的转换，将uv值限制在0-1之间。  

为了尽可能的铺满整个纹理贴图，需要加一个放大系数来控制所有的uv面。  

其中涉及到的几个主要计算：  
[1. 三角面投影到2D计算](#1)  
[2. 2D点进行分组](#2)  
[3. 寻找最小的矩形包围盒](#3)  
[4. 分区算法](#4)  
[5. 放大系数的选择](#5)  

<span id="1"></span>
## **1. 三角面投影到2D计算**  

1.1 计算三角面的法线并对每个分量求绝对值，得到normal  
1.2 分量中选择最大的那一个维度值d，则投影平面为梁歪两个维度确定的平面  
1.3 三角面顶点数据去除最大维度的那个数据，则变成2d  

数据组织：  
```
int plane; (1表示YZ，2表示XZ，0表示XY 平面)
Vector3 p; 将需要丢弃的数据放在p[2]下，将另外两个维度的数据按照xyz的顺序放在0,1位置  
```

<span id="2"></span>
## **2. 2D点进行分组**  

对顶点进行分组，按2个点，1个点分组(可以按只要有共顶点就分在一组)  
此处的分组算法可以选择 **并查集 算法**  

<span id="3"></span>
## **3. 寻找最小的矩形包围盒**  

上面分组完成后，对每一组计算其最小的矩形包围盒(x，y值参与计算)  
1.1 对组内数组按照5度进行旋转18次(即共90度)，每次得到一个矩形  
1.2 对每次的矩形求面积，选择最小的那个面积，并确定旋转的角度值  
1.3 得到最小的矩形，获得其width和height  

<span id="4"></span>
## **4. 分区算法**  

由上面的矩形长和宽，乘以放大系数，在一个固定大小的贴图区域内，寻找矩形区域，记录起始坐标点。  

分区算法: 二叉树和贪心算法  
核心代码如下：  
```
struct SPartition
{
    float left, top, width, height;
    SPartition *lChild, *rChild;
    __int32 texIndex;
    bool isChildren;
}

SPartition *SPartition::Insert(float w, float h)
{
    SPartition *newNode = 0;
    float dw, dh;
    if (!isChildren && 0==texIndex)
    {
        dw = width - w;
        dh = height - h;
        if ((*(int *)&dw >= 0) && (*(int *)&dh >= 0))
        {
            if ((*(int *)&dw == 0) && (*(int *)&dh == 0))
            {
                return this;
            }
            else
            {
                lChild = new SPartition;
                rChild = new SPartition;
                float t = dh - dw;
                if (*(int *)&t < 0)
                {
                    lChild->CreateNode(left, top, w, height);
                    rChild->CreateNode(left + w + 1.0f, top, dw - 1.0f, height);
                    isChildren = true;
                }
                else
                {
                    lChild->CreateNode(left, top, width, h);
                    rChild->CreateNode(left, top + h + 1.0f, width, dh - 1.0f);
                    isChildren = true;
                }
                return lChild->Insert(w, h);
            }
        }
        else
            return 0;   
    }
    else 
        if (isChildren)
        {
            newNode = lChild->Insert(w, h);
            if(!newNode)
                newNode = rChild->Insert(w, h);
        }
        else 
            return 0;
    return newNode;
}
```
初始化时，为贴图的大小，起始坐标为(0, 0)  
后期根据 Insert 函数，查找区块  

<span id="5"></span>
## **5. 放大系数的选择**  
使用二分法进行放大系数的查找，然后不断做uv划分尝试，直到找到最好的放大系数，然后根据这个放大系数再处理一次(此步可以取消)   
此处代码为光照贴图时，现确定场景里所有模型的uv坐标(lightMapUVSet为贴图索引)  
```
{
    unsigned __int32 lightmapSize = 1024;
    float factorMax = 1000.0f;
    float factorMin = 1e-5f;
    float factor = 500.0f;
    float lastFactor = -1.0f;
    bool loop = true;
    while (loop)
    {
        if (DoUVMapping(meshInfo, lightmapSize, factor, seam, lCount, lightMapUVSet))
        {
            lastFactor = factorMax = factor;
            factor = (factorMin + factorMax) * 0.5f;
        }
        else
        {
            if ((factorMax - factorMin) < 1e-4f)
                loop = false;
            else
            {
                factorMin = factor;
                factor = (factorMin + factorMax) * 0.5f;
            }
        }
    }
    return DoUVMapping(meshInfo, lightmapSize, lastFactor, seam, lCount, lightMapUVSet);
}
```

最后的计算处理中，将模型的光照贴图的uv值写在对应的那一套uv集中  