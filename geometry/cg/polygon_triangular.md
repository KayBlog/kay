[<< 返回到上页](../index.md)

**这里将介绍多边形的三角划分的博客文章**  

多边形的三角化有多种算法，这里介绍EarClipping(耳切法)，其原理也很简单  
1. 对于给定的简单多边形，判断其方向  
2. 若方向为顺时针，则转换为逆时针方向，接下来以逆时针方向来进行后面的计算  
3. 判断凹凸角： 按照前后顺序规则，找出3个顶点。后面两个顶点相对第一个顶点的向量叉积。若结果大于0，则为凸角；反之为凹角。   
4. 耳切法，最直观的处理：找到一个凸角后，将三个顶点构造成一个三角形，然后去掉这个凸角；后面的计算过程重复上述的处理；直到最后剩下3个顶点。   
5. 对于第4步的计算中，对于凹多边形，寻找到凸角后，不能直接切分。只有满足一个条件：第三条线段不能与其他的线段有交点。   
6. 另外，当存在嵌套的过变形时，需要将多边形合并成一个。合并的原则：奇数层为逆时针，偶数层为顺时针，且最外层为第一层。  
7. 合并的计算：找出相邻两层的一个顶点对，其连线段不予其他线段相交。依次完成对所有多边形的合并。可以先对内层的所有多边形按照与外层的多边形距离排序，然后依次寻找顶点对，依次合并所有顶点对。    

对于空间多边形，可以先将多边形投影到一个平面，构建一个2d的过变形，做三角化。然后根据点的对应关系，重新映射到3d上，可以完成3d多边形的三角化。不过，存在一些限制条件，比如多边形在投影面上不会出现重叠，即为简单多边形。   



```
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using UnityEngine;

using KayMath;
using KayUtils;

namespace KayAlgorithm
{
    public class EarPoint
    {
        public int mIndex;
        public Vector2 mPoint;
        public Vector3 mVertex;

        public EarPoint(int index, Vector3 v)
        {
            mIndex = index;
            mPoint = new Vector2(v[0], v[2]);
            mVertex = v;
        }
        public EarPoint(int index, float x, float y)
        {
            mIndex = index;
            mPoint = new Vector2(x, y);
            mVertex = new Vector3(x, y, 0.0f);
        }
        public float this[int idx]
        {
            get
            {
                return mPoint[idx];
            }
        }

        public override bool Equals(object o)
        {
            return this == (EarPoint)o;
        }

        public override int GetHashCode()
        {
            return string.Format("{0}_{1}", mPoint[0], mPoint[1]).GetHashCode();
        }

        public float Cross(EarPoint rhs)
        { 
            return (mPoint[0] * rhs.mPoint[1]) - (mPoint[1] * rhs.mPoint[0]);
        }
        public float Dot(EarPoint rhs)
        { 
            return Vector2.Dot(mPoint, rhs.mPoint); 
        }
        public float SqrDot(EarPoint rhs)
        { 
            return mPoint[0] * rhs.mPoint[0] + mPoint[1] * rhs.mPoint[1]; 
        }
        public static bool operator == (EarPoint e1, EarPoint e2)
        {
            return e1.mIndex == e2.mIndex;
        }
        public static bool operator !=(EarPoint e1, EarPoint e2)
        {
            return e1.mIndex != e2.mIndex;
        }
    }

    public class EarPolygon
    {
        private LinkedList<EarPoint> mHead;
        private EarPolygon mParent;
        private int mNumberOfPoints;
        private List<EarPolygon> mChildren;
        public List<List<Vector2>> mResults;
        public List<List<Vector3>> mResults3;
        private float mArea; // 带符号
        public EarPolygon()
        {
            mChildren = new List<EarPolygon>();
            mResults = new List<List<Vector2>>();
            mResults3 = new List<List<Vector3>>();
            mHead = new LinkedList<EarPoint>();
            mParent = null;
            mNumberOfPoints = 0;
        }
        public EarPolygon(EarPolygon parent)
        {
            mChildren = new List<EarPolygon>();
            mResults = new List<List<Vector2>>();
            mResults3 = new List<List<Vector3>>();
            mHead = new LinkedList<EarPoint>();
            mParent = parent;
            parent.AddChild(this);
            mNumberOfPoints = 0;
        }
        public bool AddPoint(float x, float y)
        {
            //if (mNumberOfPoints == 0)
            //{
            //    mHead.AddFirst(new EarPoint(mNumberOfPoints++, v));
            //    return true;
            //}
            mHead.AddLast(new EarPoint(mNumberOfPoints++, x, y));
            return true;
        }
        public bool AddPoint(Vector3 v)
        {
            //if (mNumberOfPoints == 0)
            //{
            //    mHead.AddFirst(new EarPoint(mNumberOfPoints++, v));
            //    return true;
            //}
            mHead.AddLast(new EarPoint(mNumberOfPoints++, v));
            return true;
        }
        public LinkedListNode<EarPoint> Get() { return mHead.First; }
        public int NumPoints() { return mHead.Count; }
        public int NumChildren() { return mChildren.Count; }
        public void AddChild(EarPolygon child) { mChildren.Add(child); }
        public void Reverse(int pos)
        {
            if (pos < 0)
            {
                LinkedList<EarPoint> head = new LinkedList<EarPoint>();
                LinkedListNode<EarPoint> last = mHead.Last;
                do
                {
                    head.AddLast(last.Value);
                    last = last.Previous;
                } while (last != null);
                mHead = head;
            }
            else
            {
                mChildren[pos].Reverse(-1);
            }
        }
        public List<EarPolygon> GetChildren() { return mChildren; }
        public EarPolygon this[int idx]
        {
            get
            {
                return mChildren[idx];
            }
        }
        public LinkedListNode<EarPoint> InsertPoint(Vector3 v, LinkedListNode<EarPoint> cur)
        {
            EarPoint newPoint = new EarPoint(mNumberOfPoints++, v);
            return mHead.AddAfter(cur, newPoint);
        }
        public void AddResult(Vector2 v1, Vector2 v2, Vector2 v3)
        {
            mResults.Add(new List<Vector2>());
            mResults[mResults.Count - 1].Add(v1);
            mResults[mResults.Count - 1].Add(v2);
            mResults[mResults.Count - 1].Add(v3);
        }
        public void AddResult(Vector3 v1, Vector3 v2, Vector3 v3)
        {
            mResults3.Add(new List<Vector3>());
            mResults3[mResults3.Count - 1].Add(v1);
            mResults3[mResults3.Count - 1].Add(v2);
            mResults3[mResults3.Count - 1].Add(v3);
        }

        public void CalculateArea()
        {
            mArea = 0.0f;
            LinkedListNode<EarPoint> active = Get();
            LinkedListNode<EarPoint> next;
            for (int i = 0; i < NumPoints(); ++i)
            {
                next = Next(active);
                mArea += (active.Value[0] * next.Value[1] - next.Value[0] * active.Value[1]);
                active = next;
            }
            mArea *= 0.5f;
        }
        public LinkedListNode<EarPoint> Previous(LinkedListNode<EarPoint> cur)
        {
            if (cur.Previous == null)
            {
                return mHead.Last;
            }
            return cur.Previous;
        }
        public LinkedListNode<EarPoint> Next(LinkedListNode<EarPoint> cur)
        {
            if (cur.Next == null)
            {
                return mHead.First;
            }
            return cur.Next;
        }
        public bool IsCCW()
        {
            return mArea > 0;
        }
        public bool Remove(LinkedListNode<EarPoint> point)
        {
            mHead.Remove(point);
            return true;
        }
    }

    public class EarClipping
    {
        // 三角化分的入口函数
        public static void Clip(EarPolygon poly)
        {
            // 将多边形调整为逆时针，且合并多个多边形  
            // 多边形内部也会存在其他的多边形
            // 合并规则：外一层的为逆时针，嵌套的一层为顺时针，如此可以合并多级嵌套多变形   
            // 合并计算关键的一个是找到两个多边形的顶点对：这个顶点对的连线不予其他的线段相交  
            Merge(poly);
            // 对多边形三角化  
            RecordEars(poly);
        }
        public static List<Vector4> Merge(EarPolygon poly)
        {
            //  确保逆时针方向
            OrientatePolygon(poly);
            // 合并多个嵌套多边形为一个多边形
            return MergePolygon(poly);
        }
        private static void OrientatePolygon(EarPolygon poly)
        {
            // 根据面积来判断多边形是逆时针还是顺时针
            poly.CalculateArea();
            if (!poly.IsCCW())
            {
                poly.Reverse(-1);
            }
            // 将内部的多边形调整为顺时针
            for (int i = 0; i < poly.NumChildren(); i++)
            {
                poly[i].CalculateArea();
                if (poly[i].IsCCW())
                {
                    poly[i].Reverse(-1);
                }
            }
        }
        // 合并多个多边形为一个多边形
        private static List<Vector4> MergePolygon(EarPolygon poly)
        {
            List<Vector4> connects = new List<Vector4>();
            if (poly.NumChildren() > 0)
            {
                List<EarPolygon> children = poly.GetChildren();
                List<KeyValuePair<int, float>> order = ChildOrder(children);
                KeyValuePair<LinkedListNode<EarPoint>, LinkedListNode<EarPoint>> connection;
                LinkedListNode<EarPoint> temp;
                for (int i = 0; i < order.Count; i++)
                {
                    connection = GetSplit(poly, children[order[i].Key], order[i].Value);
                    connects.Add(new Vector4(connection.Key.Value[0], connection.Key.Value[1], connection.Value.Value[0], connection.Value.Value[1]));
                    LinkedListNode<EarPoint> newP = poly.InsertPoint(connection.Key.Value.mVertex, connection.Value);
                    temp = connection.Key;
                    do
                    {
                        temp = children[order[i].Key].Next(temp);
                        newP = poly.InsertPoint(temp.Value.mVertex, newP);
                    } while (temp.Value != connection.Key.Value);
                    newP = poly.InsertPoint(connection.Value.Value.mVertex, newP);
                }
            }
            return connects;
        }
        private static bool RecordEars(EarPolygon poly)
        {
            LinkedListNode<EarPoint> active = poly.Get();
            int NumPoints = poly.NumPoints() - 2;
            while (poly.NumPoints() >= 3)
            {
                int num = poly.NumPoints();
                int idx = active.Value.mIndex;
                //do
                //{
                //    if (IsConvex(active, poly))
                //    {
                //        if (IsEar(active, poly))
                //        {
                //            break;
                //        }
                //    }
                //    active = poly.Next(active);
                //} while (idx != active.Value.mIndex);

                List<LinkedListNode<EarPoint>> candidate = new List<LinkedListNode<EarPoint>>();
                do
                {
                    if (IsConvex(active, poly))
                    {
                        if (IsEar(active, poly))
                        {
                            candidate.Add(active);
                        }
                    }
                    active = poly.Next(active);
                } while (idx != active.Value.mIndex);
                List<KeyValuePair<int, float>> temp = new List<KeyValuePair<int, float>>();
                for (int i = 0; i < candidate.Count; ++i)
                {
                    LinkedListNode<EarPoint> ear = candidate[i];
                    float angle = AngleWithUp(ear, poly);
                    temp.Add(new KeyValuePair<int, float>(i, angle));
                }
                temp.Sort((x, y) => { return -x.Value.CompareTo(y.Value); });
                active = candidate[temp[0].Key];
                temp.Clear();
                candidate.Clear();
                poly.AddResult(poly.Previous(active).Value.mPoint, active.Value.mPoint, poly.Next(active).Value.mPoint);
                poly.AddResult(poly.Previous(active).Value.mVertex, active.Value.mVertex, poly.Next(active).Value.mVertex);
                active = poly.Next(active);
                poly.Remove(poly.Previous(active));
                continue;
            }
            return true;
        }
        private static bool IsEar(LinkedListNode<EarPoint> ele, EarPolygon poly)
        {
            LinkedListNode<EarPoint> checkerN1 = poly.Next(ele);
            LinkedListNode<EarPoint> checker = poly.Next(checkerN1);
            while (checker.Value.mIndex != poly.Previous(ele).Value.mIndex)
            {
                if (InTriangle(checker.Value, ele.Value, poly.Next(ele).Value, poly.Previous(ele).Value))
                {
                    return false;
                }
                checker = poly.Next(checker);
            }
            return true;
        }
        private static bool InTriangle(EarPoint pointToCheck, EarPoint earTip, EarPoint earTipPlusOne, EarPoint earTipMinusOne)
        {
            bool isIntriangle = GeoTriangleUtils.IsPointInTriangle2(earTip.mPoint, earTipPlusOne.mPoint, earTipMinusOne.mPoint, ref pointToCheck.mPoint);
            if (isIntriangle)
            {
                if (pointToCheck.mPoint == earTip.mPoint || pointToCheck.mPoint == earTipPlusOne.mPoint || pointToCheck.mPoint == earTipMinusOne.mPoint) // 端点
                {
                    return false;
                }
            }
            return isIntriangle;
        }
        private static bool IsInSegment(LinkedListNode<EarPoint> ele, EarPolygon poly)
        {
            LinkedListNode<EarPoint> a = poly.Previous(ele);
            LinkedListNode<EarPoint> b = ele;
            LinkedListNode<EarPoint> c = poly.Next(ele);
            return GeoSegmentUtils.IsPointInSegment2(a.Value.mPoint, c.Value.mPoint, ref b.Value.mPoint);
        }
        private static bool IsConvex(LinkedListNode<EarPoint> ele, EarPolygon poly)
        {
            LinkedListNode<EarPoint> a = poly.Previous(ele);
            LinkedListNode<EarPoint> b = ele;
            LinkedListNode<EarPoint> c = poly.Next(ele);
            return GeoPolygonUtils.IsConvexAngle(a.Value.mPoint, b.Value.mPoint, c.Value.mPoint);
        }

        private static float AngleWithUp(LinkedListNode<EarPoint> ele, EarPolygon poly)
        {
            LinkedListNode<EarPoint> a = poly.Previous(ele);
            LinkedListNode<EarPoint> b = ele;
            LinkedListNode<EarPoint> c = poly.Next(ele);
            Vector3 ab = a.Value.mVertex - b.Value.mVertex;
            Vector3 cb = c.Value.mVertex - b.Value.mVertex;
            float angle = Vector3.Angle(ab, cb);
            if (angle > 90)
            {
                angle = 180 - angle;
            }
            return angle;
        }

        private static KeyValuePair<LinkedListNode<EarPoint>, LinkedListNode<EarPoint>> GetSplit(EarPolygon outer, EarPolygon inner, float smallestX)
        {
            LinkedListNode<EarPoint> smallest = inner.Get();
            do
            {
                if (smallest.Value[0] == smallestX)
                    break;
                smallest = smallest.Next;
            } while (smallest != null);

            LinkedListNode<EarPoint> closest = GetClosest(OrderPoints(outer, smallest), 0, outer, smallest, inner);
            KeyValuePair<LinkedListNode<EarPoint>, LinkedListNode<EarPoint>> split = new KeyValuePair<LinkedListNode<EarPoint>, LinkedListNode<EarPoint>>(smallest, closest);
            return split;
        }
        class EarNodeValueCompare : IComparer<LinkedListNode<EarPoint>>
        {
            LinkedListNode<EarPoint> mActivePoint;
            public EarNodeValueCompare(LinkedListNode<EarPoint>  activePoint)
            {
                mActivePoint = activePoint;
            }
            public int Compare(LinkedListNode<EarPoint> n1, LinkedListNode<EarPoint>n2)
            {
                Vector2 t1 = n1.Value.mPoint - mActivePoint.Value.mPoint;
                Vector2 t2 = n2.Value.mPoint - mActivePoint.Value.mPoint;
                return t1.sqrMagnitude.CompareTo(t2.sqrMagnitude);
            }
        }
        private static List<LinkedListNode<EarPoint>> OrderPoints(EarPolygon poly, LinkedListNode<EarPoint> point)
        {
            LinkedListNode<EarPoint> head = poly.Get();
            List<LinkedListNode<EarPoint>> pointContainer = new List<LinkedListNode<EarPoint>>();
            LinkedListNode<EarPoint> activePoint = point;
            while (head != null)
            {
                pointContainer.Add(head);
                head = head.Next;
            }
            pointContainer.Sort(new EarNodeValueCompare(activePoint));
            return pointContainer;
        }
        private static List< KeyValuePair< int, float > > ChildOrder(List<EarPolygon> children)
        {
            List<KeyValuePair<int, float>> toSort = new List<KeyValuePair<int, float>>();
            LinkedListNode<EarPoint> head;
            int size = children.Count;
            for (int i = 0; i < size; i++)
            {
                head = children[i].Get();
                float smallestX = head.Value[0];
                do
                {
                    smallestX = head.Next.Value[0] > smallestX ? smallestX : head.Next.Value[0];
                    head = head.Next;
                } while (head.Next != null);
                toSort.Add(new KeyValuePair<int, float>(i, smallestX));
            }
            toSort.Sort((x, y) => x.Value.CompareTo(y.Value));
            return toSort;
        }
        private static LinkedListNode<EarPoint> GetClosest(List<LinkedListNode<EarPoint>> pointsOrdered, int index, EarPolygon poly, LinkedListNode<EarPoint> innerPoint, EarPolygon polychild)
        {
            LinkedListNode<EarPoint> a = innerPoint;
            LinkedListNode<EarPoint> b = pointsOrdered[index];
            LinkedListNode<EarPoint> c = poly.Get();
            bool intersection = false;
            do
            {
                intersection = DoIntersect(a.Value, b.Value, c.Value, poly.Next(c).Value);
                c = c.Next;
            } while ((!intersection) && (c != null));
            if (!intersection)
            {
                c = a;
                do
                {
                    intersection = DoIntersect(a.Value, b.Value, c.Value, polychild.Next(c).Value);
                    c = polychild.Next(c);
                } while ((!intersection) && (c.Value != a.Value));
                if (!intersection)
                {
                    return b;
                }
            }
            return GetClosest(pointsOrdered, index + 1, poly, innerPoint, polychild);
        }
        private static bool DoIntersect(EarPoint a, EarPoint b, EarPoint c, EarPoint d)
        {
            GeoInsectPointInfo insect = new GeoInsectPointInfo();
            bool isInsect = GeoSegmentUtils.IsSegmentInsectSegment2(a.mPoint, b.mPoint, c.mPoint, d.mPoint, ref insect);
            if (isInsect)
            {
                float x = insect.mHitGlobalPoint[0];
                float y = insect.mHitGlobalPoint[1];
                if ((x == c[0] && y == c[1]) || (x == d[0] && y == d[1])) // 端点检测
                {
                    return false;
                }
            }
            return isInsect;
        }
    }
}

```
