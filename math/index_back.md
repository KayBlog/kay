[<< 返回到主页](../index.md)

这里将简单介绍矩阵理论，概率统计，微积分以及凸优化。这里只是一个知识点的梳理，对于机器学习算法，会用到很多数学理论。   

1. 行向量和列向量   
2. 线性组合和线性无关   
3. 线性空间的基  
4. 线性生成空间和空间维度   
5. 向量和向量空间   
6. 向量范数(0,1,2,...,p, 无穷)  
7. 向量内积和投影   
8. 线性映射  
    1. 线性方程组   
    2. 等阶矩阵方程   
    3. 矩阵  
9. 矩阵行列式的值   
10. 矩阵的秩，转置和逆  
11. 矩阵的特征值和特征向量   
12. 矩阵范数  
    1. 正定性   
    2. 齐次性   
    3. 三角不等式   
13. 线性回归问题   
    1. 模型:y=b0 + b1x1 + b2x2 + ... + bnxn + e, e 服从正态分布N(0,sigma)    
    2. 数据：样本(yi,xi1,xi2,...,xin) , 1<= i <= n   
    3. 目的：估计参数bi, 0 <= i <= n      
    4. 最小二乘(l2范数极小化)   
    5. 极大似然估计   
    6. 梯度下降法   

14. 矩阵变换  
    0. 矩阵
        1. 对称矩阵   
            1. 协方差矩阵   
            2. 海森矩阵(牛顿法)   
            3. 内积矩阵   
        2. 正定矩阵  
        3. 其他特殊矩阵
            1. 正交矩阵   
            2. 三角矩阵   
            3. 对角矩阵  
            4. 稀疏矩阵   
    1. 相似变换:存在矩阵P，hat(A) = P-1AP.同一线性变换在不同基下的表达形式   
        1. 不变量：迹    
        2. 不变量：特征值   
    2. 相合变换:存在矩阵P，hat(A) = P'AP.同一内积结构在不同基下的表示形式     
    3. 正交相似变换：P-1 = P'. 

15. 矩阵分解  
    1. LU分解   
    2. Cholesky分解   
    3. QR分解：Q正交矩阵，R上三角矩阵    
        1. Gram-Schmidt正交化   
    4. SVD分解：正交矩阵U和V，奇异值对角矩阵。   
        1. U AA'的特征向量   
        2. V A'A的特征向量   
        3. Sigma A'A或AA'的非零特征值平方根，奇异值    
        4. 伪逆   
        5. 矩阵近似  
16. PCA主成分分析    
    1. 线性降维，通过某种线性投影，将高维的数据映射到低维的空间，并期望所投影的维度上数据方差最大   
    2. 特征向量构成矩阵做线性投影   

17. 重要极限   
18. 导数定义   
19. 导数线性逼近   
20. 求导法则  
21. 泰勒级数   
    1. 欧拉公式    
    2. 向前差分  
    3. 向后差分   
    4. 中心差分   
    5. 二阶中心差分   
    6. 牛顿法   
22. 黎曼积分：求和取极限   
23. 牛顿莱布尼兹公式   
24. 勒贝格积分   
25. 大数定理
    1. 依概率收敛      
26. 中心极限定理   
    1. 样本依分布收敛于标准正态分布     
27. 参数估计问题   
    1. 点估计   
    2. 矩估计   
    3. 极大似然估计  
28. 评判标准   
    1. 相合性   
    2. 无偏性   
    3. 有效性   
    4. 渐近正态性   


29. 凸优化问题：  
    1. 一般形式：
```
最小化： f0(x)
条件： fi(x) <= bi, i =  1, ..., m
```
    2. 条件：fi都是凸函数   
    3. 特点：局部最优等价于全局最优   
    4. 求解：总有现成的工具求解   

30. 凸优化的应用：  
    1. 逼近非凸优化问题，寻找非凸问题的初始值  
    2. 利用对偶问题的凸性给原问题提供下届估计   
    3. 给非凸问题带来启发   

31. 凸集合和凸函数   
    1. 凸组合   
    2. 集合凸包的性质  
    3. 函数凸闭包的性质    
    4. Jensen不等式   
32. 优化问题：最大期望算法EM和混合高斯模型GMM   
33. 共轭函数   
    1. 勒让德变换   
    2. 函数上镜图的支撑超平面的截距   
    3. 
34. 对偶问题   
    1. 拉格朗日对偶函数   
    2. 拉格朗日对偶问题   
    3. 共轭函数与拉格朗日对偶函数   
35. 对偶性    
    1. 弱对偶性与强对偶性   
    2. 强对偶性成立的几种情况   
    3. 凸优化问题求解(KKT)   
36. 支持向量机SVM的算法推导   



