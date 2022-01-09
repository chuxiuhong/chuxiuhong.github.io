
------


先说好，我们的家庭监控是每分钟的照片的监控，并不是真正的实时视频，这种实时视频树莓派性能可能不够。

我们这一次工程的大体步骤：
![image0](/static/20161218215526.png)


## 硬件准备

我们至少需要一个树莓派3，树莓派的摄像头，一个tf卡（16G，class10的比较推荐），出于便于传数据，你最好还有读卡器或者SD卡套，常用的USB鼠标，USB键盘，HDMI接口的显示器（这个有则最好，没有会麻烦但是也能搞定，我默认你有）

首先看看我们的树莓派长成什么样子：

![image1](/static/IMG_0643.JPG)

它有四个USB接口，一个网线口，一个HDMI接口，一个摄像头的接口，还有电源口，还有一些不是很常用的口，以及GPIO。

看看我们的摄像头长成什么样子

![image2](/static/IMG_0644.JPG)

很简单的一个小摄像头，大概500W像素，淘宝一般卖35左右。

除了上面的两个我们要求是统一的，至于键盘鼠标显示器我都不管你用的什么型号的。



## 安装系统和基本设置

安装系统这方面的教程网上实在是太多了，不需要搜英文的资料，只需看看百度的结果就可以完美解决。下面我默认树莓派上的系统已经做完了。

在树莓派上进入终端（如果选择debian系统的话，和Ubuntu的操作基本一样），执行
```
sudo raspi-config
```

出现下面的页面

![image3](/static/raspiconfig.png)

先选择第一项，扩充文件系统，让树莓派可以完全的占有你的tf卡。完事之后可能需要重启，重启之后我们还是执行上述命令，然后还是来到这个页面，选择选择第五项，然后一直选OK，打开摄像头的接口。

然后我们关机
```
sudo shutdown -h now
```
或者是干脆直接断电源其实也没有问题。

**警告!!!!!!!!!!!!!!!!!!!!!!!!!**
绝对不可以带着电源的情况下插入摄像头，如果带电操作，十之八九你的摄像头会GG，博主血泪教训。而且摄像头GG了之后每次调用还是会亮灯，只是你接受不到数据，这个问题我已经Google了很长时间，老外们也是一脸懵逼，大家普遍认为应该是被烧坏了= = 


我们把摄像头插到树莓派上，如图：


![image5](/static/IMG_0638.JPG)

![image6](/static/IMG_0647.JPG)

需要将摄像头底下那个蓝色的一面朝向USB接口那个方向，不要插反了。

等到你都安装完毕了，确保连接好各个硬件之后再给电源。（千万记得不要热插拔摄像头）


## 准备七牛云

为什么非常突兀的在这里提到七牛云，原因是我们总需要一个存储监控的数据的空间，自己写一个简单的服务器代码也是可以，不过云服务器现在便宜的带宽太小，贵的我们穷苦学生又玩不起，不如用一个七牛云，简单还免费。（实名注册用户拥有10G免费空间，题主markdown的图片外链都是拿这儿做的）

首先我们来到七牛云官网，注册账号  http://www.qiniu.com/

登录之后，如图操作

![image6](/static/qiniu1.png)

![image7](/static/qiniu2.png)

![image8](/static/qiniu3.png)

把这这个密钥对存起来，我们一会用

![image9](/static/qiniu4.png)

我们需要新建一个仓库，点开之后自己任意选节点，其实国内的几个节点速度都差不多，完全可以满足需求。

以后我们获取的监控照片就可以来这里查询


## 代码

下面的代码既可以现在本地上写之后再用github克隆过去或者是U盘copy过去，或者是直接在树莓派上写都可以，不过记得如果是前者，那么安装第三方库和配置东西要同步配置。

首先，我们写一个.sh脚本
take_photo.sh

```
raspistill -o current_photo.jpg
python test.py
```

然后安装七牛云的python SDK，在命令行内执行
```
sudo pip install qiniu
```

在take_photo.sh同目录下我们建立一个test.py

```python
# -*- coding: utf-8 -*-

import time
from qiniu import Auth, put_file, etag, urlsafe_base64_encode
import qiniu.config
import os
#需要填写你的 Access Key 和 Secret Key
access_key = '' #这里的密钥填上刚才我让你记住的密钥对
secret_key = '' #这里的密钥填上刚才我让你记住的密钥对

#构建鉴权对象
q = Auth(access_key, secret_key)

#要上传的空间
bucket_name = 'mypi'

#上传到七牛后保存的文件名
key = '%s_%s_%s_%s_%s_%s.jpg'%(time.localtime()[0],time.localtime()[1],time.localtime()[2],time.localtime()[3],time.localtime()[4],time.localtime()[5])

#生成上传 Token，可以指定过期时间等
token = q.upload_token(bucket_name, key, 3600)

#要上传文件的本地路径
localfile = 'current_photo.jpg'

ret, info = put_file(token, key, localfile)

filename = 'current_photo.jpg'
if os.path.exists(filename):
    os.remove(filename)
```

这样一来，我们每次执行take_photo.sh脚本，都可以让树莓派拍一张照片并且发送到七牛云上，我们只需登录就能看见下面这样的数据

![image10](/static/qiniu5.png)
文件命名是以年月日时分秒的方式命名的

但是这样我们总不可能手动的一次次执行，那样也不叫监控了。最简单的想法，我们可以利用Linux的定时任务crontab管理这个脚本

进入命令行，执行
```
crontab -e
```

![image11](/static/crontab.png)
在末尾追加上
```
* * * * * /home/pi/take_photo.sh
```

然后按Ctrl+x，按Y，保存修改。
之后重启cron
```
sudo service cron restart
```

然后我们的定时监控就完成了！把它安放到想要的位置，它会每分钟拍下照片并且发送到七牛云，你可以使用七牛云的本地同步工具qshell来方便的查看更新照片。

qshell使用教程 http://developer.qiniu.com/code/v6/tool/qshell.html

写代码的时候自动拍摄的样图：
![image12](/static/2016_12_18_21_14_7.jpg)

![image13](/static/2016_12_18_16_57_53.jpg)


其实本文涉及的内容仅仅是我们一门课程中的小项目三分之一的部分，原本的用途也不是作为家庭监控的，但拿出来与大家分享，无论你是从头到尾实现这个监控器，还是取一小段用于他途，只要有帮助就好。

博客保持更新，愿意来定时看我脑洞的，请关注我