
本来这个技术含量不足以写进博客的，不过想想好久不写博客都快把markdown语法忘了（汗颜），之前做的信安比赛的项目未来会写一篇总结。

代码比较短，直接就着代码加注释讲吧：
```python
import requests
import re
import os
import threading

class Caoliu:
    def __init__(self):
        '''
        初始化定义一个请求头，后面省的重复定义。同时创建存种子的文件夹
        '''
        self.header_data = {
            'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8',
            'Accept-Encoding': '',
            'Accept-Language': 'zh-CN,zh;q=0.8',
            'Cache-Control': 'max-age=0',
            'Connection': 'keep-alive',
            'Host': 'www.t66y.com',
            'Upgrade-Insecure-Requests': '1',
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/59.0.3071.115 Safari/537.36',
        }
        if "torrent_dir" not in os.listdir(os.getcwd()):
            os.makedirs("torrent_dir")

    def download_page(self, url):
        '''
        针对草榴的第三级页面的方法，负责下载链接
        '''
        header_data2 = {
            'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8',
            'Accept-Encoding': 'gzip, deflate',
            'Accept-Language': 'zh-CN,zh;q=0.8',
            'Cache-Control': 'max-age=0',
            'Connection': 'keep-alive',
            'Host': 'rmdown.com',
            'Referer': 'http://www.viidii.info/?http://rmdown______com/link______php?' + url.split("?")[1],
            'Upgrade-Insecure-Requests': '1',
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/59.0.3071.115 Safari/537.36'
        }
        try:
            download_text = requests.get(url, headers=header_data2).text
            p_ref = re.compile("name=\"ref\" value=\"(.+?)\"")#点击下载时会有表单提交，几个参数都是页面内hidden属性的值，把他们先提取出来
            p_reff = re.compile("NAME=\"reff\" value=\"(.+?)\"")
            ref = p_ref.findall(download_text)[0]
            reff = p_reff.findall(download_text)[0]
            r = requests.get("http://www.rmdown.com/download.php?ref="+ref+"&reff="+reff+"&submit=download")
            with open("torrent_dir\\" + ref + ".torrent", "wb") as f:
                f.write(r.content) #下载种子到文件
        except:
            print("download page " + url + " failed")

    def index_page(self, fid=2, offset=1):
        '''
        针对草榴的第一级页面(浏览帖子题目的页面)
        '''
        p = re.compile("<h3><a href=\"(.+?)\"")
        try:
            tmp_url = "http://www.t66y.com/thread0806.php?fid=" + str(fid) + "&search=&page=" + str(offset)
            r = requests.get(tmp_url)
            for i in p.findall(r.text):
                self.detail_page(i)

        except:
            print("index page " + str(offset) + " get failed")

    def detail_page(self, url):
        '''
        针对具体一个帖子，提取其中的绿色链接(给网盘链接的太不厚道了)
        '''
        p1 = re.compile("(http://rmdown.com/link.php.+?)<")
        p2 = re.compile("(http://www.rmdown.com/link.php.+?)<")
        base_url = "http://www.t66y.com/"
        try:
            r = requests.get(url=base_url + url, headers=self.header_data)
            url_set = set()
            for i in p1.findall(r.text):
                url_set.add(i)
            for i in p2.findall(r.text):
                url_set.add(i)
            url_list = list(url_set)
            for i in url_list:
                self.download_page(i)
        except:
            print("detail page " + url + " get failed")

    def start(self, type, page_start=1, page_end=10, max_thread_num=10):
        '''
        启动方法，其中type参数负责类型
        下载类型 | type
        -------- | -------
        亚洲无码 | yazhouwuma
        亚洲有码 | yazhouyouma
        欧美原创 | oumeiyuanchuang
        动漫原创 | dongmanyuanchuang
        国产原创 | guochanyuanchuang
        中字原创 | zhongziyuanchuang
        page_start,page_end代表起始页和终止页 max_thread_num代表允许程序使用的最大线程数
        '''
        if type == "yazhouwuma":
            fid = 2
        elif type == "yazhouyouma":
            fid = 15
        elif type == "oumeiyuanchuang":
            fid = 4
        elif type == "dongmanyuanchuang":
            fid = 5
        elif type == "guochanyuanchuang":
            fid = 25
        elif type == "zhongziyuanchuang":
            fid = 26
        else:
            raise ValueError("type wrong!")
        max_thread_num = min(page_end-page_start+1,max_thread_num)
        thread_list = []
        for i in range(page_start, page_end + 1):
            thread_list.append(threading.Thread(target=self.index_page, args=(fid, i,)))
        for t in thread_list:
            t.start()
            while True:
                if (len(threading.enumerate()) < max_thread_num):
                    break
if __name__ == "__main__":
    c = Caoliu()
    c.start(type="dongmanyuanchuang",page_start=1,page_end=20,max_thread_num=50)
```
下载后如下：
![image](/static/1024.png)

具体的使用方法，github上的readme更详尽
项目的github地址: https://github.com/chuxiuhong/spider1024