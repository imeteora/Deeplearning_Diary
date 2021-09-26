# 简化版mnet结构

RetinaFace的mnet本质是基于RetinaNet的结构，采用了特征金字塔的技术，实现了多尺度信息的融合，对检测小物体有重要的作用，RetinaNet的结构如下
![RetinaNet](https://img-blog.csdnimg.cn/20190821155358325.png#pic_center)

简化版的mnet与RetinaNet采用了相同的proposal策略，即保留了在feature pyramid net的3层特征图每一层检测框分别proposal，生成3个不同尺度上的检测框，每个尺度上又引入了不同尺寸的anchor大小，保证可以检测到不同大小的物体。

简化版mnet与RetinaNet的区别除了在于主干网络的选择上使用了mobilenet做到了模型的轻量化，最大的区别在于检测模块的设计。mnet使用了SSH检测网络的检测模块，SSH检测模块由SSH上下文模块组成

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190821155814691.png#pic_centercenter)

上下文模块的作用是扩张预检测区域的上下文信息。上下文模块和conv结合组成了一个检测模块

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190821155617191.png#pic_centercenter)

上图为SSH网络的检测模块，将一个上下文模块与conv叠加后生成分类头和回归头得到网络的输出。
mnet网络在使用SSH检测模块的同时实现了多任务学习，即在分类和回归的基础上加入了目标点的回归。官方的网络结构采用了5个目标点的学习，后续也可以修改为更多目标点，比如AFLW中的21个目标点以及常用的68或者106个目标点
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190821160126691.png#pic_center)
上图是原版RetinaFace论文中的检测分支图，注意在开源简化版的不包含self-supervision部分，但是有5个关键点的extra-supervision部分



# 损失函数



![img](https://img-blog.csdnimg.cn/2020030112571338.png)



分别是分类focal-loss，box回归IOU loss，人脸关键点，密集的回归。

## 应用细节

* 加入FPN

* 在5个金字塔特征图中加入了独立的上下文特征提高建模能力

* 还可用变性卷积（DCN）替换了横向连接和上下文模块中的所有3x3卷积

* 多正样本，采用同样的损失头和不同的特征图在计算多损失和

* anchor设定：在从P2到P6的特征金字塔级别上使用特定比例的描点

* 采用OHEM优化正负anchor不平衡的问题

**图像增强**

因为wider face有20%的小人脸，于是增强的时候，以图像的短边为基准，随机resize [0.3, 1]，在调整成640*640的方形块，如果人脸的中心点在方形块中，则保留这个人脸。

## 文章的思路（最关键）

1. 为了速度，所以在一阶段的通用检测器上进行改进，而retina检测算法又是当前精度最高的。
2. 加入了上下文建模模块，DCN等小的部件
3. 加入了人脸关键点分支，这个确实有一定作用
4. 多损失函数的融合。
5. 自监督方法的引入。（个人认为这是最可以借鉴的一点，也是借鉴之后具有通用性和可复制性的一点）

# focal-loss

​	focal loss主要是为了解决one-stage目标检测中正负样本比例严重失衡的问题，该损失函数降低了大量简单负样本在训练中所占的权重，也可以理解为一种困难样本挖掘



##  损失函数形式



  focal loss是交叉熵损失函数基础上进行的修改，首先回归二分类交叉上损失：



  ![img](https://images2018.cnblogs.com/blog/1055519/201808/1055519-20180818162755861-24998254.png)



y'是经过激活函数的输出，所以在0-1之间。可见普通的交叉熵对于正样本而言，输出概率越大损失越小。对于负样本而言，输出概率越大则损失越小。此时的损失函数在大量简单样本的迭代过程中比较缓慢且可能无法优化到最优。那么Focal loss是怎么改进的呢？



![img](https://images2018.cnblogs.com/blog/1055519/201808/1055519-20180818174822290-765890427.png)



![img](https://images2018.cnblogs.com/blog/1055519/201808/1055519-20180818170840882-453549240.png)



首先在原有的基础上加了一个因子，其中gamma>0使得减少易分类样本的损失。使得更关注困难的，错分的样本。

例如gamma为2，对于正类样本而言，预测结果为0.95肯定是简单样本，所以（1-0.95）的gamma次方就会很小，这时损失函数值就变得更小。而预测概率为0.3的样本其损失相对很大。对于负类样本而言同样，预测0.1的结果应当远比预测0.7的样本损失值要小得多。对于预测概率为0.5时，损失只减少了0.25倍，所以更加关注于这种难以区分的样本。这样减少了简单样本的影响，大量预测概率很小的样本叠加起来后的效应才可能比较有效。

此外，加入平衡因子alpha，用来平衡正负样本本身的比例不均：文中alpha取0.25，即正样本要比负样本占比小，这是因为负例易分。



 ![img](https://images2018.cnblogs.com/blog/1055519/201808/1055519-20180818174944824-933422059.png)



只添加alpha虽然可以平衡正负样本的重要性，但是无法解决简单与困难样本的问题。



gamma调节简单样本权重降低的速率，当gamma为0时即为交叉熵损失函数，当gamma增加时，调整因子的影响也在增加。实验发现gamma为2是最优。

##  总结

作者认为one-stage和two-stage的表现差异主要原因是大量前景背景类别不平衡导致。作者设计了一个简单密集型网络RetinaNet来训练在保证速度的同时达到了精度最优。在双阶段算法中，在候选框阶段，通过得分和nms筛选过滤掉了大量的负样本，然后在分类回归阶段又固定了正负样本比例，或者通过OHEM在线困难挖掘使得前景和背景相对平衡。而one-stage阶段需要产生约100k的候选位置，虽然有类似的采样，但是训练仍然被大量负样本所主导。






