# mxnet-insightface-cpp
<p align="center"> 
<img src="https://github.com/njvisionpower/mxnet-insightface-cpp/blob/master/demo.jpg" width = 60% height = 60%>
</p>  

### 基础
识别框架：MTCNN人脸检测+相似度变换进行对齐+识别网络抽取特征（比如性价比很高的**mobilefacenet**），i5-4590 CPU上**单线程**进行640 * 480人脸完整的识别过程(检测+对齐+识别)在30ms~40ms之间，这里要注意速度测试需要公平的环境，除硬件外比如线程数、你用的最小人脸大小、图像金字塔的缩放尺度、过滤人脸的置信度参数等等，并且级联的网络通常会有一定的波动，最大的原因是喂给RNet和ONet的proposal人脸的个数可能比较多，造成这两个网络的输入batch增多带来更多的开销，后面本项目会给出测试的具体时间。

### 更新日志
**2019-2-20  更新demo图片**

### 环境
    1.CPU和GPU都可以，主要对CPU进行优化，GPU后面补上~
    2.编译的MXNet库，看情况开启MKL，MKL-DNN或者openblas加速，速度会有不小提升，另外还需要预编译的OpenCV
### 运行
    先根据image文件夹中的图像打标签并生成特征，后根据这些标签和特征识别。
    1.本项目有个默认的标签和特征，标签"labels.txt"，特征"features.xml"
    2.你可以添加新的标签图像重新生成特征，见main的test_make_label函数。
### 检测部分
MTCNN是一个级联网络(原始版本任意输入的PNet,24 * 24输入的RNet和48 * 48的ONet)，想要优化的点还蛮多的，比如:  
1. 最开始的输入部分，由图像金字塔生成不同尺度大小的图像喂给PNet，PNet的输入虽然不多，但是前几个尺度较大的图像会比较吃开销，假如你的场景的图像尺寸固定，可以初始化k个(图像尺度决定)个PNet预测模型，一来大大省去模型reshape或者reload的时间，二来非常方便用多线程并行加速  

2. RNet输入小，但是在某些情况PNet输出的proposal face非常多，这个时候可以用batch进行并行，也就是对于k x channels x height x width会比k个1 x channels x height x width循环计算要快(GPU不用多说，CPU也会有一定的速度提升)  

3. ONet和RNet类似可以进行batch，另外ONet的输入稍微大点，且卷积运算较多，可以考虑砍模型，重新设计网络，用更少的卷积核，改变尺寸等等重新训练  

4. MTCNN的参数较多，实际测试中发现金字塔尺度参数不需要0.709，可以改小点如0.5，速度会有不小提升；三个网络过滤的置信度可以看情况改大一点缩小proposal的个数，minsize视场景调整  

注意:

1. MXNet预测模型的输入要求固定，改变大小需要reshape或者reload模型，这二者都非常吃开销，实际情形1,2,3点部分原因为此优化  
2. MTCNN的检测需要和识别保持统一，不要训练的时候用一套检测模型，部署的时候用另一套检测模型，这样会造成比较大的误差  

### 对齐部分
对齐部分用的是相似度变换，而不是opencv中常规的仿射变换，代码源自insightface提供的cpp版本。根据一个平均脸的五个landmark点，把检测得到的人脸根据平均脸进行相似度变换。注意：  

1. 这里用的相似度变换在python的skimage、dlib都有类似的实现，paper基于"Least-squares estimation of transformation parameters between two point patterns"  

2. 代码中insightface中平均脸的坐标有两个，一个是基于112 * 112，一个是基于112 * 96的，根据训练的部分自己选择

### 识别部分
用insightface或者其他MXNet训练的模型抽取特征，比较人脸之间的欧式距离，进行人脸验证或者识别。注意的点：  

1. 注意MXNet中模型的输入通道是batch x channels x height x width，并且默认的通道是RGB，用opencv需要把默认的BGR转成RGB，然后按照通道的张量形式把数据喂给识别的网络  

2. 一张图像可能有多个人，更好的方法是组成batch，然后根据加载好的模型的batch数丢给网络抽取特征  

3. 抽取的特征要进行归一化，有助于提高识别率  

4. 相似度阈值可以视具体的场景设置，是要更低的误识别率或者更高的召回率
