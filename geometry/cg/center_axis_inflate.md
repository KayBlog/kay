[<< 返回到上页](../index.md)

**这里将介绍中轴膨胀的博客文章**  

这里所说的中轴膨胀，实质是对中轴提取的逆过程。  

![样例图](images/center_inflate.png)   
上面是已知多条线段，然后求包围这些线段的简单多边形。黑色线条为输入数据，红色多边形即为所求。  

这里只做一个算法的简答介绍，实现代码不展示  
1. 首先对线条做膨胀，可以对每一个线上点的附近的点标识为1  
2. 如此，可以将线段外层可以构成一个填充的多边形区域  
3. 接下来就是寻找边界。可以看成是findContour，在图像算法中很常见，查找轮廓  
4. 轮廓的提取算法稍微复杂些: 根据点的信息求得距离场DistanceField，即最外层的是最小值；求得的距离场后，根据分水岭watershed算法淹没；最后求得的分区，找到区块边界；然后根据区块边界，找到多边形的边界  
5. 找到边界后，做一个最小二乘法拟合  

在后面分析detour和recast库时，再详细介绍这里提到的   
