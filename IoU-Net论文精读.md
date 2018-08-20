# IoU-Net论文解读
---
###### title: "IoU-Net论文解读"
###### date: "2018-08-17 17:30"
---

论文：[Acquisition of Localization Confidence for Accurate Object Detection](https://arxiv.org/pdf/1807.11590.pdf)

### 一、摘要
> 在当前的目标检测算法中，都是通过检测框回归和非极大值抑制（NMS）来定位目标，将分类置信度当作key来筛选proposal进行分类，但却缺少了定位置信度，这使得正确定位的边界框在回归期间退化或者在NMS时被抑制。作者针对上述问题提出了IoU-Net来学习预测检测框和ground truth之间的IoU，进而获得定位置信度，再通过定位置信度来提高NMS的效果，保留更多正确分类的检测框。作者还提出了一种将预测的IoU作为目标的基于优化的检测框精修方法。将上述提出的方法作用在COCO数据集上，较之前的目标检测算法，均有不同程度的提升。

### 二、背景介绍
>目前的目标检测算法都被当作多任务学习问题:
> 1. 区分前景和背景，并分配相应的类别标签
> 2. 回归一系列通过最大化IoU来定位物体的检测框或者表征检测框和ground truth的向量
> 3. 通过非极大值抑制NMS来精简检测框
>
> 其中分类的标签可以天然的反映分类置信度，但是在优化检测框向ground truth靠近的时候，却缺少了定位置信度，这种情况会带来以下两个问题：
> 1. 错位的分类置信度和定位准确度：抑制重复检测框的过程忽略了定位精度，而分类置信度通常用作排名提案的度量标准，如下图中有较高分类置信度的检测框，却和ground truth有较小的IoU，这种分类置信度和定位准确度的不匹配，会导致定位检测框的准确度降低，在NMS时被定位准确度（IoU）低的检测框所抑制。
> ![分类置信度高，定位置信度低](assets/markdown-img-paste-20180817200003449.png)
> 2. 非单调性的bounding box regression：定位置信度的缺失，使得广泛采用的边界框回归变得难以解释。如在Cascade-Net中提出当进行多次检测框 regression迭代的时候可能会出现定位检测框退化。
> ![检测框 regression多次迭代](assets/markdown-img-paste-20180817201255541.png)
>
> 基于前述的问题，作者提出了IoU-Net，使得网络能够像了解分类置信度一样了解定位信息。未解决上述问题提出以下内容：
> 1. IoU-guided NMS： IoU是定位准确度的自然表征。作者使用预测的IoU代替分类置信度来用作proposal排名的关键值，来消除NMS抑制失败的情况
> 2. optimization-based bounding box refinement： 基于优化的检测框精修方法
> 3. Precise RoI Pooling (PrRoI Pooling)：精确RoI Pooling

### 三、问题分析
> #### 1. Misaligned classification and localization accuracy
> ![](assets/markdown-img-paste-2018081720471777.png)
> 上图显示了没有进行NMS前，被检测框检测框与ground truth的IoU和相应的分类置信度(classification confidence)或者定位置信度(localization confidence)的分布情况：
> 论文中作者对比了IoU大于0.5的情况下，(a)和(b)两个分布中的皮尔逊相关系数各自为0.217和0.617，从图中也可以看出来，(b)中显示的分类置信度能够更好的表征被检测框IoU的分布情况，下图也从IoU的数量上说明了分类置信度和定位置信度之间的不匹配
> ![](assets/markdown-img-paste-2018082010061377.png)
> 上图中，对于IoU > 0.9的情况，传统的缺失定位置信度的NMS对检测框框的抑制超过一半，最后得到的结果要比作者提出的使用定位置信度IoU-guided NMS得到的结果要少。
>
> #### 2. non-monotonic bounding box regression
> 通常情况下，目标检测算法中物体的定位问题被形式化为bounding box regression任务，且大多通过多次迭代的方法进行精修，即iteration bounding box regression，但在Faster R-CNN中使用两次迭代，但在Cascade R-CNN论文中说明多次迭代并没有太大的提升，并将这种结果归因于多步边界框回归中的分布不匹配，并通过多阶段边界框回归中的重采样策略对其进行处理。
> ![](assets/markdown-img-paste-20180817204825621.png)
> 在上图中，对比了不同网络结构中，作者提出的基于优化的方法和基于回归的方法中各自的迭代次数AP之间的关系图，可以看出基于回归的方法呈现一个非线性的分布趋势，并且在多次迭代后，对AP的提升不升反降，这种非线性的没有很好的解释性，很难应用于实际问题中。而且，没有对检测框检测框的定位置信度，很难进行细粒度的精修，例如针对不同的检测框进行不同次数的迭代。

### 四、IoU-Net
作者提出的IoU-Net主要体现在下边几方面：
#### 1. 学习预测IoU
> ![IoU-Net](assets/markdown-img-paste-2018082011242539.png)
> 如上图所示，作者在生成RoIs的过程有别于之前的Faster R-CNN的RPN网络，不是通过在feature map上生成anchors筛选出RoIs，而是将所有的ground truth加上一个随机的偏移量，并剔除掉IoU小与0.5的检测框，再从这些检测框中采样用于IoU-Net的训练，这样对于IoU-Net来说可以带来更好的表现效果和健壮性。然后通过作者提出的精准RoI Pooling从FPN输出的结果中抽取特征，再将这些特征喂给后边的两层网络的进行IoU预测。整个训练过程独立于检测器的，这样对于输入分布的变化有更好的鲁棒性。
>
#### 2. IoU-guided NMS
> 作者为了解决定位置信度和定位准确度的不匹配，提出了IoU-guided NMS方法，类比于传统的NMS方法，IoU-guided NMS将定位执行度作为检测框排名的关键词，具体如下：
> ![IoU-guided NMS](assets/markdown-img-paste-2018082011413179.png)
> 1. 创建集合D，保存检测框和对应的分类置信度s
> 2. 若检测框的集合B不为空，则将定位置信度I(bj)最大的检测框赋给bm，然后将bm从B中剔除，并将bm的分类置信度S(bm)赋给s
> 3. 遍历B中剩下的所有的检测框，记为bj，当bm和bj的IoU大于某个阈值的时候，用bm和bj中分类置信度高的值去更新s，并将bj从B中剔除
> 4. 将bm和对应的s成对存入D中
> 5. 重复执行2、3、4步骤，直至B为空，返回D
>
#### 3. 基于优化的检测框精修
> 检测框精修问题可以转化为寻找最有的C*参数，如下：
> ![公式(1)](assets/markdown-img-paste-20180820134524376.png)
> 上式中![](assets/markdown-img-paste-20180820134859930.png)代表了被检测框，![](assets/markdown-img-paste-20180820134945607.png)代表了ground truth，**transform**是以c为参数的变换方法，将给定的框进行变换，**crit**作为评价两个box之间距离的函数，在Faster R-CNN中，**crit**使用的是以log表示的smooth-L1距离。
>
> 基于回归的算法使用前馈网络直接估计参数c，但是迭代检测框回归的方法更容易受到输入分布的变化所干扰，而且许多结果都是非线性的提升，为解决该问题，作者提出了基于优化的的检测框精修的方法，这种方法使用IoU-Net作鲁棒性的定位精确度评估。IoU-Net直接预测检测框和ground truth的IoU，同时提出Precise RoI Pooling用来计算对应检测框的IoU梯度，如下图所示：
> ![](assets/markdown-img-paste-2018082014533692.png)
> 上图中，将IoU的预测作为优化对象，作者通过多次迭代计算梯度来精修检测框的坐标，并最大化检测框和对应ground truth的IoU。同时预测的IoU可以解释每个检测框的定位置信度和坐标变换。实现过程中，作者在算法的第6行，使用![](assets/markdown-img-paste-20180820150128901.png)手工增大坐标的梯度。作者在实现过程中使用一步检测框回归来初始化坐标。上述算法的具体细节如下：
> 1. 创建空的集合A，用来存放检测框
> 2. 迭代一步，遍历所有在B中但是不在A中的检测框，记作bj，计算IoU(PrPool(F, bj))和其梯度，分别赋给PrevScore和**grad**
> 3. 用2中得到的**gred**使用图中的公式更新bj(注意：此处用的是梯度上升)，再用更新后的bj计算IoU(PrPool(F, bj))赋值给NewScore
> 4. 若NewScore与PrevScore的间距小于早停阈值和定位置信的退化容忍度，则将bj放入A，否则遍历下一个检测框
> 5. 当遍历完所有B中的检测框，一次迭代完成，跳到2执行
> 6. 执行完T次后，返回更新后的B
>
> 作者同时还提出了Precise RoI Pooling来避免RoI Pooling和RoI Align引起的量化误差，下图是在中方法的对比
> ![](assets/markdown-img-paste-20180820152351128.png)
> 当给定池化前的feature map，以及上边离散的点W(i,j)，当使用双线性插值，离散特征映射在任何连续坐标（x，y）处都可以被认为是连续的：
> ![](assets/markdown-img-paste-2018082015375988.png)
> 其中![](assets/markdown-img-paste-20180820153822475.png)作为插值系数，并用**bin**={(x1, y1), (x2, y2)}表示每个**bin**，然后用对f(x, y)的二重积分并处以bin的面积来表征池化结果，如下所示：
> ![](assets/markdown-img-paste-2018082015422436.png)
>
#### 4. 联合训练
> IoU-Net可以兼容很多当前的目标检测网络架构中，如**1. 学习预测IoU**中的图所示，将PrRoI Pooling的结果并行的送入IoU预测网络和R-CNN网络。
> 训练细节体现在一下几个方面：
> 1. 使用在ImageNet上预训练的网络作为骨干网络，并使用均值为0，方差为0.01或者0.001的高斯分布来初始化所有新的层
> 2. 使用smooth-L1损失作为IoU预测网络的损失
> 3. IoU网络的训练数据是单独生成的，同时数据对应的labels被归一化到[-1, 1]区间
> 4. 分类和回归分支每张图片上取512个RoIs，并且训练过程使用的batch size为16
> 5. 学习率0.01迭代160k次，0.001时迭代120k，同时在训练前使用0.004的学习率迭代了10k次
> 6. 权重衰减为0.0001，momentum参数为0.9
> 7. 首次使用检测框回归来初始化检测框坐标
> 8. 使用IoU-guided NMS作用全部得检测框，再对分类置信度最高的前100个检测框使用基于优化的算法，其中step size为0.5，早停阈值0.001，定位置信的退化容忍度为-0.01，并且迭代次数为5
>
>
### 五、实验
#### 1. IoU-guided NMS
> ![](assets/markdown-img-paste-20180820161522881.png)
> 可以看到使用IoU-guided NMS方法后所有网络在AP90的检测结果上均有明显提升

#### 2. Optimization-base bounding box refinement
> ![](assets/markdown-img-paste-20180820161656278.png)
> 可以看到使用精修方法后所有网络在AP90的检测结果上均有明显提升

#### 3. joint training
> 联合训练结果对比
> ![](assets/markdown-img-paste-20180820162132482.png)
>
> 检测速度对比
> ![](assets/markdown-img-paste-20180820162357346.png)

### 六、总结
#### 1. IoU-Net网络的提出，解决了分类置信度和定位置信度不匹配的问题，使检测框获得了定位置信度，同时IoU-guided NMS使准确定位的检测框不被抑制
#### 2. 基于优化的检测框的精修方式与基于回归的方法不同，通过提出PrRoI Pooling来避免RoI Pooling和RoI Align的量化误差，达到了比较好的检测效果
