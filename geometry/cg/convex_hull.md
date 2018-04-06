[<< 返回到上页](../index.md)

**这里将介绍凸包和凸多边形的生成算法**  

**2d 散点集创建凸多边形**  
主要根据Jarves算法。算法原理很简单：   
1. 找出散点的最低点P(且x值最大)和确定一个寻找其他凸多边形的一个方向，初始值为(0, 1)向右的方向  
2. 最低点肯定是凸多边形的一个顶点，确定初始方向后，遍历其他的散点相对P点的向量与方向做点积，寻找夹角最小的点。注意：若存在至少两个点的夹角最小，则选择最远的那个顶点。   
3. 寻找到下一个顶点后，与上一个顶点确定了一个新的方向，然后重复第2步的计算，直到最后的点为初始点则结束。   
4. 将上面遍历到的点，按照顺序构成凸多边形即为所求  

```
using UnityEngine;

namespace KayAlgorithm
{
    // 对每个点做一次封装
    // 记录点和对应散点集的索引值
    public class JarvisElement
    {
        public Vector2 mPoint;
        public int mIndex;
        public JarvisElement(int i, Vector2 p)
        {
            mPoint = p;
            mIndex = i;
        }
        // 计算方向
        public Vector2 Direction(JarvisElement other)
        {
            Vector2 v = mPoint - other.mPoint;
            return v.normalized;
        }
        // 计算夹角。此处可以不求角度值，用cos值表示即可
        public float AngleTo(JarvisElement other, Vector2 dir)
        {
            Vector2 v = Direction(other);
            float dot = Vector2.Dot(v, dir);
            return Mathf.Acos(dot);
        }
        // 判断距离，针对夹角相等时，选择最远的那一个点
        public float DistanceTo(JarvisElement other)
        {
            Vector2 v = mPoint - other.mPoint;
            return v.magnitude;
        }
    }

    public class JarvisConvex
    {
        // 保存所有的点的数据
        private List<JarvisElement> mElements = new List<JarvisElement>();
        // 存储索引值
        public List<int> mResultIndex = new List<int>();
        public JarvisConvex()
        {

        }
        // 创建凸多边形，参数为散点集和一个缩放系数(若点很紧密，可以放大后再做处理。避免误差)
        public static List<Vector2> BuildHull(List<Vector2> points, float scale = 10)
        {
            JarvisConvex jarvis = new JarvisConvex();
            foreach (Vector2 p in points)
            {
                jarvis.AddPoint(p * scale);
            }
            return jarvis.Calculate(scale);
        }
        public static List<Vector2> BuildHullIndex(List<Vector2> points, out List<int> resultIndex, float scale = 10)
        {
            JarvisConvex jarvis = new JarvisConvex();
            foreach (Vector2 p in points)
            {
                jarvis.AddPoint(p * scale);
            }
            var ps = jarvis.Calculate(scale);
            resultIndex = jarvis.mResultIndex;
            return ps;
        }

        public void AddPoint(Vector2 p)
        {
            int index = mElements.Count;
            mElements.Add(new JarvisElement(index, new Vector2(p.x, p.y)));
        }
        // 函数算法核心的计算
        public List<Vector2> Calculate(float scale)
        {
            // 寻找第一个点
            int first = FindFirstPoint();
            // 距离第一个索引
            mResultIndex.Add(first);
            // 根据初始点和方向，寻找下一个点
            // 初始化第二个顶点
            FindNextPoint(new Vector2(1.0f, 0.0f), ref mResultIndex);
            List<Vector2> hull = new List<Vector2>();
            for (int i = 0; i < mResultIndex.Count; ++i)
            {
                hull.Add(mElements[mResultIndex[i]].mPoint * 1.0f / scale);
            }
            return hull;
        }
        // 寻找第一个点
        private int FindFirstPoint()
        {
            float min = 1e10f;
            int index = -1;
            // 寻找y值做小的点
            // 若存在多个最小值，则选择x值最大的哪一个点  
            for (int i = 0; i < mElements.Count; ++i)
            {
                if (mElements[i].mPoint[1] < min)
                {
                    min = mElements[i].mPoint[1];
                    index = i;
                }
                else 
                {
                    if (mElements[i].mPoint[1] == min)
                    {
                        if (mElements[i].mPoint[0] > mElements[index].mPoint[0])
                        {
                            index = i;
                        }
                    }
                }
            }
            return index;
        }
        // 寻找下一个点
        private void FindNextPoint(Vector2 dir, ref List<int> result)
        {
            // 获取最近一次寻找到的点
            int index = result[result.Count - 1];
            float minAngle = 1e10f;
            int minIndex = -1;
            float maxDistacne = -1e10f;
            for (int i = 0; i < mElements.Count; ++i)
            {
                if (i != index)
                {
                    // 每一个元素，相对index的向量，求与dir的夹角
                    // 获得最小的夹角的点
                    float angle = mElements[i].AngleTo(mElements[index], dir);
                    if (angle < minAngle)
                    {
                        minAngle = angle;
                        minIndex = i;
                        maxDistacne = mElements[i].DistanceTo(mElements[index]);
                    }
                    else if (angle == minAngle)
                    {
                        // 夹角相等时，选择距离最长的点
                        float dist = mElements[i].DistanceTo(mElements[index]);
                        if (dist > maxDistacne)
                        {
                            maxDistacne = dist;
                            minIndex = i;
                        }
                    }
                }
            }
            if (minIndex == result[0])
            {
                return;
            }
            else
            {
                result.Add(minIndex);
                try
                {
                    FindNextPoint(mElements[minIndex].Direction(mElements[index]), ref result);
                }
                catch (StackOverflowException e)
                {
                    int i = 0;
                }
                
            }
        }

    }
}
```
2d凸包原理和实现都较容易  

**3d 凸包创建**

使用QuickHull算法来创建凸包。模型的碰撞体可以使用这个算法来创建，凸包的性质很好。凸包碰撞检测时，使用GJK和EPA来计算是否碰撞以及碰撞的深度和法向  

QuickHull的算法原理：  
1. 选择第一个平面将3d散点集分成两部分，一部分保存在正面，另一部分保存在背面  
2. 针对每一个平面和包含的散点集，选择最远点，然后构建3个平面，将点再分到3个平面的正面。其中处理三个面的背面的散点剔除。然后将基础平面去除。  
3. 对于上面的分割过程，直到找不到最远点(不包含面上的点)。  
4. 将所有生成的平面构成需要的凸包   

注意：寻找多个三角面的边界的计算方法  

```
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using KayMath;
using UnityEngine;
namespace KayAlgorithm
{
    // 三角面的边的封装
    public class QHullEdgeBounding
    {
        // 保存边
        public List<GeoSegment3> mEdgeStack;
        public QHullEdgeBounding()
        {
            mEdgeStack = new List<GeoSegment3>();
        }

        public GeoSegment3 Pop()
        {
            GeoSegment3 first = mEdgeStack[0];
            mEdgeStack.RemoveAt(0);
            return first;
        }

        public bool IsEmpty()
        {
            return mEdgeStack.Count == 0;
        }

        public void Push(Vector3 p1, Vector3 p2)
        {
            Push(new GeoSegment3(p1, p2));
        }
        // 这步很关键
        // 若边不存在，则保存；若已经存在，则删除
        // 对于很多个三角面，可以利用此方法寻找边界
        public void Push(GeoSegment3 edge)
        {
            int index = mEdgeStack.FindIndex(delegate(GeoSegment3 p) { return p.Equal(edge); });
            if (index == -1)
                mEdgeStack.Add(edge);
            else
                mEdgeStack.RemoveAt(index);
        }
    }

    public class QHullTrianglePlane
    {
        public GeoPlane mPlane;
        public Vector3[] mVertexPoints;
        
        public QHullTrianglePlane(Vector3 a, Vector3 b, Vector3 c)
        {
            Vector3 normal = Vector3.Cross(b - a, c - a);
            normal.Normalize();
            float d = -Vector3.Dot(normal, a);
            mPlane = new GeoPlane(normal, d);
            mVertexPoints = new Vector3[3];
            mVertexPoints[0] = a;
            mVertexPoints[1] = b;
            mVertexPoints[2] = c;
        }

        public QHullTrianglePlane(Vector3 a, Vector3 b)
        {
            Vector3 normal = Vector3.Cross(b - a, Vector3.forward);
            normal.Normalize();
            float d = -Vector3.Dot(normal, a);
            mPlane = new GeoPlane(normal, d);
            mVertexPoints = new Vector3[3];
            mVertexPoints[0] = a;
            mVertexPoints[1] = b;
            mVertexPoints[2] = Vector3.zero;
        }
        // 判断一个点是否在平面正向，且求出这个点到平面的距离
        public bool Inside(Vector3 p, ref float distance)
        {
            if (IsNotFacePoints(p))
            {
                // 计算距离
                float dist = GeoPlaneUtils.PointDistanceToPlane(mPlane, p);
                if (dist < 0)
                    return false;
                distance = dist;
                return true;
            }
            return false;
        }
        // 判断点是否在平面上
        private bool IsNotFacePoints(Vector3 p)
        {
            return !GeoPlaneUtils.IsPointOnPlane(mPlane.mNormal, mPlane.mD, p);
        }

    }
    // 三角面和包含的点集
    public class QHullTrianglePlanePoints
    {
        // 三角面的定义：3个顶点，顺序不能错  
        public QHullTrianglePlane mTrianglePlane;
        // 在这个平面正方向上所有的点集
        public List<Vector3> mPointsList;
        // 若这个平面还能够找到最远点，则在下一步分割的时候，这个平面是需要被删除的  
        public bool mIsDelete;
        // 最远点记录
        public Vector3 mFathestPoint;
        // 记录最远距离值  
        public float mFathestDistance;
        // 通过两个点来初始化平面，且第三个点默认为原点
        public QHullTrianglePlanePoints(Vector3 a, Vector3 b)
        {
            mPointsList = new List<Vector3>();
            mTrianglePlane = new QHullTrianglePlane(a, b);
            mFathestPoint = Vector3.zero;
            mFathestDistance = -1e20f;
            mIsDelete = false;
        }
        // 通过三个点构建
        public QHullTrianglePlanePoints(Vector3 a, Vector3 b, Vector3 c)
        {
            mPointsList = new List<Vector3>();
            mTrianglePlane = new QHullTrianglePlane(a, b, c);
            mFathestPoint = Vector3.zero;
            mFathestDistance = -1e20f;
            mIsDelete = false;
        }
        // 平面添加点的过程
        public bool AddPoint(Vector3 p)
        {
            float distance = 0;
            // 判断点是否在平面正面
            // 并计算点到平面的距离
            bool isInside = mTrianglePlane.Inside(p, ref distance);
            if (isInside)
            {
                // 添加点
                mPointsList.Add(p);
                // 保存最远距离和最远的点
                if (distance > mFathestDistance)
                {
                    mFathestDistance = distance;
                    mFathestPoint = p;
                }
            }
            return isInside;
        }
        // 判断一个顶点是否在平面的正面
        public bool Inside(Vector3 p)
        {
            float distance = 0;
            return mTrianglePlane.Inside(p, ref distance);
        }

        public bool IsEmpty()
        {
            return mPointsList.Count == 0;
        }
    }

    public class QuickHull
    {
        public static List<Vector3> BuildHull(List<Vector3> points)
        {
            return BuildHull(new GeoPointsArray3(points));
        }

        public static List<Vector3> BuildHull(GeoPointsArray3 points)
        {
            QuickHull qHull = new QuickHull(points);
            qHull.BuildHull();
            return qHull.GetTriangles();
        }
        // 散点集
        private GeoPointsArray3 mPoints;
        // 顶点数量
        private int mVertexCount;
        // 保存创建的平面列表
        List<QHullTrianglePlanePoints> mTrianglePlanePoints;
        public QuickHull(GeoPointsArray3 points)
        {
            mPoints = points;    
            mPoints.Distinct();
            mVertexCount = mPoints.Size();
            mTrianglePlanePoints = new List<QHullTrianglePlanePoints>();
        }
        // 数据初始化后，电泳此函数来构建凸包
        public void BuildHull()
        {
            if (mPoints.Size() < 4)
            {
                return;
            }
            // 构建凸包
            Build();
            // 去掉被删除的平面，剩下的平面就是凸包的平面
            CleanTrianglePlaneList();
        }
        // 去除被删除的平面
        private void CleanTrianglePlaneList()
        {
            int size = mTrianglePlanePoints.Count;
            List<QHullTrianglePlanePoints> trianglePlanePoints = new List<QHullTrianglePlanePoints>();
            for (int i = 0; i < size; ++i)
            {
                if (!mTrianglePlanePoints[i].mIsDelete)
                {
                    trianglePlanePoints.Add(mTrianglePlanePoints[i]);
                }
            }
            mTrianglePlanePoints = trianglePlanePoints;
        }
        // 将凸包的三角面存放到一个列表里
        public List<Vector3> GetTriangles()
        {
            List<Vector3> points = new List<Vector3>();
            int triangleCount = mTrianglePlanePoints.Count;
            for (int i = 0; i < triangleCount; ++i)
            {
                QHullTrianglePlane plane = mTrianglePlanePoints[i].mTrianglePlane;
                points.AddRange(plane.mVertexPoints);
            }
            return points;
        }
        // 将凸包按照顶点索引存放
        private void CalcHullVertexes()
        {
            int triangleCount = mTrianglePlanePoints.Count;
            List<Vector3> vertes = new List<Vector3>();
            List<int> indices = new List<int>();           
            for (int i = 0; i < triangleCount; ++i)
            {
                QHullTrianglePlane plane = mTrianglePlanePoints[i].mTrianglePlane;
                for (int j = 0; j < 3; ++j)
                {
                    int index = vertes.FindIndex(delegate(Vector3 v) { return v == plane.mVertexPoints[j]; });
                    if (index == -1)
                    {
                        indices.Add(vertes.Count);
                        vertes.Add(plane.mVertexPoints[j]);
                    }
                    else
                    {
                        indices.Add(index);
                    }
                }
            }
        }

        private void Build()
        {
            Vector3 min;
            Vector3 max;
            Vector3 fathestPoint;
            // 点集中，找到相距最远的两个点最为初始三角平面的两个顶点
            // 根据两个点和原点可以确定一个平面，然后根据这个初始平面选择一个最远的点
            FindOnce(out min, out max, out fathestPoint);
            // 找到3个点，接下来确定两个平面(一个正面和一个反面，主要是法线方向想法)
            QHullTrianglePlanePoints facePositive = new QHullTrianglePlanePoints(min, max, fathestPoint);
            QHullTrianglePlanePoints faceNegative = new QHullTrianglePlanePoints(min, fathestPoint, max);
            // 将散点集按照点在面的正方向，分组到两个平面上
            SaveOnce(facePositive, faceNegative);
            // 将找到的两个平面先保存
            mTrianglePlanePoints.Add(facePositive);
            mTrianglePlanePoints.Add(faceNegative);
            // 接下来遍历所有的平面，创建新的平面，直到结束操作
            int step = 0;
            while (step < mTrianglePlanePoints.Count)
            {
                // 平面处理
                QHullTrianglePlanePoints trianglePlanePointsLoop1 = mTrianglePlanePoints[step++];
                // 平面若被删除或者平面找不到点，则不作处理
                if (trianglePlanePointsLoop1.mIsDelete || trianglePlanePointsLoop1.IsEmpty())
                {
                    continue;
                }

                fathestPoint = trianglePlanePointsLoop1.mFathestPoint;
                int size = mTrianglePlanePoints.Count;
                QHullEdgeBounding edgeStack = new QHullEdgeBounding();
                List<Vector3> tempPointsList = new List<Vector3>();
                // 对所有的面做一个处理
                // 处理所有面：最远点在面的正方向
                // 这个面需要被删除，保存这些面包含的顶点集合边
                for (int i = 0; i < size; ++i)
                {
                    QHullTrianglePlanePoints trianglePlanePointsLoop2 = mTrianglePlanePoints[i];
                    if (!trianglePlanePointsLoop2.mIsDelete && trianglePlanePointsLoop2.Inside(fathestPoint))
                    {
                        // 最远点存在的情况，这个面需要删除
                        trianglePlanePointsLoop2.mIsDelete = true;
                        // 保存面的三条边，后面的处理时每条边会与最远点构造成一个面
                        Vector3[] planePoints = trianglePlanePointsLoop2.mTrianglePlane.mVertexPoints;
                        // 这里保存边的时候，保留出现一次的，去掉出现两次的边  
                        // 这里保存三角面网格的边界
                        edgeStack.Push(planePoints[0], planePoints[1]);
                        edgeStack.Push(planePoints[1], planePoints[2]);
                        edgeStack.Push(planePoints[2], planePoints[0]);
                        // 保存顶点
                        if (!trianglePlanePointsLoop2.IsEmpty())
                        {
                            int tempSize = trianglePlanePointsLoop2.mPointsList.Count;
                            for (int j = 0; j < tempSize; ++j)
                            {
                                tempPointsList.Add(trianglePlanePointsLoop2.mPointsList[j]);
                            }
                            trianglePlanePointsLoop2.mPointsList.Clear();
                        }
                    }
                }
                while (!edgeStack.IsEmpty())
                {
                    // 取出一条边
                    GeoSegment3 edge = edgeStack.Pop();
                    // 与最远点构成一个三角面
                    QHullTrianglePlanePoints newTrianglePlanePs = new QHullTrianglePlanePoints(edge.mP1, edge.mP2, fathestPoint);
                    List<Vector3> tempPointsList2 = new List<Vector3>();
                    // 最保存的顶点做一个处理
                    // 在面正向的点保存到面中
                    for (int k = 0; k < tempPointsList.Count; ++k)
                    {
                        Vector3 point = tempPointsList[k];
                        if (!newTrianglePlanePs.AddPoint(point))
                        {
                            tempPointsList2.Add(point);
                        }
                    }
                    // 将新建的三角面保存
                    mTrianglePlanePoints.Add(newTrianglePlanePs);
                    // 重新设置点集合  这里不包含已经被分的点集
                    tempPointsList = tempPointsList2;
                }
                // 剩下的点集是不在任和平面的正向的，直接剔除
                tempPointsList.Clear();
            }
        }
        // 寻找初始化的三个点
        private void FindOnce(out Vector3 min, out Vector3 max, out Vector3 fathestPoint)
        {
            int minI = -1;
            int maxI = -1;
            FindGappestTwoPointsOnce(ref minI, ref maxI);
            min = mPoints[minI];
            max = mPoints[maxI];
            // parallel z axis
            QHullTrianglePlanePoints trianglePlanePoints = new QHullTrianglePlanePoints(min, max);
            FindFathestPointToHalfSpaceOnce(trianglePlanePoints);
            fathestPoint = trianglePlanePoints.mFathestPoint;
        }
        
        // 寻找最远点，初始化时使用
        private void FindFathestPointToHalfSpaceOnce(QHullTrianglePlanePoints plane)
        {
            for (int i = 0; i < mVertexCount; ++i)
            {
                plane.AddPoint(mPoints[i]);
            }
        }

        private void FindGappestTwoPointsOnce(ref int min, ref int max)
        {
            float xMin = 1e10f;
            int minIndex = -1;
            float xMax = -1e10f;
            int maxIndex = -1;
            for (int index = 0; index < mVertexCount; ++index)
            {
                float temp = mPoints[index][0];
                if (temp < xMin)
                {
                    xMin = temp;
                    minIndex = index;
                }
                if (temp > xMax)
                {
                    xMax = temp;
                    maxIndex = index;
                }
            }
            min = minIndex;
            max = maxIndex;
        }

        private void SaveOnce(QHullTrianglePlanePoints facePositive, QHullTrianglePlanePoints faceNegative)
        {
            for (int i = 0; i < mVertexCount; ++i)
            {
                if (!facePositive.AddPoint(mPoints[i]))
                {
                    faceNegative.AddPoint(mPoints[i]);
                }
            }
        }
    }
}

```
