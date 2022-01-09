
直方图均衡化是对于一幅图像，其具有多个灰度等级的像素，我们尽可能让这些灰度等级出现频率的概率密度函数趋近于常数。这么做的意义在哪里？当一幅图像比较暗的时候，灰度等级绝大部分处于低等级的状态，那么由于我们使灰度等级频率的概率密度函数尽可能趋向于常数，即尽可能保证在各个灰度等级出现频率一样，我们认为此时应该有更高的对比度，展示的细节更加细腻。在图像处于过亮的情况下也可以得出相近结论。

举个例子，来自于《数字图像处理》第三章
有大小为64 * 64 像素的3比特数字图像的灰度分布和直方图值如下：


![imagexx](/static/his_table.png)


下面我们计算原本的灰度等级各自映射到哪个等级。
$s_0 = 7\Sigma_{j=0}^0p_r(r_0) = 7 * p_r(r_0) = 1.33$
$s_1 = 7\Sigma_{j=0}^1p_r(r_0) = 7 * p_r(r_0) + 7 * p_r(r_1) = 3.08$
则$s_0$四舍五入得1（实际计算机中灰度等级只能是离散的）
$s_1$四舍五入得3。
其他每个灰度等级的映射都类似，可以分别计算出均衡化后每个原灰度等级应该映射到的新的灰度等级。

![image1](/static/lena_gray.jpg)

Lena的灰度图

![image2](/static/lena_gray_his.jpg)

Lena灰度图做直方图均衡化后

![image3](/static/lena.jpg)

Lena的彩色图

![image4](/static/lena_his.jpg)

Lena彩色图做直方图均衡化后

对于彩色图像的直方图均衡化，可以考虑使用R,G,B三个通道分别均衡化，然后将三个通道合在一起。但这样有可能会改变色调。
下面是直方图均衡化的python实现，依赖PIL包
```python
def his_equ(img, outfile,level=256,mode='RGB'):
    '''

    :param img: Image.open打开的文件句柄
    :param outfile: 输出文件的文件名
    :param level:灰度等级，彩色图是每个通道对应的等级数
    :param mode:'rgb'为彩色模式，'gray'为灰度图
    :return: 按照输出文件路径保存均衡化之后的图片
    '''
    if mode == 'RGB' or mode == 'rgb':
        r, g, b = [], [], []
        width, height = img.size[0], img.size[1]
        sum_pix = width * height
        pix = img.load()
        for x in range(width):
            for y in range(height):
                r.append(pix[x, y][0])
                g.append(pix[x, y][1])
                b.append(pix[x, y][2])
        r_c = dict(Counter(r))
        g_c = dict(Counter(g))
        b_c = dict(Counter(b))
        r_p,g_p,b_p = [],[],[]

        for i in range(level):
            if r_c.has_key(i):
                r_p.append(float(r_c[i]) / sum_pix)
            else:
                r_p.append(0)
            if g_c.has_key(i):
                g_p.append(float(g_c[i])/sum_pix)
            else:
                g_p.append(0)
            if b_c.has_key(i):
                b_p.append(float(b_c[i])/sum_pix)
            else:
                b_p.append(0)
        temp_r,temp_g,temp_b = 0,0,0
        for i in range(level):
            temp_r += r_p[i]
            r_p[i] = int(temp_r * (level-1))
            temp_b += b_p[i]
            b_p[i] = int(temp_b *(level-1))
            temp_g += g_p[i]
            g_p[i] = int(temp_g*(level -1))
        new_photo = Image.new('RGB',(width,height))
        for x in range(width):
            for y in range(height):
                new_photo.putpixel((x,y),(r_p[pix[x,y][0]],g_p[pix[x,y][1]],b_p[pix[x,y][2]]))
        new_photo.save(outfile)
    elif mode == 'gray' or mode == 'GRAY':
        width, height = img.size[0], img.size[1]
        sum_pix = width * height
        pix = img.load()
        pb = []
        for x in range(width):
            for y in range(height):
                pb.append(pix[x,y])
        pc = dict(Counter(pb))
        pb = []
        for i in range(level):
            if pc.has_key(i):
                pb.append(float(pc[i]) / sum_pix)
            else:
                pb.append(0)
        temp = 0
        for i in range(level):
            temp += pb[i]
            pb[i] = int(temp * (level-1))
        new_photo = Image.new('L',(width,height))
        for x in range(width):
            for y in range(height):
                new_photo.putpixel((x,y),pb[pix[x,y]])
        new_photo.save(outfile)
if __name__ == '__main__':
    ppp = Image.open('Lena.jpg','r')
    his_equ(ppp,'lena_his.jpg')
```