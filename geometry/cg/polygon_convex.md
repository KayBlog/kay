[<< 返回到上页](../index.md)

**这里将介绍多边形的凸分解的博客文章**  

多边形的凸分解对于三角划分而言，这里是分割成多个凸多边形。算法原理很简单：  
1. 首先将多边形三角化  
2. 在三角化的基础上，完成凸分解  
3. 凸分解的过程：将所有的三角面按照共边保存，则可以通过共边找到两个三角面。另外共边被分解成两个线段：指向相反，即每个点都是一个向量边。 通过顶点很容易找到边对应的顶点。比如三角形ABC和ABD，共边AB。则在存储的时候A和B是一个Pair，A的向量为AB，保存在ABC三角形；B的向量为BA，保存在BAD中。则根据AB边，若以A开始，则A边对应的点为C点；A找到B，B对应的点为D点；然后依次类推，知道边界点或者左右寻找点是，构成的两个向量的夹角大于180度。    
4. 上面的过程中，需要明确：这里是根据三角划分得到的三角面，则所有的点都是在简单多边形上；存在两种类型的边：边界边(只有一个三角面包含)和共边(两个三角面包含)。  
5. 第4步是边的特性，对数据初始化时需要保存边的这些信息。  

注意：三角形本身是带有方向的，凸分解时，将共边拆分成两个halfEdge。可以看成所有的三角形都是带方向，且不共边。但是可以通过一个halfEdge找到另一个halfEdge  

```
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;

using UnityEngine;
using KayMath;
using KayUtils;
using KayDatastructure;
using EdgeID = System.Collections.Generic.KeyValuePair<int, int>;

namespace KayAlgorithm
{
    class SharedTriangle
    {
        public int mFaceIndex;
        public int[] mVertexIndex;
        public SharedEdge[] mEdgeList;
        public SharedTriangle(int index, int v1, int v2, int v3)
        {
            mFaceIndex = index;
            mVertexIndex = new int[3]{v1, v2, v3};
            mEdgeList = new SharedEdge[3] { new SharedEdge(mFaceIndex, v1, v2, v3), new SharedEdge(mFaceIndex, v1, v3, v2), new SharedEdge(mFaceIndex, v3, v2, v1) };
        }
    }
    class SharedEdge : ClosestObject
    {
        public int mFaceIndex;
        public int mOtherVertex;
        public Vector2i mEdgeVertexIndex;
        Vector2i mSharedTriangles;
        public SharedEdge(int index, int i, int j, int other)
        {
            mEdgeVertexIndex = Vector2i.GetVector2i(i, j);
            mFaceIndex = index;
            mOtherVertex = other;
        }

        public void SharedFace(int f1, int f2)
        {
            mSharedTriangles = Vector2i.GetVector2i(f1, f2);
        }

        override
        public bool IsCloseTo(ClosestObject other)
        {
            SharedEdge o = (SharedEdge)other;
            return mEdgeVertexIndex == o.mEdgeVertexIndex;
        }
    }

    class HalfEdge
    {
        public int mKey;
        public int mIndex;
        public HalfEdge mNext;
        public HalfEdge mPartner;
    }

    public class ConvexPolygonDecompose
    {
        /// <summary>
        /// 简单多边形或者三角面点数组
        /// </summary>
        /// <param name="tries"></param>
        /// <param name="isPolygon">是否为简单多边形</param>
        /// <param name="vert"></param>
        /// <returns></returns>
        public static List<List<int>> Decompose(List<Vector2> tries, bool isPolygon, ref List<Vector2> vert) 
        {
            if (isPolygon)
            {
                EarPolygon poly = new EarPolygon();
                foreach (Vector2 v in tries)
                {
                    poly.AddPoint(v.x, v.y);
                }
                EarClipping.Clip(poly);
                List<Vector2> triangles = new List<Vector2>();
                GeoUtils.FlatList(poly.mResults, ref triangles);
                List<int> indices = new List<int>();
                GeoUtils.MeshVertexPrimitiveType(triangles, ref vert, ref indices);
                return HMDecompose(vert, indices);
            }
            else
            {
                List<int> indices = new List<int>();
                GeoUtils.MeshVertexPrimitiveType(tries, ref vert, ref indices);
                return HMDecompose(vert, indices);
            }
        }
        /// <summary>
        ///  三角面数组 每个子List为一个三角面
        /// </summary>
        /// <param name="triangleLst"></param>
        /// <param name="vert"></param>
        /// <returns></returns>
        public static List<List<int>> Decompose(List<List<Vector2>> triangleLst, ref List<Vector2> vert)
        {
            List<Vector2> tries = new List<Vector2>();
            GeoUtils.FlatList(triangleLst, ref tries);
            List<int> indices = new List<int>();
            GeoUtils.MeshVertexPrimitiveType(tries, ref vert, ref indices);    
            return HMDecompose(vert, indices);
        }

        public static List<List<int>> Decompose(EarPolygon poly, ref List<Vector2> vert)
        {
            EarClipping.Clip(poly);
            List<Vector2> triangles = new List<Vector2>();
            GeoUtils.FlatList(poly.mResults, ref triangles);
            List<int> indices = new List<int>();
            GeoUtils.MeshVertexPrimitiveType(triangles, ref vert, ref indices);
            return HMDecompose(vert, indices);
        }

        private static List<List<int>> HMDecompose(List<Vector2> vertes, List<int> indices)
        {
            //List<List<Vector2>> triangleDraw = new List<List<Vector2>>();
            //for (int i = 0; i < indices.Count / 3; ++i)
            //{
            //    List<Vector2> tmp = new List<Vector2>();
            //    tmp.Add(vertes[indices[i * 3]]);
            //    tmp.Add(vertes[indices[i * 3 + 1]]);
            //    tmp.Add(vertes[indices[i * 3 + 2]]);
            //    triangleDraw.Add(tmp);
            //}
            //DrawLineUtils.BackPair2[0] = new KeyValuePair<int, System.Drawing.Color>(triangleDraw.Count - 1, DrawLineUtils.BackPair2[0].Value);
            List<HalfEdge> edges = BuilderEdge(indices);
            //foreach (HalfEdge eg in edges)
            //{
            //    List<Vector2> tmp = new List<Vector2>();
            //    tmp.Add(vertes[eg.mIndex]);
            //    tmp.Add(vertes[eg.mNext.mIndex]);
            //    triangleDraw.Add(tmp);
            //}
            //DrawLineUtils.BackPair2[1] = new KeyValuePair<int, System.Drawing.Color>(triangleDraw.Count - 1, DrawLineUtils.BackPair2[1].Value);
            KeyedPriorityQueue<int, HalfEdge, float> remoableEdge = new KeyedPriorityQueue<int, HalfEdge, float>();
            GetRemovableEdgeQueue(vertes, edges, ref remoableEdge);
            //foreach (HalfEdge eg in remoableEdge.Values)
            //{     
            //    if (eg.mIndex < eg.mNext.mIndex)
            //    {
            //        List<Vector2> tmp = new List<Vector2>();
            //        tmp.Add(vertes[eg.mIndex]);
            //        tmp.Add(vertes[eg.mNext.mIndex]);
            //        triangleDraw.Add(tmp);
            //    }
            //}
            //DrawLineUtils.BackPair2[2] = new KeyValuePair<int, System.Drawing.Color>(triangleDraw.Count - 1, DrawLineUtils.BackPair2[2].Value);
            //DrawLineUtils.SavePolygonToFile("decompose.bmp", triangleDraw, DrawLineUtils.BackPair2);
            HashSet<EdgeID> st = DeleteEdges(remoableEdge, vertes);
            return ExtractPolygonList(edges, st);
        }

        private static List<List<int>> ExtractPolygonList(List<HalfEdge> graph, HashSet<EdgeID> deletedEdgeSet)
        {
            List<List<int>> resultList = new List<List<int>>();
            HashSet<HalfEdge> visited = new HashSet<HalfEdge>();
            foreach (HalfEdge edge in graph)
            {
                if (visited.Contains(edge))
                    continue;
                if (deletedEdgeSet.Contains(GetEdgeId(edge)))
                    continue;
                List<int> polygon = new List<int>();
                HalfEdge current = edge;
                do
                {
                    if (!visited.Contains(current))
                    {
                        visited.Add(current);
                        polygon.Add(current.mIndex);
                    }
                    current = current.mNext;
                    while (current.mPartner != null && deletedEdgeSet.Contains(GetEdgeId(current)))
                    {
                        current = current.mPartner.mNext;
                    }
                } while (current.mKey != edge.mKey);
                resultList.Add(polygon);
            }
            return resultList;
        }

        private static HashSet<EdgeID> DeleteEdges(KeyedPriorityQueue<int, HalfEdge, float> priorityQueue, List<Vector2> vertes)
        {
            HashSet<EdgeID> deletedEdgeSet = new HashSet<EdgeID>();
            while (priorityQueue.Count > 0)
            {
                HalfEdge edgeToRemove = priorityQueue.Dequeue();
                deletedEdgeSet.Add(GetEdgeId(edgeToRemove.mIndex, edgeToRemove.mNext.mIndex));
                UpdateEdge(edgeToRemove, priorityQueue, deletedEdgeSet, vertes);
                UpdateEdge(edgeToRemove.mPartner, priorityQueue, deletedEdgeSet, vertes);
            }
            return deletedEdgeSet;
        }
        private static HalfEdge RepresentActive(HalfEdge e)
        {
            if (e.mPartner != null && e.mIndex > e.mPartner.mIndex)
                return e.mPartner;
            else
                return e;
        }
        private static void UpdateEdge(HalfEdge edgeToRemove, KeyedPriorityQueue<int, HalfEdge, float> priorityQueue, HashSet<EdgeID> deletedEdgeSet, List<Vector2> pointList)
        {
            HalfEdge left = GetUndeletedLeft(edgeToRemove, deletedEdgeSet);
            HalfEdge right = GetUndeletedRight(edgeToRemove, deletedEdgeSet);
            HalfEdge reLeft = RepresentActive(left);
            if (priorityQueue.Contain(reLeft.mKey))
            {
                // Check if this is still removable
                HalfEdge leftOfLeft = GetUndeletedLeft(left.mPartner, deletedEdgeSet);
                if ((leftOfLeft.mPartner != null && (leftOfLeft.mPartner.mKey == right.mKey || leftOfLeft.mPartner.mKey == edgeToRemove.mKey)) ||
                    !GeoPolygonUtils.IsConvexAngle(pointList[edgeToRemove.mIndex], pointList[right.mNext.mIndex], pointList[leftOfLeft.mIndex]))
                {
                    priorityQueue.Remove(reLeft.mKey);
                }
                else
                {
                    // Need to update the priority
                    float pri = GetSmallestAdjacentAngleOnEdge(left, deletedEdgeSet, pointList);
                    priorityQueue.Remove(reLeft.mKey);
                    priorityQueue.Enqueue(reLeft.mKey, reLeft, pri);
                }
            }
            HalfEdge reRight = RepresentActive(right);
            if (priorityQueue.Contain(reRight.mKey))
            {
                HalfEdge rightOfRight = GetUndeletedRight(right, deletedEdgeSet);
                if ((rightOfRight.mPartner != null && (rightOfRight.mPartner.mKey == left.mKey || rightOfRight.mKey == edgeToRemove.mKey)) ||
                    !GeoPolygonUtils.IsConvexAngle(pointList[edgeToRemove.mIndex], pointList[rightOfRight.mNext.mIndex], pointList[left.mIndex]))
                {
                    priorityQueue.Remove(reRight.mKey);
                }
                else
                {
                    priorityQueue.Remove(reRight.mKey);
                    priorityQueue.Enqueue(reRight.mKey, reRight, GetSmallestAdjacentAngleOnEdge(right, deletedEdgeSet, pointList));
                }
            }
        }
        private static EdgeID GetEdgeId(int a, int b)
        {
            return (b < a) ? new EdgeID(b, a) : new EdgeID(a, b);
        }

        private static EdgeID GetEdgeId(HalfEdge edge)
        {
            return GetEdgeId(edge.mIndex, edge.mNext.mIndex);
        }

        private static List<HalfEdge> BuilderEdge(List<int> triangleList)
        {
            List<HalfEdge> halfEdgeList = new List<HalfEdge>();
            Dictionary<EdgeID, HalfEdge> openEdgeList = new Dictionary<EdgeID,HalfEdge>();
            int n = triangleList.Count;
            for (int i = 0; i < n; ++i)
            {
                halfEdgeList.Add(new HalfEdge());
            }
            n = n / 3;
            for (int i = 0; i < n; ++i)
            {
                int start = i * 3;
                for (int j = 0; j < 3; ++j)
                {
                    int a = start + j;
                    int b = start + (j + 1) % 3;

                    halfEdgeList[a].mIndex = triangleList[a];
                    halfEdgeList[a].mNext = halfEdgeList[b];
                    halfEdgeList[a].mKey = a;
                    EdgeID edgeID = GetEdgeId(triangleList[a], triangleList[b]);                  
                    if (openEdgeList.ContainsKey(edgeID))
                    {
                        HalfEdge edge = openEdgeList[edgeID];
                        edge.mPartner = halfEdgeList[a];
                        halfEdgeList[a].mPartner = edge;
                        openEdgeList.Remove(edgeID);
                    }
                    else
                    {
                        openEdgeList.Add(edgeID, halfEdgeList[a]);
                        halfEdgeList[a].mPartner = null;
                    }
                }
            }
            return halfEdgeList;
        }
        private static void GetRemovableEdgeQueue(List<Vector2> vertes, List<HalfEdge> edges, ref KeyedPriorityQueue<int, HalfEdge, float> remoableEdge)
        {
            HashSet<EdgeID> deleteSet = new HashSet<EdgeID>();
            foreach (HalfEdge edge in edges)
            {
                if (edge.mIndex > edge.mNext.mIndex)
                    continue;
                if (IsEdgeRemovable(vertes, edge))
                {
                    remoableEdge.Enqueue(edge.mKey, edge, GetSmallestAdjacentAngleOnEdge(edge, deleteSet, vertes));
                }
            }
        }
        private static bool IsEdgeRemovable(List<Vector2> vertes, HalfEdge edge)
        {
            if (edge.mPartner == null)
                return false;
            return GeoPolygonUtils.IsConvexAngle(vertes[edge.mIndex], vertes[edge.mPartner.mNext.mNext.mIndex], vertes[edge.mNext.mNext.mIndex]) &&
                GeoPolygonUtils.IsConvexAngle(vertes[edge.mPartner.mIndex], vertes[edge.mNext.mNext.mIndex], vertes[edge.mPartner.mNext.mNext.mIndex]);
        }
        private static float GetSmallestAdjacentAngleOnEdge(HalfEdge edge, HashSet<EdgeID> deleteSet, List<Vector2> vertes)
        {
            float em = GetSmallestAdjacentAngleOnHalfEdge(edge, deleteSet, vertes);
            float pm = GetSmallestAdjacentAngleOnHalfEdge(edge.mPartner, deleteSet, vertes);
            return Mathf.Max(em, pm);
        }
        private static float GetSmallestAdjacentAngleOnHalfEdge(HalfEdge edge, HashSet<EdgeID> deleteSet, List<Vector2> vertes)
        {
            HalfEdge leftEdge = GetUndeletedLeft(edge, deleteSet);
            HalfEdge rightEdge = GetUndeletedRight(edge, deleteSet);

            Vector2 centerPoint = vertes[edge.mIndex];
            Vector2 forwardPoint = vertes[edge.mNext.mIndex];
            Vector2 leftPoint = vertes[leftEdge.mIndex];
            Vector2 rightPoint = vertes[rightEdge.mNext.mIndex];

            return GetSmallestAdjacentAngleOnHalfEdge(centerPoint, forwardPoint, leftPoint, rightPoint);
        }
        private static HalfEdge GetUndeletedLeft(HalfEdge edge, HashSet<EdgeID> deleteSet)
        {
            HalfEdge edge_right = edge;
            HalfEdge edge_left = edge.mNext.mNext;
            while (deleteSet.Contains(GetEdgeId(edge_left.mIndex, edge_right.mIndex)))
            {
                edge_right = edge_left.mPartner;
                edge_left = edge_right.mNext.mNext;
            }
            return edge_left;
        }
        private static HalfEdge GetUndeletedRight(HalfEdge edge, HashSet<EdgeID> deleteSet)
        {
            do
            {
                edge = edge.mPartner.mNext;
            } while (deleteSet.Contains(GetEdgeId(edge.mIndex, edge.mNext.mIndex)));
            return edge;
        }
        private static float GetSmallestAdjacentAngleOnHalfEdge(Vector2 centerPoint, Vector2 forwardPoint, Vector2 leftPoint, Vector2 rightPoint)
        {
            Vector2 forwardDirection = (forwardPoint-centerPoint).normalized;
            return Mathf.Max(Vector2.Dot((leftPoint - centerPoint).normalized, forwardDirection),
                            Vector2.Dot((rightPoint - centerPoint).normalized, forwardDirection));
        }
    }
}

```
