[<< 返回到主页](index.md)

**这里将介绍Unity3D游戏资源管理的技术**  

每一份资源都作为单独的Assetbundle，记录每一个Assetbundle的依赖关系。则可以构建一个拓扑图形。后面再加载gameobject时，如果内存没有则先加载依赖，否则不需要加载。当然卸载也是如此，需要卸载依赖。   

这里涉及到多个物体依赖同一个物体的情形，则引入引用计数规则，对每一个资源做计数，直到计数为0，则卸载物理内存。  

另外，理解一张图是很重要的：   
![Unity 资源管理](images/assetbundle.jpg)   

转载一篇博文：[链接](https://blog.csdn.net/poem_of_sunshine/article/details/46986373)   


