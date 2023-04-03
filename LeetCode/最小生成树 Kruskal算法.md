## 前 言
求连通图的最小生成树，可以用 `Kruskal(克鲁斯卡尔)` 算法和 `Prim(普里姆)` 算法。本文介绍**Kruskal**算法的思路和实现。

## 思 路
Kruskal算法以**边**为基础，每次从集合中选择最小边，判断该边的两个端点是否属于同一个连通分量：若是，则跳过该边；反之，将两个端点合并连通分量，直到所有端点属于同一个连通分量，算法结束。  
不难看出，我们需要使用[并查集](./并查集.md)。

由于每次选择最小边，所以需要对所有边进行排序，设计如下的结构体，包含两个端点 `x`、`y` 以及权值 `len`，即两点之间的距离
```
struct Edge {
    int len; //权值
    int x, y; //两个端点
    Edge(int len, int x, int y) : len(len), x(x), y(y) {
    }
};
```

## 实 现
根据算法思路，实现代码如下：
```
class Djset { //并查集模板
public:
    vector<int> parent;
    vector<int> rank;
    int count;

    Djset(int n): parent(vector<int>(n)), rank(vector<int>(n)), count(n) {
        for (int i = 0; i < n; i++) parent[i] = i;
    }
    
    int find(int x) {
        if (x != parent[x]) parent[x] = find(parent[x]);
        return parent[x];
    }
    
    bool merge(int x, int y) {
        int rootx = find(x);
        int rooty = find(y);
        if (rootx != rooty) {
            if (rank[rootx] < rank[rooty]) swap(rootx, rooty);
            parent[rooty] = rootx;
            count--;
            if (rank[rootx] == rank[rooty]) rank[rootx] += 1;
            return true;
        } else return false;
    }
};

struct Edge {
    int len; //权值
    int x, y; //两个端点
    Edge(int len, int x, int y) : len(len), x(x), y(y) {
    }
};

class Solution {
public:
    int minCostConnectPoints(vector<vector<int>>& points) {
        int ret = 0;
        int n = points.size();
        Djset ds(n);
        vector<Edge> edges;
        
        // 初始化所有边
        for (int i = 0; i < n; i++) {
            for (int j = i + 1; j < n; j++) {
                edges.emplace_back(abs(points[i][0] - points[j][0]) + abs(points[i][1] - points[j][1]), i, j);
            }
        }

        // 边排序
        sort(edges.begin(), edges.end(), [](auto& a, auto& b) {
            return a.len < b.len;
        });

        for (auto &e : edges) {
           if (ds.merge(e.x, e.y)) ret += e.len;
           if (ds.count == 1) return ret;
        }
        return 0;
    }
};
```
LeetCode题目：[1584. 连接所有点的最小费用](https://leetcode-cn.com/problems/min-cost-to-connect-all-points/)  
更多我的Leetcode题解，详见 [Leetcode题解](https://github.com/cyh1998/algorithm)