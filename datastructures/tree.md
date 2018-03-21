[<< 返回到主页](index.md)

**这里将介绍树结构的博客文章**  

[1. 二叉树](#1)  
[2. 二叉搜索树](#2)  
[3. 多叉树](#3)  
[4. 四叉树](#4)  
[5. 八叉树](#5)  
[6. kd-tree](#6)  

<span id="1"></span>
## **1. 二叉树**  

**http://blog.csdn.net/IT_DS/article/details/50810498**

<span id="2"></span>
## **2. 二叉搜索树和平衡二叉树**  



<span id="3"></span>
## **3. 多叉树**  

```
template <class TreeType>
class Tree
{
public:
    Tree();
    Tree(TreeType &inputData);
    ~Tree();
    void LevelOrderTraversal(DataStructures::List<Tree*> &output);
    void AddChild(TreeType &newData);
    void DeleteDecendants(void);

    TreeType data;
    DataStructures::List<Tree *> children;
};

template <class TreeType>
Tree<TreeType>::Tree()
{

}

template <class TreeType>
Tree<TreeType>::Tree(TreeType &inputData)
{
    data=inputData;
}

template <class TreeType>
Tree<TreeType>::~Tree()
{
}

template <class TreeType>
void Tree<TreeType>::LevelOrderTraversal(DataStructures::List<Tree*> &output)
{
    unsigned i;
    Tree<TreeType> *node;
    DataStructures::Queue<Tree<TreeType>*> queue;

    for (i=0; i < children.Size(); i++)
        queue.Push(children[i]);

    while (queue.Size())
    {
        node=queue.Pop();
        output.Insert(node);
        for (i=0; i < node->children.Size(); i++)
            queue.Push(node->children[i]);
    }
}

template <class TreeType>
void Tree<TreeType>::AddChild(TreeType &newData)
{
    children.Insert(new Tree(newData));
}

template <class TreeType>
void Tree<TreeType>::DeleteDecendants(void)
{
    DataStructures::List<Tree*> output;
    LevelOrderTraversal(output);
    unsigned i;
    for (i=0; i < output.Size(); i++)
        delete output[i];
}
```

<span id="4"></span>
## **4. 四叉树**  

<span id="5"></span>
## **5. 八叉树**  

<span id="6"></span>
## **6. kd-tree**  