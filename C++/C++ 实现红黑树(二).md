红黑树的左旋、右旋和插入操作，请移步：[C++ 实现红黑树(一)](./C++%20实现红黑树(一).md)  
### 删除
整体思路：1.将红黑树当作一颗二叉查找树，寻找待删除的结点；2.删除该结点；3.通过"旋转和重新着色"等一系列操作来修正该树，使之重新成为一棵红黑树  
第二步的删除操作分为三种情况：  
1. 待删除结点没有孩子 -> 直接删除
2. 待删除结点有一个孩子 -> 删除结点，并让唯一的孩子结点顶替其位置
3. 待删除结点左右孩子均存在 -> 寻找待删除结点的后继结点，两者互换，即将删除"待删除结点"转化为删除"后继结点"。 在"待删除节点"有两个非空子节点的情况下，其后继结点不可能是双子非空。若没有儿子，则按"情况1 "进行处理；若只有一个儿子，则按"情况2 "进行处理。
```
//删除
template <typename keytype>
void RBTree<keytype>::__remove(Node *&root,Node *node){
    Node *parent,*child;
    //待删除结点的左右孩子都不为空
    if((node->left != nullptr) && (node->right != nullptr)){
        Node *replace = find_successor_node(node);
        node->key = replace->key;
        parent = replace->parent;
        child = replace->right;
        if(child){
            if(parent == node){
                node->right = child;
                child->parent = node;
            }else{
                parent->left = child;
                child->parent = parent;
            }
        }
        if(replace->color == Black) out_correct(root,child);
        delete replace;
        return;
    }
    //待删除结点有一个孩子或没有孩子结点
    if(node->left != nullptr) child = node->left; //待删除结点有左孩子
    else child = node->right; //待删除结点有右孩子
    parent = node->parent;
    if(child) child->parent = parent;
    if(parent){ //待删除结点不是根结点
        if(parent->left == node){
            parent->left = child;
        }else{
            parent->right = child;
        }
    }else{
        root = child;
    }
    if(node->color == Black) out_correct(root,child);
    delete node;
}
```
**删除修正**  
当删除的结点颜色为黑色时，需要对红黑树进行修正操作
针对顶替删除结点的结点，分为三种情况：
1. 顶替结点为红色 -> 直接将顶替结点改为黑色
2. 顶替结点为黑色且为根结点 -> 不做任何操作
3. 顶替结点为黑色且不是根结点 -> 分情况讨论

对于顶替结点为黑色且不是根结点的情况，分为四种可能分析：  
设：顶替结点的兄弟结点为x结点  
(1) x结点为红色  
=> 将顶替结点的兄弟结点设为黑色；  
=> 将顶替结点的父结点设为红色；  
=> 对顶替结点的父结点进行左旋；  
=> 左旋后，重新设置顶替结点的兄弟结点。  

(2) x结点为黑色，x结点的两个孩子结点都是黑色  
=> 将顶替结点的兄弟结点设为红色；  
=> 将顶替结点的父结点设为当前节点。  

(3) x结点为黑色，x结点的左孩子是红色，右孩子是黑色  
=> 将顶替结点的兄弟结点的左孩子设为黑色；  
=> 将顶替结点的兄弟结点设为红色；  
=> 对顶替结点的兄弟结点进行右旋；  
=> 右旋后，重新设置顶替结点的兄弟结点。  

(4) x结点为黑色，x结点的右孩子是红色，左孩子任意颜色  
=> 将顶替结点的父结点颜色赋值给顶替结点的兄弟结点；  
=> 将顶替结点的父结点设为黑色；  
=> 将顶替结点的兄弟结点的右子节设为黑色；  
=> 对顶替结点的父结点进行左旋；  
=> 设置顶替结点为根结点。  

注：以上的四种情况，是针对顶替结点是其父结点左孩子的情况，则兄弟结点就是顶替结点父结点的右孩子。若顶替结点是其父结点右孩子，则兄弟结点就是顶替结点父结点的左孩子，且上面操作中的"right"和"left"对调，依次执行。
```
//删除修正
template <typename keytype>
void RBTree<keytype>::out_correct(Node *&root,Node *node){
    while(node != root && node->color == Black){
        Node *brother;
        Node *parent = node->parent;
        if(parent->left = node){
            brother = parent->right;
            if(brother->color == Red){
                brother->color = Black;
                parent->color = Red;
                left_rotate(root,parent);
                brother = parent->right;
            }
            if((brother->left->color == Black) && (brother->right->color == Black)){
                brother->color = Red;
                node = node->parent;
            }else if(brother->right->color == Black){
                brother->left->color = Black;
                brother->color = Red;
                right_rotate(root,brother);
                brother = parent->right;
            }else{
                brother->color = parent->color;
                parent->color = Black;
                brother->right->color = Black;
                left_rotate(root,parent);
                node = root;
            }
        }else{
            brother = parent->left;
            if(brother->color == Red){
                brother->color = Black;
                parent->color = Red;
                right_rotate(root,parent);
                brother = parent->left;
            }
            if((brother->left->color == Black) && (brother->right->color == Black)){
                brother->color = Red;
                node = node->parent;
            }else if(brother->left->color == Black){
                brother->right->color = Black;
                brother->color = Red;
                left_rotate(root,brother);
                brother = parent->left;
            }else{
                brother->color = parent->color;
                parent->color = Black;
                brother->left->color = Black;
                right_rotate(root,parent);
                node = root;
            }
        }
    }
    node->color = Black;
}
```
### 整体实现代码以及测试
```
#include <iostream>

using namespace std;

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

//内部实现
//插入
template <typename keytype>
void RBTree<keytype>::__insert(Node *&root,const keytype key){
    // cout << key << endl;
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

//查找
template <typename keytype>
RBTNode<keytype> *RBTree<keytype>::__search(Node *&root,const keytype key){
    Node *p = root;
    while(p != nullptr){
        if(key == p->key) return p;
        else if(key > p->key) p = p->right;
        else p = p->left;
    }
    return p;
}

//寻找后继结点
template <typename keytype>
RBTNode<keytype> *RBTree<keytype>::find_successor_node(Node *node){
    Node *p = node->right;
    while (p != nullptr){
        p = p->left;
    }
    return p;
}

//遍历(中序)
template <typename keytype>
void RBTree<keytype>::__traversal(Node *&root){
    if(root == nullptr) return;
    __traversal(root->left);
    // cout << root->key << "(" << root->color << ") ";
    cout << root->key << " ";
    __traversal(root->right);
}

//删除
template <typename keytype>
void RBTree<keytype>::__remove(Node *&root,Node *node){
    Node *parent,*child;
    //待删除结点的左右孩子都不为空
    if((node->left != nullptr) && (node->right != nullptr)){
        Node *replace = find_successor_node(node);
        node->key = replace->key;
        parent = replace->parent;
        child = replace->right;
        if(child){
            if(parent == node){
                node->right = child;
                child->parent = node;
            }else{
                parent->left = child;
                child->parent = parent;
            }
        }
        if(replace->color == Black) out_correct(root,child);
        delete replace;
        return;
    }
    //待删除结点有一个孩子或没有孩子结点
    if(node->left != nullptr) child = node->left; //待删除结点有左孩子
    else child = node->right; //待删除结点有右孩子
    parent = node->parent;
    if(child) child->parent = parent;
    if(parent){ //待删除结点不是根结点
        if(parent->left == node){
            parent->left = child;
        }else{
            parent->right = child;
        }
    }else{
        root = child;
    }
    if(node->color == Black) out_correct(root,child);
    delete node;
}

//删除修正
template <typename keytype>
void RBTree<keytype>::out_correct(Node *&root,Node *node){
    while(node != root && node->color == Black){
        Node *brother;
        Node *parent = node->parent;
        if(parent->left = node){
            brother = parent->right;
            if(brother->color == Red){
                brother->color = Black;
                parent->color = Red;
                left_rotate(root,parent);
                brother = parent->right;
            }
            if((brother->left->color == Black) && (brother->right->color == Black)){
                brother->color = Red;
                node = node->parent;
            }else if(brother->right->color == Black){
                brother->left->color = Black;
                brother->color = Red;
                right_rotate(root,brother);
                brother = parent->right;
            }else{
                brother->color = parent->color;
                parent->color = Black;
                brother->right->color = Black;
                left_rotate(root,parent);
                node = root;
            }
        }else{
            brother = parent->left;
            if(brother->color == Red){
                brother->color = Black;
                parent->color = Red;
                right_rotate(root,parent);
                brother = parent->left;
            }
            if((brother->left->color == Black) && (brother->right->color == Black)){
                brother->color = Red;
                node = node->parent;
            }else if(brother->left->color == Black){
                brother->right->color = Black;
                brother->color = Red;
                left_rotate(root,brother);
                brother = parent->left;
            }else{
                brother->color = parent->color;
                parent->color = Black;
                brother->left->color = Black;
                right_rotate(root,parent);
                node = root;
            }
        }
    }
    node->color = Black;
}

//删除树
template <typename keytype>
void RBTree<keytype>::deleteTree(Node *&root){
    if(root == nullptr) return;
    deleteTree(root->left);
    deleteTree(root->right);
    delete root;
    root = nullptr;
    return;
}

//外部接口
//插入
template <typename keytype>
void RBTree<keytype>::insert(const keytype key){
    __insert(RBTRoot,key);
}

//查找
template <typename keytype>
bool RBTree<keytype>::search(const keytype key){
    return __search(RBTRoot,key) == nullptr ? false : true;
}

//遍历(中序)
template <typename keytype>
void RBTree<keytype>::traversal(){
    __traversal(RBTRoot);
}

//删除
template <typename keytype>
void RBTree<keytype>::remove(const keytype key){
    Node *p = __search(RBTRoot,key);
    if(p != nullptr) __remove(RBTRoot,p);    
}

//析构函数
template <typename keytype>
RBTree<keytype>::~RBTree(){
    deleteTree(RBTRoot);
}

int main(){
    RBTree<int> tree;
    tree.insert(20);
    tree.insert(10);
    tree.insert(50);
    tree.insert(15);
    tree.insert(40);
    tree.traversal();
    cout << endl;
    cout << tree.search(15) << endl;
    tree.remove(15);
    tree.traversal();
    cout << endl;
    return 0;
}
```
运行结果：

![结果](https://upload-images.jianshu.io/upload_images/22192996-55272b4c1a69d119.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

github地址：[R-B-Tree](https://github.com/cyh1998/R-B-Tree)  
参考：  
https://www.cnblogs.com/skywang12345/p/3245399.html
https://www.cnblogs.com/alantu2018/p/8462017.html

