# Alibaba-Cloud-German-AI-Challenge-2018

这是天池大数据的比赛<br/>
https://tianchi.aliyun.com/competition/introduction.htm?spm=5176.100067.5678.1.3e7731f5WP7NmY&raceId=231683<br/>



1.数据集描述：<br/>
数据集分为训练集和验证集，图像包括sen1,sen2两部分，其中sen1是sar图像,分为实部和虚部,有两个通道是滤波处理的。sen2是高光谱图像<br/>

特点：<br/>
a.由两个卫星的图层叠加而成，可以视为32,32,18的图片<br/>
b.测试集和验证集均出现类别分布不平衡情况，可以视为多标签不平衡分类问题<br/>
c.sen1图层的像素波动很大（从负数到上千），sen2图层像素多为0-1之间的浮点数<br/>

2.训练模型（分为预处理，特征提取，模型训练，后处理4个模块）<br/>

模型训练进展：<br/>
网络采用了L_Resnet_E_IR, 损失函数基于softmax entropy<br/>
尝试不同激活函数和损失函数，focal loss, center loss等<br/>

预处理模块：<br/>
1.对于数据分布不平衡的问题，采用重复过采样，保证训练时的数据分布平衡<br/>
2.图像增强，旋转，对称操作

特征工程模块：<br/>
图像归一化<br/>
rgb通道去雾+对比度增强<br/>
对sen1实行滤波去噪+实复通道融合<br/>
参考blog http://blog.sina.com.cn/s/articlelist_1984634525_4_1.html<br/>
https://zhuanlan.zhihu.com/p/52477264<br/>

后处理模块：<br/>
先利用神经网络学习特征，然后获取神经网络最后一层的向量，利用传统分类器，如GBDT,LightGBM来分类<br/>

1.利用训练集，迭代70000步，在训练集上达到过拟合（拟合度100%），在验证集上面准确率在60%左右<br/>
2.训练期间，实现early stopping，34000步时达到最优，此时对于训练集拟合度在90%，验证集准确率61%<br/>
3.结合1，2，可以看出训练集过不过拟合对于验证集的分类没有太大影响<br/>
4.提交在验证集上性能最优的模型（61%准确率），线上测试集效果不到60%<br/>
5.从测试集和验证集准确率近似，提出猜想，测试集和验证集的分布近似，所以决定利用验证集来辅助模型训练<br/>
6.将训练集和验证集融合起来进行训练，达到过拟合（100%）后提交结果，线上效果达到75%<br/>
7.训练集为train+val, 针对数据的不平衡情况，采用了权重采样处理，保证每个batch的迭代中各个类别的分布比例平衡，采用early stopping，线上达到77.9%<br/>
---------------------------------------------------------------------------------------12.6日进展，线上排名前5%<br/>
8.构造数据集：从val训练集里面拆分3000个样本出来构造val21000和val3000，保证类别分布一致<br/>
9.train+val21000做训练集，val3000做验证集，将18个通道每个通道训练一个神经网络，利用投票记数法，将18个网络模型进行集成，线上72%<br/>
10.train+val21000做训练集，val3000做测试集，将18个通道每个通道训练一个神经网络，根据在每个模型在各个类别上的performance构建权重矩阵(18* 17)，对18个网络模型进行集成，线上74%<br/>
11.5层神经网络，val21000做训练集，只用sen2, val3000上达到92%，线上78.1%<br/>
12.L-Resnet-E-IR网络，val21000做训练集，只用sen2, val3000上达到94%，线上79.4%<br/>
13.利用线上77%，78%，79%的结果进行加权投票，线上81.7%<br/>
14.train+val21000做训练集，只用sen2, val3000上90.4%, 线上77.7%<br/>
---------------------------------------------------------------------------------------12.28日进展，线上排名113/1300<br/>
15.因为即将要换榜了，2天时间内是无法拿到多个换榜后的在线准确率的，无法实施加权投票，所以加权投票的方式改为简单投票看一下效果，标签不确定的取较容易出现的样本（即如果1，2，3模型分别投票给a,b,c标签，那么选择a,b,c三类中样本数量最多的标签，我们暂且假设网络对于样本多的标签预测能力好一点），依然是77,78,79集成，线上81.2%，可以看出没有太大区别，基本确定简单投票方案可行<br/>
16.为了保证模型的泛化能力，将train和valid数据集进行合并成mega.h5，并且拆分出3000+1000合计4000个样本进行验证<br/>
17.将val21000+sen2通道，train，val+sen1,sen2通道，train,val+预测性能最强的3个通道，train,val+sen2通道训练出4个网络进行简单投票加权，线上82.1%<br/>
----------------------------------------------------------------------------------------1.5日进展，线上排名150/1375<br/>
18.换榜后，线上81.4%<br/>
----------------------------------------------------------------------------------------1.7日进展，线上排名92/1368<br/>
19.集成了一个Resnet50,train+val,sen2,5W训练数据的结果，线上82.5%<br/>
----------------------------------------------------------------------------------------1.8日进展，线上排名91/1328<br/>
##赛季二数据集更新<br/>
20.mega.h5，sen2，acc4000上0.89，线上77%<br/>
21.mega.h5，sen2原始数据+sen1的0,1,2,3复变量长度生成2个通道，acc4000上88%，线上75%<br/>
22.mega.h5，sen1,sen2全部通道不做任何处理，acc4000上39%，mega1和mega2上的效果差不多，都在0.38+，线上73%<br/>
23.将train和val打乱，28划分数据集，线下88%，线上75%<br/>
24.val21000，sen1+sen2，val3000上88%，线上73%<br/>
25.mega.h5，sen1,sen2归一化，acc4000上91%，线上77%<br/>
26.mega.h5，sen2的所有通道除以2.8，对0，1，2（rgb）通道做去雾+对比度增强，acc4000上84%<br/>
27.mega.h5，sen2，sen1(复数通道融合+其余通道)，线下18%<br/>
28.依然采用赛季1的模型融合方案，将手上线上准确率在75%上的模型进行融合，线上81.1%，最终赛季排名71/1329
