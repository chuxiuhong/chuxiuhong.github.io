
这个项目实现了：
a) 网站过滤：允许/不允许访问某些网站；
b) 用户过滤：支持/不支持某些用户访问外部网站；
c) 网站引导：将用户对某个网站的访问引导至一个模拟网站（钓
鱼）。
d) 缓存功能：要求能缓存原服务器响应的对象，并能够通过修改请求报文（添加 if-modified-since头行），向原服务器确认缓存对象是否是最新版本

首先，先要把django包内的C:\Python27\Lib\site-packages\django\core\handlers\base.py 中的^$改为.* 。（共有两处需要修改），以此来保证能让所有的url目标都传到views里面的函数中。
如图
![image1](/static/proxy1.png)

然后构建一个django项目，可以不带有admin模块，然后建立一个新的app
博主构建的项目结构如下，其中Cache.py是一会再创建的
![image2](/static/proxy2.png)

下面是models.py的代码

```python
#coding=utf8
from django.db import models


class fish(models.Model):
    #钓鱼规则表
    user_ip = models.CharField(max_length=15) #客户ip
    forbidden_host = models.CharField(max_length=100)  #禁止的host
    fish_url = models.CharField(max_length=100)  #跳转的网站链接


class firewall(models.Model):
    #黑名单规则表
    user_ip = models.CharField(max_length=15)   #客户ip
    forbidden_host = models.CharField(max_length=100)  #禁止的host


class cache(models.Model):
    #缓存表
    timestamp = models.CharField(max_length=80) #时间戳
    name = models.CharField(max_length=150) #文件名对应的url
    content_type = models.CharField(max_length=50)  #内容类型
    content = models.TextField()  #内容


def check_if_replace(user_ip, host):
    user_list = firewall.objects.filter(user_ip=user_ip).all()  #先到黑名单中查找
    for user in user_list:
        if user.forbidden_host in host or host == '*':
            return (1, '<h1>You have been forbidden!</h1>') #返回禁止页
    general = firewall.objects.filter(user_ip='*').all()
    for i in general:
        if i.forbidden_host in host:
            return (1, '<h1>You have been forbidden!</h1>')
    fish_list = fish.objects.filter(user_ip=user_ip).all()  #然后到钓鱼规则中查找
    for fisher in fish_list:
        if fisher.forbidden_host in host:
            return (-1, fisher.fish_url)
    return (0, host)

```

这一部分是用来做规则的存储和缓存的存储，migrate之后各表如下所示
![image3](/static/proxy3.png)
这张表用来做禁止规则的配置，可以用'*'作为通配符，实现所有用户对单一网站和对某用户做屏蔽。

![image4](/static/proxy4.png)
这张表用来做钓鱼规则的配置，用户转到指定的目标

![image5](/static/proxy5.png)
这张表作为缓存的实现


views.py的代码：
```python
#coding=utf8
import re
from contextlib import closing
from django.http import HttpResponse,HttpResponseRedirect
from Cache import *
def home(request):
    url = request.path[1:].split('/')
    url = url[0] + '//' + url[1] + '/'
    url = request.path[1:].replace(':/', '://')  #获得目标url
    host = request.get_host()
    method = request.method
    if request.META.has_key('HTTP_X_FORWARDED_FOR'):
        ip = request.META['HTTP_X_FORWARDED_FOR']  #获取客户端ip
    else:
        ip = request.META['REMOTE_ADDR']
    regex = re.compile('^HTTP_')
    headers = dict((regex.sub('', header), value) for (header, value)
                   in request.META.items() if header.startswith('HTTP_'))
    if len(request.REQUEST.items()) > 0:
        url += '?'
        for (i, j) in request.REQUEST.items():
            url += str(i) + '=' + str(j) + '&'
        url = url[:-1]  #将带参数的get请求恢复成原始链接状态
    check_tuple = check_if_replace(ip, host)
    if check_tuple[0] == 1:
        return HttpResponse(check_tuple[1])  #如果符合黑名单规则，返回禁止页
    elif check_tuple[0] == -1:
        url = check_tuple[1]
        return HttpResponseRedirect(url)   #如果符合钓鱼规则，那么重定向到指定的网站
    cacher = Cache(url)
    if cacher.check_cache():      #检查是否有与目标网站一致的资源
        res = cacher.get()
        return HttpResponse(res[0], content_type=res[1])  #将缓存的内容返回
    else:
        with closing(requests.request(method, url, headers=headers, data=request.POST, stream=True)) as r:
            cacher.update(bytes(r.content), content_type=r.headers['content-type'])  #更新缓存
            return HttpResponse(bytes(r.content), status=r.status_code, content_type=r.headers['content-type'])  #返回资源

```

Cache.py的代码
```python
#coding=utf8
import requests
from models import *
import time
class Cache():
    def __init__(self, url):
        self.url = url

    def check_cache(self):
        '''
        检查是否有一致的缓存
        :return: Boolean型
        '''
        try:
            f = cache.objects.get(name=self.url)
            headers = {'If-Modified-Since': f.timestamp}
            if requests.get(self.url, headers=headers).status_code == 304:
                return True
            else:
                return False
        except:
            return False

    def update(self, content, content_type):
        '''
        更新缓存
        :param content: 内容
        :param content_type: 内容类型
        :return:
        '''
        try:
            f = cache.objects.get(name=self.url)
            f.content = content
            f.content_type = content_type
            f.name = self.url
            t = time.asctime().split()
            f.timestamp = t[0] + ', '+t[2] + ' '+t[1] +' '+t[4]+' '+t[3] + 'GMT'
            f.save()
        except:
            f = cache()
            f.content = content
            f.name = self.url
            t = time.asctime().split()
            f.timestamp = t[0] + ', ' + t[2] + ' ' + t[1] + ' ' + t[4] + ' ' + t[3] + ' GMT'
            f.content_type = content_type
            f.save()

    def get(self):
        #获取缓存
        f = cache.objects.get(name=self.url)
        return (f.content, f.content_type)
```

最后修改urls.py
```python
from django.conf.urls import include, url
# from django.contrib import admin
from server.views import *
urlpatterns = [
    # url(r'^admin/', include(admin.site.urls)),
    url('.*',home),
]
```

为了防止post的时候受干扰，把settings.py中的中间件去掉csrf，剩下的如图
![image6](/static/proxy6.png)


下面启动项目，监听8000端口
![image7](/static/proxy7.png)

把浏览器设置代理为127.0.0.1:8000
![image8](/static/proxy8.png)


打开博客园首页，正常显示
![image9](/static/proxy9.png)

访问人人网，由于黑名单的表中有这条规则，返回我们写的禁止页
![image10](/static/proxy10.png)

访问爱奇艺，由于钓鱼规则中配置了将爱奇艺引导到博客园，返回的是对博客园的重定向
![image11](/static/proxy11.png)


到此就完成了一个简单的HTTP代理服务器