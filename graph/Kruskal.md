[<< 返回到主页](index.md)

**这里将介绍克鲁斯克尔算法**   

这里是介绍最小生成树的Kruskal算法。最小生成树，指在一个图形中(可能是带有环状的图形)找到一颗树，这棵树没有环且边最短。   
Kruskal算法寻找最小生成树，是一个贪心算法，最后的结果并不一定是最优解，其算法性能很快。  
Kruskal算法可以分两步：1. 对所有的边进行从小到大排序  2. 遍历所有的边，这条边若添加后不构成回路就添加，否则跳过这条边。  

算法核心是并查集算法(Union Find)，判断有无环。  

```
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using KayUtils;

namespace KayAlgorithm
{
    public class MSTEdge
    {
        public MSTEdge(int begin, int end, int weight)
        {
            this.Begin = begin;
            this.End = end;
            this.Weight = weight;
        }

        public int Begin { get; private set; }
        public int End { get; private set; }
        public int Weight { get; private set; }

        public override string ToString()
        {
            return string.Format(
              "Begin[{0}], End[{1}], Weight[{2}]",
              Begin, End, Weight);
        }
    }
    public class MSTSubset
    {
        public int Parent { get; set; }
        public int Rank { get; set; }
    }
    // 最小生成树  Kruskal
    public class MSTKruskal
    {
        public static void TestMain()
        {
            MSTKruskal g = new MSTKruskal(9);
            g.AddEdge(0, 1, 4);
            g.AddEdge(0, 7, 8);
            g.AddEdge(1, 2, 8);
            g.AddEdge(1, 7, 11);
            g.AddEdge(2, 3, 7);
            g.AddEdge(2, 5, 4);
            g.AddEdge(8, 2, 2);
            g.AddEdge(3, 4, 9);
            g.AddEdge(3, 5, 14);
            g.AddEdge(5, 4, 10);
            g.AddEdge(6, 5, 2);
            g.AddEdge(8, 6, 6);
            g.AddEdge(7, 6, 1);
            g.AddEdge(7, 8, 7);

            SimpleLogger.Log("Graph Vertex Count : {0}", g.VertexCount);
            SimpleLogger.Log("Graph Edge Count : {0}", g.EdgeCount);
            SimpleLogger.Log("Is there cycle in graph: {0}", g.HasCycle());

            MSTEdge[] mst = g.Kruskal();
            Console.WriteLine("MST Edges:");
            foreach (var edge in mst)
            {
                SimpleLogger.Log("\t{0}", edge);
            }
        }

        private Dictionary<int, List<MSTEdge>> mAdjacentEdges
        = new Dictionary<int, List<MSTEdge>>();

        public MSTKruskal(int vertexCount)
        {
            this.VertexCount = vertexCount;
        }

        public int VertexCount { get; private set; }

        public IEnumerable<int> Vertices { get { return mAdjacentEdges.Keys; } }

        public IEnumerable<MSTEdge> Edges
        {
            get { return mAdjacentEdges.Values.SelectMany(e => e); }
        }

        public int EdgeCount { get { return this.Edges.Count(); } }

        public void AddEdge(int begin, int end, int weight)
        {
            if (!mAdjacentEdges.ContainsKey(begin))
            {
                var edges = new List<MSTEdge>();
                mAdjacentEdges.Add(begin, edges);
            }
            mAdjacentEdges[begin].Add(new MSTEdge(begin, end, weight));
        }
        
        private int Find(MSTSubset[] subsets, int i)
        {
            // 找到根，并将其设为parent
            if (subsets[i].Parent != i)
                subsets[i].Parent = Find(subsets, subsets[i].Parent);
            return subsets[i].Parent;
        }

        // 合并两个分支成一个分支
        private void Union(MSTSubset[] subsets, int x, int y)
        {
            int xroot = Find(subsets, x);
            int yroot = Find(subsets, y);
            // 将小的挂到大的树下
            if (subsets[xroot].Rank < subsets[yroot].Rank)
                subsets[xroot].Parent = yroot;
            else if (subsets[xroot].Rank > subsets[yroot].Rank)
                subsets[yroot].Parent = xroot;
            // 相等，则随便找一个，将rank 加一
            else
            {
                subsets[yroot].Parent = xroot;
                subsets[xroot].Rank++;
            }
        }

        // 可以对最小生成树进行检测
        public bool HasCycle()
        {
            MSTSubset[] subsets = new MSTSubset[VertexCount];
            for (int i = 0; i < subsets.Length; i++)
            {
                subsets[i] = new MSTSubset();
                subsets[i].Parent = i;
                subsets[i].Rank = 0;
            }
            // 这里检测有无回路
            foreach (var edge in this.Edges)
            {
                int x = Find(subsets, edge.Begin);
                int y = Find(subsets, edge.End);
                if (x == y)
                {
                    return true;
                }
                Union(subsets, x, y);
            }
            return false;
        }

        // 先判断有没有回路，有回路再处理
        public MSTEdge[] Kruskal()
        {
            // n个顶点组成无回路且连通的最小生成树，则边是n-1条  
            MSTEdge[] mst = new MSTEdge[VertexCount - 1];
            // 根据权重值递增排序
            var sortedEdges = this.Edges.OrderBy(t => t.Weight);
            var enumerator = sortedEdges.GetEnumerator();
            // 初始化 并查集数据
            // 初始化时，Parent等于下标i值
            MSTSubset[] subsets = new MSTSubset[VertexCount];
            for (int i = 0; i < subsets.Length; i++)
            {
                subsets[i] = new MSTSubset();
                subsets[i].Parent = i;
                subsets[i].Rank = 0;
            }
            int e = 0;
            while (e < VertexCount - 1)
            {
                // 找最小的边做处理
                MSTEdge nextEdge;
                if (enumerator.MoveNext())
                {
                    nextEdge = enumerator.Current;
                    // 对边的两个端点进行查找根
                    int x = Find(subsets, nextEdge.Begin);
                    int y = Find(subsets, nextEdge.End);
                    // 两个根不相等，表明不构成回路，即 x != y
                    if (x != y)
                    {
                        // 添加这条边
                        mst[e++] = nextEdge;
                        // 两端点对应到根合并在一起，构成一个根
                        Union(subsets, x, y);
                    }
                    else
                    {
                        // 对当前边不作处理
                    }
                }
            }
            return mst;
        }
    }
}

```


