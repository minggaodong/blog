# tensorflow 运行原理
## 基本概念
TensorFlow 是 Tensor 和 Flow 的结合，翻译成中文就是 “张量” 和 “流” 。

TensorFlow 设计原理是先定义一张计算图，然后通过会话模式执行计算图上的操作。

### 张量
在深度学习里，Tensor（张量）实际上就是一个多维数组，相当于于 numpy 中的 array 。

#### 张量属性
每个 Tensor 都包含下面三个属性
- rank： 维度数
- shape: 各维度的长度
- type:  数据类型(float, string, int32, uint64)

#### 张量维度
Tensor 是向量和矩阵的扩展，在机器学习中，特征往往通过one-hot编码转成1维张量(向量)输入模型。
- 0 阶 Tensor 代表标量，是一个单独的数
- 1 阶 Tensor 代表向量，是一个1维数组
- 2 阶 Tensor 代表矩阵，是一个2维数组
- 3 阶 Tensor 代表立方体，是一个3维数组

### 计算图
在 TensorFlow 中用计算图来表示计算任务，计算图是一种有向图，用来定义计算的结构，实际上就是一系列的函数的组合。

计算图中每个节点包含了运算函数，多个 Tensor 输入和一个 Tensor 输出，而连接节点的有向线段就是 Flow，表达了张量之间通过计算相互转化的过程。

### 会话
在计算图中，张量是没有数据的，只是存储了“计算方式”，tensorflow 需要发起一个会话去执行计算图，张量对象只有在会话中初始化，访问和保存。

### 惰性计算
惰性计算是指当获取某个节点的输出结果时，Tensorflow 会从这个节点向前追溯，比如 Z = X + Y， 当获取节点 Z 的输出结果时，会先计算 Z 的两个输入节点 X 和 Y。

### 占位符
计算图中节点的输入依赖于前一个节点的输出，但是起始节点的输入依赖于计算图中的原始输入，不依赖于任何前驱节点，但是其它节点依赖它，tf 提供了占位符（placeholder）先将位置占住，
然后在执行的时候再将原始数据输入。

## 完整代码示例
```
import tensorflow as tf
import numpy as np
import matplotlib.pyplot as plt
 
#创建占位符
X = tf.placeholder(tf.float32)
Y = tf.placeholder(tf.float32)
 
#创建变量
w = tf.Variable(tf.random_normal([1], name='weight'))
b = tf.Variable(tf.random_normal([1], name='bias'))

#定义输出结果
y_predict = tf.sigmoid(tf.add(tf.multiply(X, w), b))

#定义损失函数
cost=tf.reduce_sum(tf.pow(y_predict-Y,2.0))/400
 
#定义优化方法
optimizer=tf.train.AdamOptimizer().minimize(cost)
 
#创建session，执行训练
num_epoch=500
cost_accum=[]
cost_prev=0
#np.linspace（）创建agiel等差数组，元素个素为num_samples
xs=np.linspace(-5,5,num_samples)
ys=np.sin(xs)+np.random.normal(0,0.01,num_samples)
 
with tf.Session() as sess:
    #初始化所有变量
    sess.run(tf.initialize_all_variables())
    #开始训练
    for epoch in range(num_epoch):
        for x,y in zip(xs,ys):
            sess.run(optimizer,feed_dict={X:x,Y:y})
        train_cost=sess.run(cost,feed_dict={X:x,Y:y})
        cost_accum.append(train_cost)
        print("train_cost is:",str(train_cost)) 
 
        #当误差小于10-6时 终止训练
        if np.abs(cost_prev-train_cost)<1e-6:
            break
        #保存最终的误差

```


