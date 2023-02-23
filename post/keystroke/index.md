


简单来说，我们要做的就是一种通过用户敲击键盘的习惯进行身份鉴别的系统。国内外之前有一些相关研究，但是通常是数千条数据训练，而且不能随意改变敲击的字符串，或者是有的要求采用带有压力传感器的键盘，难以实用和推广。我们做一个比较简单的根据匹配相似度的系统，采用普通键盘即可使用，其算法实现很简单。

关于该领域的介绍推荐看这篇综述及其引用：https://www.hindawi.com/journals/tswj/2013/408280/

首先说我们的一个应用场景：我们可以在各种网站的登录界面部署系统，当用户输入密码时，不止验证密码是否正确，同时将这次密码输入同注册时的密码输入习惯进行匹配，如果相似度较低，则增加验证方式，如手机验证码。那么针对这种场景，我们想了一点方法。

## 数据采集处理

我们需要考虑从键盘输入我们能得到什么，最基本的能得到多个三元数据，分别为（键盘码，时间点，按键类型）。通过键盘码我们可以确定按键的字符，按键类型指的是键盘被按下还是抬起。

例如某次输入一个字符串，我们所得到的基本数据：


key|time|type
|:----:|:----:|:---:|
|65|0|0|
|65|101|1|
|80|151|0|
|80|241|1|
|83|260|0|
|83|350|1|
|76|421|0|
|76|601|1|
|75|670|0|
|75|740|1|
|81|770|0|
|87|871|0|
|81|961|1|
|87|994|1|

key为键盘码，time单位为ms，type=0为按下，type=1为抬起

采集数据后，我们要考虑键盘按键事件之间的关系。采用前面提到的那篇综述的一张图
![image1](/static/keytrace_01.png)

在这里，可以看出能提出两种特征，一种是驻留时间特征(dwell time)，另一种是飞跃时间特征(flight time)。驻留时间特征是说一个按键被按下后持续的事件，飞跃时间特征有图中的四种定义，我们采用的是从某个按键被抬起到下一个按键被按下之间的时间差作为飞跃时间特征。（即图中的$F_{type1}$）

从前面那个例子提出的驻留时间特征为(101,90,90,60,80,70,90)，为什么没有标出是哪些按键的驻留时间，是因为在这个应用场景中，我们的密码不会变化，这样的话我们就无需考虑记录对应位置。
相应地，提出的飞跃时间特征如下：{“65-80”:50, “80-83”:19, “83-76”:71, “76-76”:40, “76-75”:69, “75-81”:30,“81-87”:-90}，与驻留时间特征不同的是，驻留时间是一个向量，其每一项的含义特定，而飞跃时间特征为一个键值对的集合，假设出现多个键值对的键值相同时，我们就将其取平均值。而且注意到，最后一个“81-87”这一项对应的值为负数，并不是bug，这种情况是非常常见的现实情况，读者可以想想是什么情况出现了。

从鉴别的角度来看，实验中飞跃时间特征对于区分用户的作用非常明显，应该说相比于飞跃时间特征，驻留时间特征对于系统贡献微乎其微，如果有想实现这种系统的读者，建议先用飞跃时间特征。（纯粹的经验啦，可能不正确，也没什么论文支撑）


## 匹配算法

至此，我们拿到了两种特征。我后面仅以飞跃时间特征为例（若想综合驻留时间特征的信息，可以分别去按照下面的算法计算相似度，然后做加权平均）。

我们假设由注册时采集的用户密码输入得到的特征为$F_A$,其是一个键值对集合，登录时同样可得到类似的特征$F_B$。我们将其对应键的值排好，分别生成两个序列
$A = v_{A1},v_{A2},....,v_{A_n}$
$B = v_{B1},v_{B2},....,v_{B_n}$

for(i = 1;i < n;i++){
    B[i] = $\lambda * B[i] * \frac {sum_B - sum_A} {n*sum_A}$
}
这里我加了一个系数$\lambda$，其实是一种惩罚系数，因为实验发现如果密码输入过快或者密码过短，通常的匹配方式都会出现过高的相似度，导致漏检率较高。在我们的实验中，发现通常取2到3之间效果不错。

$m = \frac {\Sigma_{i=1}^n v_{Ai} * v_{Bi} } {\sqrt {\Sigma_{i=1}^n v_{Ai}^2} * \sqrt {\Sigma_{i=1}^n v_{Bi}^2}}$

S = m if m > 0 else 0
将S输出，即为我们给出的输入习惯相似度。

## Demo系统
我们基于这个算法做的Demo系统，链接：
https://www.keystroke.cn/keytrace/toSignup.action

使用JavaScript作为键盘事件采集，后台用Java实现的算法。

一起完成这个的小伙伴的github:
[https://github.com/WindInWillows](https://github.com/WindInWillows)
[https://github.com/hitcxy](https://github.com/hitcxy)
[https://github.com/s65b40](https://github.com/s65b40)
[https://github.com/chuxiuhong](https://github.com/chuxiuhong) //我