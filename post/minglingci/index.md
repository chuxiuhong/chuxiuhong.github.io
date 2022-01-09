
想要容易理解核心的特征计算的话建议先去看看我之前的听歌识曲的文章，传送门:[https://chuxiuhong.com/post/shazam/](https://chuxiuhong.com/post/shazam/)

本文主要是实现了一个简单的命令词识别程序，算法核心一是提取音频特征，二是用DTW算法进行匹配。当然，这样的代码肯定不能用于商业化，大家做出来玩玩娱乐一下还是不错的。

转载请保留本文链接，谢谢。

-----------------
**设计思路**
----------------

就算是个小东西，我们也要先明确思路再做。音频识别，困难不小，其中提取特征的难度在我听歌识曲那篇文章里能看得出来。而语音识别难度更大，因为音乐总是固定的，而人类说话常常是变化的。比如说一个“芝麻开门”，有的人就会说成“芝麻\~~开门”，有的人会说成“芝麻开门\~~”。而且在录音时说话的时间也不一样，可能很紧迫的一开始录音就说话了，也可能不紧不慢的快要录音结束了才把这四个字说出来。这样难度就大了。

算法流程：
![image00](/static/minglingci_001.png)

-----------
**特征提取**
----------

和之前的听歌识曲一样，同样是将一秒钟分成40块，对每一块进行傅里叶变换，然后取模长。只是这不像之前听歌识曲中进一步进行提取峰值，而是直接当做特征值。
看不懂我在说什么的朋友可以看看下面的源代码，或者看听歌识曲那篇文章。

-------
**DTW算法**
--------

DTW，Dynamic Time Warping，动态时间归整。算法解决的问题是将不同发音长短和位置进行最适合的匹配。

算法输入两组音频的特征向量: A:[fp1,fp2,fp3,......,fpM1] B:[fp1,fp2,fp3,fp4,.....fpM2]
A组共有M1个特征，B组共有M2个音频。每个特征向量中的元素就是之前我们将每秒切成40块之后FFT求模长的向量。计算每对fp之间的代价采用的是欧氏距离。

设D(fpa,fpb)为两个特征的距离代价。

那么我们可以画出下面这样的图

![image01](/static/minglingci_002.png)

我们需要从(1,1)点走到(M1,M2)点，这会有很多种走法，而每种走法就是一种两个音频位置匹配的方式。但我们的目标是走的总过程中代价最小，这样可以保证这种对齐方式是使我们得到最接近的对齐方式。

我们这样走：首先两个坐标轴上的各个点都是可以直接计算累加代价和求出的。然后对于中间的点来说D(i,j) = Min{D(i-1,j)+D(fpi,fpj) , D(i,j-1)+D(fpi,fpj) , D(i-1,j-1) + 2 * D(fpi,fpj)}
为什么由(i-1,j-1)直接走到(i,j)这个点需要加上两倍的代价呢？因为别人走正方形的两个直角边，它走的是正方形的对角线啊

按照这个原理选择，一直算到D(M1,M2)，这就是两个音频的距离。

![image01](/static/minglingci_003.png)

![image01](/static/minglingci_004.png)

![image01](/static/minglingci_005.png)



---------------
**源代码和注释**
-----------

```python
# coding=utf8
import os
import wave
import dtw
import numpy as np
import pyaudio

def compute_distance_vec(vec1, vec2):
    return np.linalg.norm(vec1 - vec2) #计算两个特征之间的欧氏距离

class record():
    def record(self, CHUNK=44100, FORMAT=pyaudio.paInt16, CHANNELS=2, RATE=44100, RECORD_SECONDS=200,
               WAVE_OUTPUT_FILENAME="record.wav"):
        #录歌方法
        p = pyaudio.PyAudio()
        stream = p.open(format=FORMAT,
                        channels=CHANNELS,
                        rate=RATE,
                        input=True,
                        frames_per_buffer=CHUNK)
        frames = []
        for i in range(0, int(RATE / CHUNK * RECORD_SECONDS)):
            data = stream.read(CHUNK)
            frames.append(data)
        stream.stop_stream()
        stream.close()
        p.terminate()
        wf = wave.open(WAVE_OUTPUT_FILENAME, 'wb')
        wf.setnchannels(CHANNELS)
        wf.setsampwidth(p.get_sample_size(FORMAT))
        wf.setframerate(RATE)
        wf.writeframes(''.join(frames))
        wf.close()

class voice():
    def loaddata(self, filepath):
        try:
            f = wave.open(filepath, 'rb')
            params = f.getparams()
            self.nchannels, self.sampwidth, self.framerate, self.nframes = params[:4]
            str_data = f.readframes(self.nframes)
            self.wave_data = np.fromstring(str_data, dtype=np.short)
            self.wave_data.shape = -1, self.sampwidth
            self.wave_data = self.wave_data.T #存储歌曲原始数组
            f.close()
            self.name = os.path.basename(filepath)  # 记录下文件名
            return True
        except:
            raise IOError, 'File Error'

    def fft(self, frames=40):
        self.fft_blocks = [] #将音频每秒分成40块，再对每块做傅里叶变换
        blocks_size = self.framerate / frames
        for i in xrange(0, len(self.wave_data[0]) - blocks_size, blocks_size):
            self.fft_blocks.append(np.abs(np.fft.fft(self.wave_data[0][i:i + blocks_size])))
    @staticmethod
    def play(filepath):
        chunk = 1024
        wf = wave.open(filepath, 'rb')
        p = pyaudio.PyAudio()
        # 播放音乐方法
        stream = p.open(format=p.get_format_from_width(wf.getsampwidth()),
                        channels=wf.getnchannels(),
                        rate=wf.getframerate(),
                        output=True)
        while True:
            data = wf.readframes(chunk)
            if data == "": break
            stream.write(data)
        stream.close()
        p.terminate()
if __name__ == '__main__':
    r = record()
    r.record(RECORD_SECONDS=3, WAVE_OUTPUT_FILENAME='record.wav')
    v = voice()
    v.loaddata('record.wav')
    v.fft()
    file_list = os.listdir(os.getcwd())
    res = []
    for i in file_list:
        if i.split('.')[1] == 'wav' and i.split('.')[0] != 'record':
            temp = voice()
            temp.loaddata(i)
            temp.fft()
            res.append((dtw.dtw(v.fft_blocks, temp.fft_blocks, compute_distance_vec)[0],i))
    res.sort()
    print res
    if res[0][1].find('open_qq') != -1:
        os.system('C:\program\Tencent\QQ\Bin\QQScLauncher.exe') #我的QQ路径
    elif res[0][1].find('zhimakaimen') != -1:
        os.system('chrome.exe')#浏览器的路径，之前已经被添加到了Path中了
    elif res[0][1].find('play_music') != -1:
        voice.play('C:\data\music\\audio\\audio\\ (9).wav') #播放一段音乐
    # r = record()
    # r.record(RECORD_SECONDS=3,WAVE_OUTPUT_FILENAME='zhimakaimen_09.wav')

```

事先可以先用这里的record方法录制几段命令词，尝试用不同语气说，不同节奏说，这样可以提高准确度。然后设计好文件名，根据匹配到的最接近音频的文件名就可以知道是哪种命令，进而自定义执行不同的任务

