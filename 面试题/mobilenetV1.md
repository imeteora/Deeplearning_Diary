# mobilnetv1

相对于传统的CNN网络，在准确率小幅度降低的前提下，大大减少模型参数和运算量。

一句话概括，V1就是把VGG的标准卷积换成了深度可分离卷积；

**模型亮点**

\* 深度可分离卷积，大大减少参数量

\* 增加超参数 α、β 

## 可分裂卷积

可分离卷积大致可以分为***\*空间可分离和深度可分离\****

### 空间可分离

顾名思义，就是把一个 大卷积核 换成两个小卷积核，例如；



![img](https://img2020.cnblogs.com/blog/1603920/202003/1603920-20200327155603835-1182149102.png)



### 深度可分离

深度可分离由深度卷积和逐点卷积组成

***深度卷积：***论文中称为 DW 卷积，就是把卷积核变成单通道，输入有 M 个通道数，就需要 M 个卷积核，每个通道 分别进行卷积，最后做叠加，如下图



![img](https://img2020.cnblogs.com/blog/1603920/202003/1603920-20200327160052136-625672916.png)



***\*逐点卷积\****：论文中称为 PW 卷积，就是用 1x1 的卷积核进行卷积，作用是 对深度卷积后的 特征 进行 升维；

![img](https://img2020.cnblogs.com/blog/1603920/202003/1603920-20200327160256541-150002879.png)

上面两步实现的效果如下图



![img](https://img2020.cnblogs.com/blog/1603920/202003/1603920-20200327160421399-2026782228.png)



可以看到输出 的 shape 是一样的；

下面我们看看在 输入输出 shape 一样时各自的参数量

标准卷积

params：Dk x Dk x M x N

计算量：Dk x Dk x M x Dh x Dw x N

可分离卷积

params：Dk x Dk x M + M x N

计算量：Dk x Dk x M x Dh x Dw + M x Dh x Dw x N



![img](https://img2020.cnblogs.com/blog/1603920/202003/1603920-20200327161250569-779381124.png)



![img](https://img2020.cnblogs.com/blog/1603920/202003/1603920-20200327161449416-864794837.png)



N 为 输出通道数， Dk 为 卷积核 size，如果 size 为 3，***\*减小了 1/9 左右\****，厉害了；

### V1 卷积结构

把 vgg 的标准卷积 和 V1 的深度可分离卷积 对比看下

![img](https://img2020.cnblogs.com/blog/1603920/202003/1603920-20200328103418264-1626165500.png)



看到了 深度卷积 （3x3 Depthwise）和逐点卷积 （1x1 conv）；

同时看到了 Relu6，什么鬼？

### Relu6 

上图，一看便知



![img](https://img2020.cnblogs.com/blog/1603920/202003/1603920-20200328132721509-1222112216.png)



Relu6 对 大于 0 的部分做了一个边界，作者认为，在模型精度要求不是很高的情况下，边界使得模型鲁棒性更强；

强行压缩数值比较大的特征，避免了 个性特征，也相当于 规范了特征，防止过拟合，玄学要自己体会；

V1 网络结构

说了那么多，该上结果了



![img](https://img2020.cnblogs.com/blog/1603920/202003/1603920-20200328100325440-1235222825.png)



Conv 代表标准卷积，Conv dw 代表深度可分离卷积，s2 代表步长为2，s1 代表步长为1； 