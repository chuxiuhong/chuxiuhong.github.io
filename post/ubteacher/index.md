
unbiased-teacher是旨在使用半监督学习应用于目标检测的项目[https://github.com/facebookresearch/unbiased-teacher](https://github.com/facebookresearch/unbiased-teacher)

按照repo首页所描述的，本应该是很简单的操作，但是由于其所依赖的detectron2的新版本V0.6删除了一些需要使用的代码（震惊吧，我也是第一次知道新版本会删除老版本关键代码的大厂项目），一旦按照它的操作来做，最终就会报错在detectron2中找不到某个类，这样一来问题就变成找到如何满足detectron2 V0.5的环境，以下给出我的答案

首先安装好conda

```bash
wget https://mirrors.tuna.tsinghua.edu.cn/anaconda/archive/Anaconda3-2021.11-Linux-x86_64.sh
bash Anaconda3-2021.11-Linux-x86_64.sh
```

安装过程最后问是否要初始化，要yes，此时ssh可能中断，重新连一下，这一步会让程序加入路径中，后面会很方便

conda安装cudatoolkit 9.2版本，cudnn找对应版本，pytorch安装1.7版本

```bash
conda create -n detectron2 python=3.6
conda activate detectron2
conda search cudatoolkit
conda search cudnn
```

这里search是查看一下cudatoolkit有没有9.2的版本，如果没有，应该是你的环境不支持，目前测试1080Ti/Titan xp/2080Ti都可以使用此方案，如果有对应版本，再看看cudnn有没有对应版本，然后按照找到的对应版本号

```bash
conda install cudatoolkit==x.x cudnn==x.x.x
```

安装好cuda环境之后就是安装pytorch环境

```bash
conda install pytorch==1.7.0 torchvision==0.8.0 torchaudio==0.7.0 cudatoolkit=9.2 -c pytorch
```

到此，安装detectron2 V0.5版本的基础环境构建好了，可以检查一下pytorch能否正常调用gpu

```python
#python
import torch
print(torch.cuda.is_available()) #True
```

下一步就是把v0.5的代码下载下来，在[https://github.com/facebookresearch/detectron2/releases](https://github.com/facebookresearch/detectron2/releases)

```bash
cd detectron2-0.5
python setup.py install
```

这里有可能会报出来各种各样的环境依赖错误，找不到某个包的某个指定版本的问题，这里见招拆招，报出来什么问题就去setup.py里改，大胆的放宽版本要求，比方说==换成>=，可能一次解决一个包，大概重复这样的操作几次就可以正常安装下来，也有可能第一轮自动就完成了安装操作。

完成detectron2的环境搭建之后，回去再执行unbiased-teacher里面给定的命令，即可运行起来demo。

