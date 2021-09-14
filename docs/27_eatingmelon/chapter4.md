# Chapter 4: 决策树算法

## 4.1 基本流程

1. 决策树分类器会基于树模型对样本进行划分，叶结点对应决策结果，每个分支对应一次属性测试，根节点包含样本全集。决策时的学习目的是为了产生一颗泛化能力强，能够处理从未见过例子的决策树。

## 4.2 划分标准

要能够创建一个决策树，其中最重要的部分就在于如何选择最优划分属性。总体来说，我们希望随着不停划分，节点的纯度越来越高。接下来主要记录三种决策时的划分方式

### 	4.2.1 信息熵与信息增益：ID3 算法

1. 信息熵 (Information Entropy)：

   信息熵是用来度量纯度的一种指标，假定当前样本集合$D$中的第$k$类样本所占比例为$p_k$，则信息熵的定义如下：
   $$
   Ent(D) = -\sum_{k=1}^{n}p_klog_2p_k
   $$
   不难发现，纯度越小，$p_k$越大，熵越小

2. 信息增益 (Information gain):

   这里采用信息增益，而不是单纯信息熵作为划分条件的目的是因为：我们需要在划分后给每个子集的熵加权。从信息熵的计算公式来看，每一个结点所占的比重是一样的，然而实际上我们希望样本多的结点所占的权重更高一些，即样本多一点的结点划分的多一些，样本少的结点划分少一些。因此我们用子集样本数量除以总体数量来作为权重进行加权。每轮划分前的信息熵减去加权后划分的信息熵就是信息增益
   $$
   Gain(D,a)=Ent(D)-\sum_{v=1}^{V}\frac{|D^V|}{|D|}Ent(D^V)
   $$
   其中$|D^V|$为划分后集合的样本个数，$|D|$为所有样本个数

   选择信息增益作为划分标准就是著名的ID3算法。我们考虑每个划分结点的信息增益，选择最大信息增益的结点作为划分标准。

### 4.2.2 增益率 C4.5

1. 引入目的：

   我们回顾ID3算法，会发现另一个局限点：ID3算法对于样本分类多的特征会有更大的偏好，比如一个特征有17个取值，那么把这17个取值分开来一个一个划分的信息增益会非常大（$D^V$特别小），但是如果碰上比如有编号这种特征，一个一个划分出来显然不具有泛化能力。解决方法也很简单，我们直接加一个权重：1/所有子集的数量，即可。因此我们定义增益率如下：
   $$
   Gain\_ratio(D,a)=\frac{Gain(D,a)}{IV(a)}
   $$

   $$
   where, IV(a) = -\sum_{v=1}^{V}\frac{|D^V|}{|D|}log_2\frac{|D^V|}{|D|}
   $$

   其中$IV(a)$称为特征的固有值（intrinsic value）

2. C4.5算法：

   由于这个信息增益率又对包含取值少的特征有偏好，所以C4.5并没有完全采用增益率取代信息增益，而是用了一种启发式算法：先算信息增益，取出信息增益高于平均的划分点，再选取其中增益率最优的作为最终划分点。

### 4.2.3 CART算法

1. 基尼系数(Gini index)：

   之前两种算法分别是用信息增益以及增益率作为划分算法，CART算法是使用的是基尼系数来决定划分属性，这些系数总体的含义都是一样的，都是用来反应集合的纯度。基尼系数很好理解，就是指在一个集合里随机抽两次，两次抽到异类的概率（即1减去两次抽到同类的概率）。这个当然可以用来反映纯度。
   $$
   Gini(D) = 1-\sum_{k=1}^{n}p_k^2
   $$

2. 具体算法：

   CART算法可以同时解决回归和分类任务，他的具体算法如下：

   注：CART算法只会生成二叉树

   1）根据特征a的取值，划分数据集分为a=v和a!=v

   2）以如下公式来计算基尼系数：
   $$
   Gini\_index = \frac{|D^{a=v}|}{|D|}Gini(D^{a=v})+\frac{|D^{a!=v}|}{|D|}Gini(D^{a!=v})
   $$
   3）选择最小值来作为划分点，分裂树直到满足条件
