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

## 真题演练
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
