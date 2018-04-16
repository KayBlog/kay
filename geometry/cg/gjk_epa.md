[<< 返回到上级](index.md)

**这里将介绍凸多边形的相交检测和相交深度计算的算法**   

这里先介绍2d的，清楚2d的算法后，扩展到3d其实也很容易理解，以后介绍3d。这里并没有分析SAT算法，以后有空再处理    

**GJK**   
[GJK较详细介绍](http://www.dyn4j.org/2010/04/gjk-gilbert-johnson-keerthi/)   
这里主要是根据dyn4j的源代码实现做的研究  

1. 两个凸性的形体进行Minkowski差和加的结果是凸的  
2. 通过一个方向dir寻找形状最远的点  
3. 方向的选择很重要，需要朝着原点的方向进行  
4. 单纯形的理解  
5. 三角单纯形去除顶点时需要清楚方向的计算  


c#代码的简单处理：  
```
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;

using UnityEngine;

public class Epision
{
    public static double E;
    static Epision()
    {
        double e = 0.5;
        while (1.0 + e > 1.0)
        {
            e *= 0.5;
        }
        E = e;
    }
}

public class GJK
{
    // 根据方向获取最远点
    public static Vector2  Support(List<Vector2> poly1, Vector2 direction)
    {
        double max = Double.MinValue;
        int index = -1;
        for (int i = 0; i != poly1.Count; ++i)
        {
            double dot = Vector2.Dot(poly1[i], direction);
            if (dot > max)
            {
                max = dot;
                index = i;
            }
        }
        return poly1[index];
    }

    // 计算一个 Minkowski 差
    public static Vector2 Minkowski(List<Vector2> poly1, List<Vector2> poly2, Vector2 dir)
    {
        Vector2 v1 = Support(poly1, dir);
        Vector2 v2 = Support(poly2, -dir);
        return v1 - v2;
    }

    // 计算主体部分，迭代
    public static bool Loop(List<Vector2> poly1, List<Vector2> poly2)
    {
        // 初始化
        List<Vector2> simplex = new List<Vector2>();
        Vector2 direction = new Vector2(1.0f, 0.0f);
        Vector2 point = Minkowski(poly1, poly2, direction);
        // 找不到正向的点，不相交
        if (Vector2.Dot(point, direction) <= 0)
        {
            return false;
        }
        simplex.Add(point);
        direction *= -1.0f;
        while (true)
        {
            point = Minkowski(poly1, poly2, direction);      
            if (Vector2.Dot(point, direction) <= 0)
            {
                return false;
            }
            simplex.Add(point);
            if (Evaluate(ref simplex, ref direction))
            {
                return true;
            }
        }
    }

    private static bool Evaluate(ref List<Vector2> simplex, ref Vector2 direction)
    {
        switch (simplex.Count)
        {
            case 2:
                {
                    // 现在还是一条线段
                    // 最后添加的点在1位置
                    // 接下来选择一个方向，且原点在正方向
                    Vector2 ab = simplex[0] - simplex[1];
                    Vector2 ao = -simplex[1];
                    // 做垂直于ba的一个向量
                    direction = Cross(ab, ao, ab);
                    // 原点在直线上 
                    if (direction.sqrMagnitude < Epision.E)
                    {
                        // 法线，顺时针旋转
                        float tmp = direction[0];
                        direction[0] = -direction[1];
                        direction[1] = tmp;
                    }
                }
                return false;
            case 3:
                {
                    // 此时已经是一个三角形
                    // 索引2为最后添加的点
                    // 原点和最后添加的点在在0和1线段的同一侧  
                    Vector2 ac = simplex[0] - simplex[2];
                    Vector2 ab = simplex[1] - simplex[2];
                    Vector2 ao = -simplex[2];

                    // 计算ab和ac的垂直方向，且方向是原理边对应的顶点的  
                    // 每个垂向量，将平面分成两个区域  
                    Vector2 abDir = Cross(ac, ab, ab);
                    Vector2 acDir = Cross(ab, ac, ac);
                    // 然后判断原点在哪一个区域
                    if (Vector2.Dot(abDir, ao) >= 0)
                    {
                        // 原点在abDir侧
                        direction = abDir;
                        simplex.RemoveAt(0);
                        return false;
                    }
                    else 
                    {
                        if (Vector2.Dot(acDir, ao) >= 0)
                        {
                            // 原点在acDir侧
                            direction = acDir;
                            simplex.RemoveAt(1);
                            return false;
                        }
                        return true;
                    }
                    
                }
            default:
                throw new Exception("wrong");
        }
    }


    public static Vector2 Cross(Vector2 a, Vector2 b, Vector2 c)
    {
        // (a×b)×c =b(a·c) - a(b·c)
        // a×(b×c) =b(a·c) - c(a·b),
        /*
            Vector2 r = new Vector2();
            // perform a.dot(c)
            double ac = a.x * c.x + a.y * c.y;
            // perform b.dot(c)
            double bc = b.x * c.x + b.y * c.y;
            // perform b * a.dot(c) - a * b.dot(c)
            r.x = b.x * ac - a.x * bc;
            r.y = b.y * ac - a.y * bc;
         */
        return Vector2.Dot(a, c) * b - Vector2.Dot(b, c) * a;
    }

}

```

**Epa**  
1. 对GJK得到的Simplex结果做进一步处理  
2. 判断三角形是顺时针还是逆时针，为了计算原点在边的左边还是右边，计算一个法向  
3. 将单纯形内的边按照原点到边的距离保存到优先队列里，距离短的先遍历处理  
4. 根据法线进一步求Minkowski差的点，然后计算点到边的距离  
5. 新的距离-边的距离小于一个阈值，则表示找到了相交的深度和法线  
6. 否则，将这个点插入到优先队列，进行下一步处理  
7. 当然，设定一个迭代的最大次数(达到这个值，就退出处理)    

```
void getPenetration(List<Vector2> simplex, MinkowskiSum minkowskiSum, Penetration penetration) {
    // create an expandable simplex
    ExpandingSimplex smplx = new ExpandingSimplex(simplex);
    ExpandingSimplexEdge edge = null;
    Vector2 point = null;
    for (int i = 0; i < this.maxIterations; i++) {
        // get the closest edge to the origin
        edge = smplx.getClosestEdge();
        // get a new support point in the direction of the edge normal
        point = minkowskiSum.getSupportPoint(edge.normal);
        
        // see if the new point is significantly past the edge
        double projection = point.dot(edge.normal);
        if ((projection - edge.distance) < this.distanceEpsilon) {
            // then the new point we just made is not far enough
            // in the direction of n so we can stop now and
            // return n as the direction and the projection
            // as the depth since this is the closest found
            // edge and it cannot increase any more
            penetration.normal = edge.normal;
            penetration.depth = projection;
            return;
        }
        
        // lastly add the point to the simplex
        // this breaks the edge we just found to be closest into two edges
        // from a -> b to a -> newPoint -> b
        smplx.expand(point);
    }
    // if we made it here then we know that we hit the maximum number of iterations
    // this is really a catch all termination case
    // set the normal and depth equal to the last edge we created
    penetration.normal = edge.normal;
    penetration.depth = point.dot(edge.normal);
}
```
其中ExpandingSimplex的代码：  
```
final class ExpandingSimplex {
    /** The winding direction of the simplex */
    private final int winding;
    
    /** The priority queue of simplex edges */
    private final PriorityQueue<ExpandingSimplexEdge> queue;
    
    /**
     * Minimal constructor.
     * @param simplex the starting simplex from GJK
     */
    public ExpandingSimplex(List<Vector2> simplex) {
        // compute the winding
        this.winding = this.getWinding(simplex);
        // build the initial edge queue
        this.queue = new PriorityQueue<ExpandingSimplexEdge>();
        int size = simplex.size();
        for (int i = 0; i < size; i++) {
            // compute j
            int j = i + 1 == size ? 0 : i + 1;
            // get the points that make up the current edge
            Vector2 a = simplex.get(i);
            Vector2 b = simplex.get(j);
            // create the edge
            this.queue.add(new ExpandingSimplexEdge(a, b, this.winding));
        }
    }
    
    /**
     * Returns the winding of the given simplex.
     * <p>
     * Returns -1 if the winding is Clockwise.<br>
     * Returns 1 if the winding is Counter-Clockwise.
     * <p>
     * This method will continue checking all edges until
     * an edge is found whose cross product is less than 
     * or greater than zero.
     * <p>
     * This is used to get the correct edge normal of
     * the simplex.
     * @param simplex the simplex
     * @return int the winding
     */
    protected final int getWinding(List<Vector2> simplex) {
        int size = simplex.size();
        for (int i = 0; i < size; i++) {
            int j = i + 1 == size ? 0 : i + 1;
            Vector2 a = simplex.get(i);
            Vector2 b = simplex.get(j);
            if (a.cross(b) > 0) {
                return 1;
            } else if (a.cross(b) < 0) {
                return -1;
            }
        }
        return 0;
    }
    
    /**
     * Returns the edge on the simplex that is closest to the origin.
     * @return {@link ExpandingSimplexEdge} the closest edge to the origin
     */
    public final ExpandingSimplexEdge getClosestEdge() {
        return this.queue.peek(); // O(1)
    }
    
    /**
     * Expands the simplex by the given point.
     * <p>
     * Removes the closest edge to the origin and adds
     * two new edges using the given point and the removed
     * edge's vertices.
     * @param point the new point
     */
    public final void expand(Vector2 point) {
        // remove the edge we are splitting
        ExpandingSimplexEdge edge = this.queue.poll(); // O(log n)
        // create two new edges
        ExpandingSimplexEdge edge1 = new ExpandingSimplexEdge(edge.point1, point, this.winding);
        ExpandingSimplexEdge edge2 = new ExpandingSimplexEdge(point, edge.point2, this.winding);
        this.queue.add(edge1); // O(log n)
        this.queue.add(edge2); // O(log n)
    }
}
```


MPR(Minkowski Portal Refinement ):  
```
using System.Collections;
using System.Collections.Generic;

    //make the associated dot / cross products easier to read
    public static class Vector3Extensions
    {
        public static float Dot(this Vector3 op1, Vector3 op2)
        {
            return Vector3.Dot(op1, op2);
        }

        public static Vector3 Cross(this Vector3 op1, Vector3 op2)
        {
            return Vector3.Cross(op1, op2);
        }
    }

public static class MPRCollision {

    //points used by the algorithm
    static Vector3 v0;
    static Vector3 v1;
    static Vector3 v2;
    static Vector3 v3;
    static Vector3 v4;

    //if the difference in iteration is less than this,
    //we aren't going to (can't!) converge.
    static float kCollideEpsilon = 1e-3f;

    static Vector3 GetSupport(List<Vector3> shape, Vector3 direction)
    {


        Vector3 point = shape[0];

        for(int i = 1; i < shape.Count; i++)
        {
            //is the current point more "direction" than our result point?
            if ( (shape[i] - point).Dot(direction) > 0 )
            {
                //new result point
                point = shape[i];
            }
        }
        return point;

    }


    public static bool CheckCollide(List<Vector3> shape1, List<Vector3> shape2,
                                    Vector3 center1, Vector3 center2, ref Vector3 penVector)
    {
        //holding variables
        Vector3 n = Vector3.zero;
        Vector3 swap = Vector3.zero;

        // v0 = center of Minkowski sum
        v0 = center2 - center1;

        // Avoid case where centers overlap -- any direction is fine in this case
        if (v0 == Vector3.zero) v0 = new Vector3(0, 0.00001f, 0);

        // v1 = support in direction of origin
        n = -v0;

        //get the differnce of the minkowski sum
        Vector3 v11 = GetSupport(shape1, -n);
        Vector3 v12 = GetSupport(shape2, n);

        v1 = v12 - v11;

        //if the support point is not in the direction of the origin
        if (v1.Dot(n) <= 0)
        {
            //return the direction of the origin, because we're not colliding
            if (penVector != null) penVector = n;
            return false;
        }

        // v2 - support perpendicular to v1,v0
        n = v1.Cross(v0);
        if (n == Vector3.zero)
        {
            //v1 and v0 are parallel, which means 
            //the origin is within the first portal?
            n = v1 - v0;

            if (penVector != null) penVector = n.normalized;
            return true;
        }

        //no early outs yet, so get the new support point
        Vector3 v21 = GetSupport(shape1, -n);
        Vector3 v22 = GetSupport(shape2, n);
        v2 = v22 - v21;

        if (v2.Dot(n) <= 0)
        {
            //can't reach the origin in this direction, ergo, no collision
            if (penVector != null) penVector = n;
            return false;
        }

        // Determine whether origin is on + or - side of plane (v1,v0,v2)
        //tests linesegments v0v1 and v0v2
        n = (v1 - v0).Cross(v2 - v0);
        float dist = n.Dot(v0);

        // If the origin is on the - side of the plane, reverse the direction of the plane
        if (dist > 0)
        {
            //swap the winding order of v1 and v2
            swap = v1;
            v1 = v2;
            v2 = swap;

            //swap the winding order of v11 and v12
            swap = v12;
            v12 = v11;
            v11 = swap;

            //swap the winding order of v11 and v12
            swap = v22;
            v22 = v21;
            v21 = swap;

            //and swap the plane normal
            n = -n;
        }


        ///
        // Phase One: Identify a portal

        while (true)
        {
            // Obtain the support point in a direction perpendicular to the existing plane
            // Note: This point is guaranteed to lie off the plane
            Vector3 v31 = GetSupport(shape1, -n);
            Vector3 v32 = GetSupport(shape2, n); 
            v3 = v32 - v31;

            if (v3.Dot(n) <= 0)
            {
                //can't enclose the origin within our tetrahedron
                if (penVector != null) penVector = n;
                return false;
            }

            // If origin is outside (v1,v0,v3), then eliminate v2 and loop
            if (v1.Cross(v3).Dot(v0) <= 0)
            {
                //failed to enclose the origin, adjust points;
                v2 = v3;
                v21 = v31;
                v22 = v32;
                n = (v1 - v0).Cross(v3 - v0);
                continue;
            }

            // If origin is outside (v3,v0,v2), then eliminate v1 and loop
            if (v3.Cross(v2).Dot(v0) < 0)
            {
                //failed to enclose the origin, adjust points;
                v1 = v3;
                v11 = v31;
                v12 = v32;
                n = (v3 - v0).Cross(v2 - v0);
                continue;
            }


            bool hit = false;

            ///
            // Phase Two: Refine the portal

            int phase2 = 0;

            // We are now inside of a wedge...
            while (phase2 < 20)
            {
                phase2++;

                // Compute normal of the wedge face
                n = (v2 - v1).Cross(v3 - v1);

                n.Normalize();

                // Compute distance from origin to wedge face
                float d = n.Dot(v1);

                // If the origin is inside the wedge, we have a hit
                if (d >= 0 && !hit )
                {
                    if (penVector != null) penVector = n;
                    hit = true;
                }


                // Find the support point in the direction of the wedge face
                Vector3 v41 = GetSupport(shape1, -n);
                Vector3 v42 = GetSupport(shape2, n);
                v4 = v42 - v41;


                float delta = (v4 - v3).Dot(n);
                float separation = -(v4.Dot(n));

                if (delta <= kCollideEpsilon || separation >= 0)
                {
                    //Debug.Log("Non-convergance detected");    
                    if (penVector != null) penVector = n;
                    return hit;
                }

                // Compute the tetrahedron dividing face (v4,v0,v1)
                float d1 = v4.Cross(v1).Dot(v0);

                // Compute the tetrahedron dividing face (v4,v0,v2)
                float d2 = v4.Cross(v2).Dot(v0);

                // Compute the tetrahedron dividing face (v4,v0,v3)
                float d3 = v4.Cross(v3).Dot(v0);

                if (d1 < 0)
                {
                    if (d2 < 0)
                    {
                        // Inside d1 & inside d2 ==> eliminate v1
                        v1 = v4;
                        v11 = v41;
                        v12 = v42;
                    }
                    else
                    {
                        // Inside d1 & outside d2 ==> eliminate v3
                        v3 = v4;
                        v31 = v41;
                        v32 = v42;
                    }
                }
                else
                {
                    if (d3 < 0)
                    {
                        // Outside d1 & inside d3 ==> eliminate v2
                        v2 = v4;
                        v21 = v41;
                        v22 = v42;
                    }
                    else
                    {
                        // Outside d1 & outside d3 ==> eliminate v1
                        v1 = v4;
                        v11 = v41;
                        v12 = v42;
                    }
                }
            }   

            return false;
        }
    }
}
```