# 二叉树
## 概述
满足这样性质的树称为二叉树：空树或节点最多有两个子树，称为左子树、右子树， 左右子树节点同样最多有两个子树。

二叉树是递归定义的，因而常用递归/DFS的思想处理二叉树相关问题。

### 二叉树的定义
```
struct TreeNode {
      int val;
      TreeNode *left;
      TreeNode *right;
      TreeNode(int x) : val(x), left(NULL), right(NULL) {}
};
```

## 递归思想
### 104. 二叉树的最大深度
给定一个二叉树，找出其最大深度。二叉树的深度为根节点到最远叶子节点的最长路径上的节点数。

#### 思路
- 递归求左右子树的各自最大深度，然后取二者最大的，然后加是1（代表本层深度）返回

#### 代码
```
int maxDepth(TreeNode* root) {
    if (!root)
        return 0;
    return max(maxDepth(root->left), maxDepth(root->right)) + 1;
}
```

### 112. 路径总和
给定一个二叉树和一个目标和，判断该树中是否存在根节点到叶子节点的路径，这条路径上所有节点值相加等于目标和。

#### 思路
- 递归左右子树，sum值每次减去当前节点的val
- 注意节点的val有可能是负值，所以一定要递归到叶子节点到底

#### 代码
```
bool hasPathSum(TreeNode* root, int sum) {
    if (root == NULL)
        return false;

    if (root->left == NULL && root->right == NULL && root->val == sum)
        return true;

    return hasPathSum(root->left, sum-root->val) || hasPathSum(root->right, sum-root->val);
}
```

### 437. 路径总和 III
给定一个二叉树，它的每个结点都存放着一个整数值。

找出路径和等于给定数值的路径总数。

路径不需要从根节点开始，也不需要在叶子节点结束，但是路径方向必须是向下的（只能从父节点到子节点）。

二叉树不超过1000个节点，且节点数值范围是 [-1000000,1000000] 的整数。

#### 示例：
```
root = [10,5,-3,3,2,null,11,3,-2,null,1], sum = 8

      10
     /  \
    5   -3
   / \    \
  3   2   11
 / \   \
3  -2   1

返回 3。和等于 8 的路径有:

1.  5 -> 3
2.  5 -> 2 -> 1
3.  -3 -> 11
```

#### 思路
- 双重递归，第一重递归所有节点和sum，第二重递归所有节点所在序列
#### 代码
```
void dfs(TreeNode* root, int sum) {
        if (!root)
            return;

        if (root->val == sum ) {
            count_++;
        }

        dfs(root->left, sum - root->val);
        dfs(root->right, sum - root->val);
    }
    int pathSum(TreeNode* root, int sum) {
        if (!root)
            return 0;

        dfs(root, sum);
        pathSum(root->left, sum);
        pathSum(root->right, sum);
        return count_;    
    }
private:
    int count_;
```

### 543. 二叉树的直径
给定一棵二叉树，你需要计算它的直径长度。一棵二叉树的直径长度是任意两个结点路径长度中的最大值。这条路径可能穿过也可能不穿过根结点。

#### 示例 :
```
给定二叉树

          1
         / \
        2   3
       / \     
      4   5    
返回 3, 它的长度是路径 [4,2,1,3] 或者 [5,2,1,3]
```
#### 思路
- 维护一个计数器，递归计算左右子树的深度，求和后得到一个直径，如果大于计数器，则更新计数器

#### 代码
```
class Solution {
public:
    int diameterOfBinaryTree(TreeNode* root) {
        max_diameter = 0;
        dfs(root);
        return max_diameter;
    }

    int dfs(TreeNode* root) {
        if (!root)
            return 0;
        
        int depth_left = dfs(root->left);
        int depth_right = dfs(root->right);
        int diameter = depth_left + depth_right;
        max_diameter = max(diameter, max_diameter);
        return 1 + max(depth_left, depth_right);

    }
private:
    int max_diameter;
};
```

### 572. 另一个树的子树
给定两个非空二叉树 s 和 t，检验 s 中是否包含和 t 具有相同结构和节点值的子树。

#### 思路
- 双重递归，验证 s 的每个节点，是否和 t 完全一样。

```
class Solution {
public:
    bool isSubtree(TreeNode* s, TreeNode* t) {
        if (!s)
            return false;
        
        return isSametree(s, t) || isSubtree(s->left, t) || isSubtree(s->right, t);
    }

    bool isSametree(TreeNode* s, TreeNode* t) {
        if (s == NULL && t == NULL)
            return true;
        
        if (s == NULL || t == NULL)
            return false;
        
        if (s->val != t->val)
            return false;
        
        return isSametree(s->left, t->left) && isSametree(s->right, t->right);
    }
};
```
