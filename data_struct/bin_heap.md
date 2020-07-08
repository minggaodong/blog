# 二叉堆详解实现优先级队列

二叉堆（Binary Heap）性质比二叉搜索树 BST 简单，就是父节点 >= 或 <= 两个子节点，其主要操作就两个，`sink`（下沉）和 `swim`（上浮），用以维护二叉堆的性质。

其主要应用有两个，首先是一种排序方法「堆排序」，第二是一种很有用的数据结构「优先级队列」。

### 二叉堆概述

二叉堆其实就是一种特殊的二叉树（完全二叉树），只不过存储在数组里。一般的链表二叉树，我们操作节点的指针，而在数组里，我们把数组索引作为指针：

假设当前节点数组下标为i，则父节点下标为 (i-1)/2 ，左孩子下标为 2i+1 ，右孩子下标为 2i+2 。

二叉堆还分为最大堆和最小堆。
- 最大堆：每个节点都大于等于它的两个子节点。
- 最小堆：每个节点都小于等于它的两个子节点。

两种堆核心思路都是一样的，本文以最大堆为例讲解，对于一个最大堆，根据其性质，显然堆顶，也就是 arr[0] 一定是所有元素中最大的元素。

### 优先级队列概述

优先级队列支持插入和删除两种操作，并支持获取元素的最大值，底层在元素被插入和删除时，会自动排序，底层维护了一个最大堆。

优先级队列插入元素时，从尾部插入，然后通过上浮调整到正确位置，删除元素时，将尾部元素交换位置，然后通过下沉调整到正确位置。

插入和删除操作，上浮和下沉调整的次数都是二叉堆的层数，所以时间复杂度是 O(logN) 。

完整代码
```C++
#include <vector>
#include <functional>

template <class ElemType, class Comp = std::less<ElemType> >
class PriorityQueue {
public:
	PriorityQueue() {
		_size = 0;
	}
	
	~PriorityQueue() {
		_heap.clear();
		_size = 0;
	}
	
	bool Empty() {
		return _size == 0 ? true : false;
	}
	
	ElemType Top() {
		return _heap[0];
	}
	
	void Push(const ElemType &elem) {
		if (_heap.size() <= _size) {
			_heap.push_back(elem);
			_size++;
		} else {
			_heap[_size++] = elem;
		}
		
		// 上浮
		adjustUp(_size-1);
	}
	
	void Pop() {
		// 下沉
		_heap[0] = _heap[--_size];
		adjustDown(0);
	}
	
private:	
	void adjustUp(int pos) {
		while (pos > 0) {
			int i = (pos - 1) / 2;
			if (_comp(_heap[i], _heap[pos])) {
				std::swap(_heap[i], _heap[pos]);
				pos = i;
			} else {
				break;
			}
		}
	}
	
	void adjustDown(int pos) {
		while (2*pos + 1 < _size) {
			int i = 2*pos + 1;
			if (i+1 < _size && _comp(_heap[i], _heap[i+1]))
				i++;
			if (_comp(_heap[pos], _heap[i])) {
				std::swap(_heap[pos], _heap[i]);
				pos = i;
			} else {
				break;
			}
		}
	}
	
	int _size;
	std::vector<ElemType> _heap;
	Comp _comp;
};
```

### 总结

二叉堆就是一种完全二叉树，所以适合存储在数组中，而且二叉堆拥有一些特殊性质。

二叉堆的操作很简单，主要就是上浮和下沉，来维护堆的性质（堆有序），核心代码也就十行。

优先级队列是基于二叉堆实现的，主要操作是插入和删除。插入是先插到最后，然后上浮到正确位置；删除是调换位置后再删除，然后下沉到正确位置。核心代码也就十行。
