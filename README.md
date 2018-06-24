Fashion AI keypoint 4.27%


# 1. 框架
比赛最后提交的方法主要是基于旷世的人体检测框架CPN，框架比较简洁  
[Cascaded Pyramid Network for Multi-Person Pose Estimation](https://arxiv.org/abs/1711.07319)  
另外吴双忱用过Hourglass  
[Stacked Hourglass Networks for Human Pose Estimation](https://arxiv.org/abs/1603.06937)  
但是后面调参的结果没有CPN好，并且比赛不允许两个以上模型，就没再用了。

# 2. 思路
思路比较简单，每一类跑两个模型（都是CPN检测关键点），第一个模型用于裁剪目标区域，第二个模型用于获得最后检测结果。首先用原始图像训练一个粗略关键点检测模型，用来获得初步检测点，然后用获得的这些点把目标区域裁剪下来，裁剪下来的图像resize pad到512*512，训练第二个精准的关键点检测模型。

## 2.1 关于目标区域检测
正常方法应该是使用额外一个检测器来检测目标区域，但是跟吴双忱尝试过SSD、Faster-RCNN-FPN、YOLO V3，可能是调参没有
调好的问题，效果没有使用CPN检测关键点来裁剪图像效果好，那三个框架Faster-RCNN-FPN效果最好，但是有些误检，就没再用了
（这部分代码要是需要，后面周末我再整理一下传上来，现在这里面没有）。用关键点来裁剪目标区域就存在一个问题，只能起到增大
分辨率和缓解一下位置分布、目标大小的干扰的作用，如果第一阶段的关键点检测有较大的失误，比如blouse几个外围点检测错，裁剪
下来的图像很可能不全或者过大，一些对CPN有影响的点如果仍然被包含进来，第二阶段很难改正过来。很多大佬的使用的是Faster-RCNN。

## 2.2 数据部分
数据增强使用了旋转、scale、色彩抖动，crop我用了以后发现效果变差了，后面就去掉了，另外在训练的过程中只对可见点的误差进行回传，这样效果最好

## 2.3 测试增强
在第一阶段获取粗略关键点的时候，采用翻转、scale测试取平均有效果提升，第二阶段由于相当于做了scale归一化，
只有左右翻转有用了。

## 2.4 分数历程
* 不使用检测，直接一阶段CPN（输入尺寸为384*384），调到6%左右（这里的6是使用DeepFashion的数据初始化训练来的） 
* 加检测，-0.8%左右  
* 从热图中提取检测点时，加高斯滤波和向次高点偏移，-0.2%左右  
* 测试增强，-0.2%左右  
* 输入尺寸从384到512，-0.3%左右  
* 其他参数调优，-0.2%左右


# 3. 代码

stage1 第一阶段相关代码，获得粗略检测点

stage2 第二阶段相关代码，获得最后结果

把除了测试集以外的所有图像都放在data/image_ori/Images中去，然后data_lib中分别生成S1和S2的tfrecord,然后分别在stage1和stage2中训练

需要训练好的权值的话，我拷给你们，代码有些注释可能不太清晰，后面我再添加一下，太讨厌文字工作了，哈哈
