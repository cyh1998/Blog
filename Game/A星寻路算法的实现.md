#### 概 述
在游戏开发中，经常会使用`A*`算法来实现寻路。本文主要介绍使用C++来简单实现`A*`算法，便于理解。

#### 算法原理
网上有关`A*`算法原理的讲解很多，这里就不阐述了。贴一个个人认为讲解比较好的链接：[初学者的 A* 寻路](https://www.gamedev.net/reference/articles/article2003.asp#google_vignette)。链接文章用图文的方式，讲解了算法原理、流程以及寻路算法相关的注意点和优化。

#### 算法实现
根据算法原理，详细的实现步骤如下：
1. 将起点添加到open表中；
2. 从open表中取成本最低的结点，即F(G + H)值的最小结点；
3. 判断此结点是否是终点，若是则结束寻路；
4. 将此结点从open表中移除，添加到close表；
5. 依次判断当前结点各个方位的结点(下面称为搜索结点)：
5.1 若搜索结点超出世界范围或不可达到，则跳过；  
5.2 判断搜索结点是否已存在于open表中，若不存在，添加到open表中；反之，若G值更小，更新结点；
6. 循环处理open表中的结点，直到open表为空，则不存在路径。
7. 根据终点反推起点，生成最终路径

实现代码如下：
```
// AStar.h
#include <vector>

struct Vec2
{
    int x;
    int y;
    bool operator == (const Vec2 &rhs)
    {
        return (x == rhs.x && y == rhs.y);
    };

};

struct Node
{
    Vec2 coordinate_; //坐标
    int G; //起点到该位置的成本
    int H; //该位置到终点的成本，启发式
    Node *parent_; //父结点

    Node(Vec2 coordinate, Node *parent = nullptr)
    {
        coordinate_ = coordinate;
        G = 0;
        H = 0;
        parent_ = parent;
    };

    int GetCost() { return G + H; } //获取总成本
};

using CoordinateList = std::vector<Vec2>;
using NodeArr = std::vector<Node *>;

class AStar
{
public:
    AStar();

    void SetWorldSize(Vec2 size); //设置地图大小
    void SetWalls(CoordinateList walls); //设置墙体
    CoordinateList FindPath(Vec2 source, Vec2 target); //A*寻路

private:
    bool IfCollision(Vec2 coordinate); //判断是否碰撞
    Node* FindNodeInArr(NodeArr &arr, Vec2 coordinate);
    int Manhattan(Vec2 source, Vec2 target); //曼哈顿距离
    void ReleaseNodes(NodeArr& arr);

private:
    Vec2 worldSize_; //地图大小
    int direction_; //搜索方位(4或8)
    CoordinateList directions_; //方向
    CoordinateList walls_; //墙
};

// AStar.cpp
#include "AStar.h"

Vec2 operator + (const Vec2 &lhs, const Vec2 &rhs)
{
    return { lhs.x + rhs.x, lhs.y + rhs.y };
}

AStar::AStar()
{
    worldSize_ = { 0, 0 };
    direction_ = 8;
    directions_ = {
        { 0, 1 }, { 1, 0 }, { 0, -1 }, { -1, 0 },
        { -1, -1 }, { 1, 1 }, { -1, 1 }, { 1, -1 }
    };
    walls_ = {};
}

void AStar::SetWorldSize(Vec2 size)
{
    worldSize_ = size;
}

void AStar::SetWalls(CoordinateList walls)
{
    walls_ = walls;
}

CoordinateList AStar::FindPath(Vec2 source, Vec2 target)
{
    NodeArr openSet; //open表
    NodeArr closedSet; //close表
    Node* nodePtr = nullptr;

    // 添加起点
    openSet.emplace_back(new Node(source));

    while (!openSet.empty()) {
        // 获取open表中总成本最小的结点
        auto node = openSet.begin();
        nodePtr = *node;
        for (auto item = openSet.begin(); item != openSet.end(); ++item) {
            if ((*item)->GetCost() < (*node)->GetCost()) {
                node = item;
                nodePtr = *item;
            }
        }

        // 判断是否到达目标
        if (nodePtr->coordinate_ == target) {
            break;
        }

        // 添加到close表中，从open表中移除
        closedSet.emplace_back(nodePtr);
        openSet.erase(node);

        // 依次判断当前结点的各个方位
        for (int i = 0; i < direction_; ++i) {
            Vec2 findCoordinate(nodePtr->coordinate_ + directions_[i]);

            // 判读是否超出世界范围 是否不可达到(结点位置是墙体)
            if (IfCollision(findCoordinate) || FindNodeInArr(closedSet, findCoordinate)) {
                continue;
            }

            // 判断搜索结点是否已存在于open表中
            int gCost = nodePtr->G + ((i < 4) ? 10 : 14);
            Node* findNode = FindNodeInArr(openSet, findCoordinate);
            if (nullptr == findNode) { //不存在，添加添加新的结点
                findNode = new Node(findCoordinate, nodePtr);
                findNode->G = gCost;
                findNode->H = Manhattan(findNode->coordinate_, target);
                openSet.emplace_back(findNode);
            } else { //存在，判断是否需要更新
                if (gCost < findNode->G) {
                    findNode->parent_ = nodePtr;
                    findNode->G = gCost;
                }
            }
        }
    }

    // 生成最终路径
    CoordinateList path;
    while (nodePtr != nullptr) {
        path.emplace_back(nodePtr->coordinate_);
        nodePtr = nodePtr->parent_;
    }

    // 释放寻路结点
    ReleaseNodes(openSet);
    ReleaseNodes(closedSet);

    return path;
}

bool AStar::IfCollision(Vec2 coordinate)
{
    if (coordinate.x < 0 || coordinate.x >= worldSize_.x ||
        coordinate.y < 0 || coordinate.y >= worldSize_.y ||
        std::find(walls_.begin(), walls_.end(), coordinate) != walls_.end()) {
        return true;
    }
    return false;
}

Node* AStar::FindNodeInArr(NodeArr& arr, Vec2 coordinate)
{
    for (auto node : arr) {
        if (node->coordinate_ == coordinate) {
            return node;
        }
    }
    return nullptr;
}

int AStar::Manhattan(Vec2 source, Vec2 target)
{ 
    return 10 * (abs(source.x - target.x) + abs(source.y - target.y));
}

void AStar::ReleaseNodes(NodeArr& arr)
{
    for (auto i = arr.begin(); i != arr.end();) {
        delete *i;
        i = arr.erase(i);
    }
}
```
**注：** 代码中H值的计算，即启发式，使用的是曼哈顿距离。当然，启发式函数可以使用很多不同的计算方式，这里不一一展示。

**用法示例**
```
#include <iostream>
#include "source/AStar.h"

using namespace std;

int main()
{
    AStar aStar;
    aStar.SetWorldSize({ 7, 7 }); //设置世界大小
    std::vector<Vec2> walls = { { 2, 3 }, { 3, 3 }, { 4, 3 } };
    aStar.SetWalls(walls); //设置墙体
    auto path = aStar.FindPath({ 3, 1 }, { 3, 5 }); //寻路

    for (auto &i : path) {
        std::cout << i.x << " " << i.y << "\n";
    }
}
```
**GitHub地址：**[A-Star](https://github.com/cyh1998/A-Star)
