## 概 述
字典树(Trie)，又叫前缀树，一种树形数据结构。常应用于字符自动补齐、搜索提示等等。本文介绍简单字典树(只包含小写字母)的实现。

## 实 现
**1. 树的节点**  
字典树，类似于多叉树，每个节点需要保存两个内容：  
- `children`：一个大小为 26 的一维数组，分别表示 `a - z` 26个小写字母  
- `isWord`：表示从根节点到当前节点，是否构成一个有效的字符串  

代码如下：
```
class TrieNode {
public:
    vector<TrieNode*> children;
    bool isWord;

    TrieNode() : isWord(false), children(26, nullptr) {
    }

    ~TrieNode() {
        for (auto &i : children) delete i; //释放内存
    }
};
```
**2. 插入操作**  
向字典树中插入一个字符串 `word`。从根节点开始向下寻找与 `word` 的第一个字符匹配的节点，一直匹配到树上没有对应的字符节点，此时创建新的结点，直到插完 `word` 的最后一个字符，并将最后一个结点 `isWord = true;`，表示它是一个有效字符串的末尾。
```
void insert(string word) {
    TrieNode* ptr = root;
    for (char &i : word) {
        if (!ptr->children[i - 'a']) ptr->children[i - 'a'] = new TrieNode();
        ptr = ptr->children[i - 'a'];
    }
    ptr->isWord = true;
}
```
**3. 查找操作**  
在字典树中查找字符串 `word` 是否存在。从根节点的子节点开始向下匹配，若不存在节点，就返回 `false`；如果匹配到了最后一个字符，即返回 `TrieNode->isWord`。
```
bool search(string word) {
    TrieNode* ptr = root;
    for (char &i : word) {
        if (!ptr->children[i - 'a']) return false;
        ptr = ptr->children[i - 'a'];
    }
    return ptr->isWord;
}
```
**4. 前缀匹配**  
判断字典树中是否有以 `prefix` 为前缀的字符串。和查找操作类似，只是当匹配到了最后一个字符，直接返回 `true`，因为一定存在以 `prefix` 为前缀的字符串。
```
bool startsWith(string prefix) {
    TrieNode* ptr = root;
    for (char &i : prefix) {
        if (!ptr->children[i - 'a']) return false;
        ptr = ptr->children[i - 'a'];
    }
    return true;
}
```
完整代码如下：
```
class TrieNode {
public:
    vector<TrieNode*> children;
    bool isWord;

    TrieNode() : isWord(false), children(26, nullptr) {
    }

    ~TrieNode() {
        for (auto &i : children) delete i;
    }
};

class Trie {
public:
    Trie() {
        root = new TrieNode();
    }
    
    void insert(string word) {
        TrieNode* ptr = root;
        for (char &i : word) {
            if (!ptr->children[i - 'a']) ptr->children[i - 'a'] = new TrieNode();
            ptr = ptr->children[i - 'a'];
        }
        ptr->isWord = true;
    }
    
    bool search(string word) {
        TrieNode* ptr = root;
        for (char &i : word) {
            if (!ptr->children[i - 'a']) return false;
            ptr = ptr->children[i - 'a'];
        }
        return ptr->isWord;
    }
    
    bool startsWith(string prefix) {
        TrieNode* ptr = root;
        for (char &i : prefix) {
            if (!ptr->children[i - 'a']) return false;
            ptr = ptr->children[i - 'a'];
        }
        return true;
    }

private:
    TrieNode* root;
};
```
LeetCode题目：[实现 Trie (前缀树)](https://leetcode-cn.com/problems/implement-trie-prefix-tree/)