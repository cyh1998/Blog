#### 红黑树的简单介绍
红黑树(Red-Black Tree，简称R-B Tree)，它一种特殊的二叉查找树。这意味着它满足二叉查找树的特征，但是也有许多自己的特性。  
红黑树的特性：  
- 每个节点的颜色非黑及红。
- 根节点是黑色。
- 每个叶子节点是黑色。 [注：此叶子节点，是指为空的叶子节点！]
- 如果一个节点是红色的，则它的子节点必须是黑色的。
- 从一个节点到该节点的子孙节点的所有路径上包含相同数目的黑色节点。

这里推荐一个数据结构可视化的网站，可以查看多种数据结构：[数据结构可视化](https://www.cs.usfca.edu/~galles/visualization/Algorithms.html)
#### 实现
#### 结点和树的定义
```
enum RBTColor{Red,Black};

template <typename keytype>
class RBTNode{
    public:
        RBTColor color;
        keytype key;
        RBTNode *left;
        RBTNode *right;
        RBTNode *parent;
        RBTNode(keytype k,RBTColor c):key(k),color(c),left(nullptr),right(nullptr),parent(nullptr){}
};

template <typename keytype>
class RBTree{
    typedef RBTNode<keytype> Node;

    private:
        Node *RBTRoot;
        void __insert(Node *&root,const keytype key); //插入内部接口
        void in_correct(Node *&root,Node *node); //插入修正接口
        void left_rotate(Node *&root,Node *x); //左旋
        void right_rotate(Node *&root,Node *y); //右旋
        void __traversal(Node *&root); //遍历(中序)内部接口
        void __remove(Node *&root,Node *node); //删除内部接口
        void out_correct(Node *&root,Node *node); //删除修正接口
        Node *__search(Node *&root,const keytype key); //查找内部接口
        Node *find_successor_node(Node *node); //寻找后继结点
        void deleteTree(Node *&root); //删除树

    public:
        RBTree():RBTRoot(nullptr){};
        ~RBTree();
        bool search(const keytype key); //查找外部接口
        void insert(const keytype key); //插入外部接口
        void remove(const keytype key); //删除外部接口
        void traversal(); //遍历(中序)外部接口
};
```
#### 左旋和右旋
详细的注释和代码：
```
/*
 * 左旋示意图(对节点x进行左旋)：
 *       px                              px
 *      /                               /
 *     x                               y                
 *   /  \      ---(左旋)--->          / \  
 *  lx   y                          x   ry     
 *      / \                        / \
 *    ly  ry                     lx  ly  
 *
 */

//左旋
template <typename keytype>
void RBTree<keytype>::left_rotate(Node *&root,Node *x){
    Node *y = x->right; //设x的右孩子为y
    x->right = y->left; //将y的左孩子设为x的右孩子
    if(y->left != nullptr) y->left->parent = x; //将y左孩子的父亲设为x
    y->parent = x->parent; //将y的父亲设为x的父亲
    //情况1：x的父亲为空结点
    if(x->parent == nullptr) root = y; //将y设为根结点
    else{
        //情况2：x是其父亲的左孩子
        if(x->parent->left == x) x->parent->left = y; //将y设为x父结点的左孩子
        //情况3：x是其父亲的右孩子
        else x->parent->right = y; //将y设为x父结点的右孩子
    }
    y->left = x; //将x设为y的左孩子
    x->parent = y; //将y设为x的父结点
}

//右旋与左旋对称
//右旋
template <typename keytype>
void RBTree<keytype>::right_rotate(Node *&root,Node *y){
    Node *x = y->left; //设y的左孩子为x
    y->left = x->right; //将x的左孩子设为y的右孩子
    if(x->right != nullptr) x->right->parent = y; //将x右孩子的父亲设为y
    x->parent = y->parent; //将x的父亲设为y的父亲
    //情况1：y的父亲为空结点
    if(y->parent == nullptr) root = x; //将x设为根结点
    else{
        //情况2：y是其父亲的右孩子
        if(y->parent->right == y) y->parent->right = x; //将x设为y父结点的右孩子
        //情况3：y是其父亲的左孩子
        else y->parent->left = x; //将x设为y父结点的作孩子
    }
    x->right = y; //将y设为x的右孩子
    y->parent = x; //将x设为y的父结点
}
```
#### 插入

1. 将红黑树当作二叉查找树，插入结点
2. 将结点着色为红色
3. 通过着色和旋转操作，使其重新成为一个红黑树
```
//插入
template <typename keytype>
void RBTree<keytype>::__insert(Node *&root,const keytype key){
    Node *node = new Node(key,Black); //新建结点
    //利用指针，非递归插入
    Node *p = nullptr;
    Node *q = root;
    while (q != nullptr){
        p = q;
        if(key < q->key) q = q->left;
        else q = q->right;
    }
    node->parent = p;
    if(p != nullptr){
        if(key < p->key) p->left = node;
        else p->right = node;
    }else root = node;
    node->color = Red;
    in_correct(root,node); //修正红黑树
}
```
这里将第三步细化为修正操作，如下
#### 插入修正
根据插入结点的情况，分为三种情况处理  
1. 插入的结点是根结点：将此结点着色为黑色
2. 插入结点的父结点是黑色的：不做操作
3. 插入结点的父结点是红色的：

针对情况3，进一步细分为三种情况  
(1) 插入结点的父结点是红色的，且叔叔结点(即当前结点的祖父结点的另一个子结点)也是红色  
=>将父结点着色为黑色  
=>将叔叔结点着色为黑色  
=>将祖父结点着色为红色  
=>将祖父结点设为"当前结点"，对"当前结点"继续进行以上的操作  

(2) 当前节点的父结点是红色，叔叔节点是黑色，且当前节点是其父节点的右孩子  
=>将父结点设为”当前节点“  
=>将”当前节点“进行左旋  

(3) 当前节点的父结点是红色，叔叔节点是黑色，且当前节点是其父节点的左孩子  
=>将父结点着色为黑色  
=>将祖父结点着色为红色  
=>将祖父结点进行右旋  

注：以上的三种情况，针对当前节点的父结点是其祖父结点的左孩子，则叔叔结点就是当前结点祖父结点的右孩子。若当前节点的父结点是其祖父结点的右孩子，则上面操作中的"right"和"left"对调，依次执行。

插入修正代码：
```
//插入修正
template <typename keytype>
void RBTree<keytype>::in_correct(Node *&root,Node *node){
    Node *gparent = nullptr;
    while(node->parent != nullptr && node->parent->color == Red){
        gparent = node->parent->parent;
        if(node->parent == gparent->left){
            Node *uncle = gparent->right;
            if(uncle->color == Red){
                node->parent->color = Black;
                uncle->color = Black;
                gparent->color = Red;
                node = gparent;
            }else{
                if(node->parent->left == node){
                    node->parent->color = Black;
                    gparent->color = Red;
                    right_rotate(root,gparent);
                }else{
                    node = node->parent;
                    left_rotate(root,node);
                }
            }
        }else{
            Node *uncle = gparent->left;
            if(uncle->color == Red){
                node->parent->color = Black;
                uncle->color = Black;
                gparent->color = Red;
                node = gparent;
            }else{
                if(node->parent->right == node){
                    node->parent->color = Black;
                    gparent->color = Red;
                    right_rotate(root,gparent);
                }else{
                    node = node->parent;
                    left_rotate(root,node);
                }
            }
        }
    }
    root->color = Black;
}
```
#### 遍历(中序)
```
//遍历(中序)内部接口
template <typename keytype>
void RBTree<keytype>::__traversal(Node *&root){
    if(root == nullptr) return;
    __traversal(root->left);
    cout << root->key << " ";
    __traversal(root->right);
}

//遍历(中序)外部接口
template <typename keytype>
void RBTree<keytype>::traversal(){
    __traversal(RBTRoot);
}
```
#### 测试
```
int main(){
    RBTree<int> tree;
    tree.insert(20);
    tree.insert(10);
    tree.insert(50);
    tree.insert(15);
    tree.insert(40);
    tree.traversal();
    return 0;
}
```
![测试结果](https://upload-images.jianshu.io/upload_images/22192996-6dd9a4080d50ed68.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
