
-------

小学期的《信号与系统》课，要求写一个频率计数器，下面是我个人理解的频率计数

**傅里叶变换的代码**：
```python
# coding=utf-8
import numpy as np
from scipy.io import wavfile
import matplotlib.mlab as mlab
import matplotlib.pyplot as plt


class FrequencyCounter():
    def loaddata(self, filename):
        try:
            samplerate, channels = wavfile.read(filename)
            self.data = np.mean(channels, axis=1)
        except:
            raise ValueError, 'Data Error'

    def fft(self, windowsize=4096, samplerate=44100, overlapratio=0.5):
        try:
            self.res = plt.specgram(self.data,
                                    NFFT=windowsize,
                                    Fs=samplerate,
                                    window=mlab.window_hanning,
                                    noverlap=int(windowsize * overlapratio))[0]
            #傅里叶变换，参数是滑动窗口大小和样例频率

            from numpy.core.umath_tests import inner1d #计算内积
            for i in xrange(len(self.res)):
                self.res[i] = inner1d(self.res[i], self.res[i])
            #plt.plot([x for x in xrange(len(self.res))],self.res)
        except:
            raise ValueError, 'No Data for FFT'

    def mainfrequency(self):
        def compare(a, b):
            return int(a[0][0] < b[0][0])

        sortlist = [i for i in range(len(self.res))]
        for i in range(len(sortlist)):
            sortlist[i] = (self.res[i], i)
        sortlist.sort(lambda x, y: cmp(sum(x[0]), sum(y[0]))) #按照内积大小结果排序
        #for i in sortlist[:200]:
            #print i[1]
        return sortlist[:5]

    def draw(self):
        '''
        画图，为GUI提供图片
        '''
        #plt.figure(figsize=(8,4))
        plt.plot([i for i in xrange(len(self.data))], self.data)
        plt.title(u'音频信号波形',fontproperties='SimHei')
        #plt.show()
        plt.savefig('wave.jpg',dpi=70)
        plt.cla()
        plt.plot([i for i in xrange(len(self.res))], self.res)
        plt.title(u'音频信号频谱分析',fontproperties='SimHei')
        #plt.show()
        plt.savefig('frequency.jpg',dpi=70)

if __name__ == '__main__':
    p = FrequencyCounter()
    p.loaddata('python-audio\\output2.wav')
    p.fft()
    p.draw()

```



**PyQt4开发GUI的代码**：

```python
# -*- coding: utf-8 -*-

from PyQt4.QtGui import *
from PyQt4.QtCore import *
import sys
import frequency_counter2
p = frequency_counter2.FrequencyCounter()
class UI(QDialog):
    def __init__(self, parent=None):
        super(UI, self).__init__(parent)
        self.txt = QLineEdit()
        self.bt_ad = QPushButton(u"选择路径")
        self.bt_a = QPushButton(u"分析")
        self.txtout = QTextEdit() 
        self.pic1 = QLabel()
        self.pic2 = QLabel()
        lay = QGridLayout() #网格布局
        lay.addWidget(self.txt, 1, 1)
        lay.addWidget(self.bt_ad, 1, 2)
        lay.addWidget(self.bt_a, 2, 1, 1, 2)
        lay.addWidget(self.pic1, 4, 1, 3, 3)
        lay.addWidget(self.pic2, 7, 1, 3, 3)
        self.setLayout(lay)
        self.connect(self.bt_a, SIGNAL("clicked()"), self.analy)
        self.connect(self.bt_ad, SIGNAL("clicked()"), self.addr)

    def analy(self):
        #self.txtout.setText("test")
        pix1 = QPixmap("wave.jpg")
        pix2 = QPixmap("frequency.jpg")
        self.pic1.setPixmap(pix1)
        self.pic2.setPixmap(pix2)

    def addr(self):
        fname = QFileDialog.getOpenFileName(self, 'Open file')
        print fname
        p.loaddata(fname)
        p.fft()
        p.draw()
        self.txt.setText(fname)


if __name__ == "__main__":
    app = QApplication(sys.argv)
    ui = UI()
    ui.setWindowTitle(u"音频信号频率分析")
    ui.show()
    app.exec_()

```

**运行时界面**：

![image1](/static/fft111.png)
![image2](/static/fft222.png)