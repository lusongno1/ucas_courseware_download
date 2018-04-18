> 国科大课程网站于18年4月9日进行了换血升级，[李博伟](http://libowei.net/)学长原有的课件下载工具不能用了（不知道他后来更新没，可以follow他的[github](https://github.com/libowei1213/CoursewareDownload)）。特此新写一份脚本，供广大校友使用。

#### 初衷
- 最初的想法就是因为上的课比较多，每个老师上传的课件都很多，手动下载很费事；
- 国科大课程网站下载课件只能一个一个下载，如果有文件夹，还要先点进去，下载十分麻烦；
- 老师上传课件不进行通知，想要知道是否有新课件，你得一个一个课程点进去看有没有课件更新；
- 并不想弄得太麻烦，主要是想知道哪些课程有课件更新，有更新的，将新文件下载，并且拷贝到我的课程文件夹，进行覆盖更新。
#### 用法
新建user.txt文件，第一行存放你登录校园云平台的账号和密码，账号密码之间用空格隔开，运行程序（可以用shell运行）即可。

建议新开辟一片空间，用以存放程序下载的文件，再将下载的文件拷贝到你想存放的地方存档。每隔一段时间，可以在存放程序下载文件目录下运行一下脚本，
看是否有文件更新，若有文件更新，就将更新的文件拷贝至课程文件存档处。

下载过的文件（本地已存在文件）不会再下载，新文件会下载，下载结束后会打印总的摘要，即哪些科目下载了哪些新文件。

####脚本如下
```
# -*- coding: utf-8 -*-
"""
Created on Sat Apr 14 15:07:29 2018

@author: Lu Song
"""

import requests
import re
import os
import sys
from bs4 import BeautifulSoup
import json
from urllib.parse import unquote


def save_html(html):
    f = open('test.html','w',encoding='utf-8')
    f.write(html)
    f.close
    
def UCAS_login():
    try:
        config = open("user.txt", encoding='utf-8')
        line = config.readline().split()
        username = line[0]
        password = line[1]
    except IOError as e:
        print(e)
    session = requests.session()
    login_url = 'http://onestop.ucas.ac.cn/Ajax/Login/0'#提交信息地址，这个地址不需要验证码
    headers=  {
                'Host': 'onestop.ucas.ac.cn',
                "Connection": "keep-alive",
                'Referer': 'http://onestop.ucas.ac.cn/home/index',
                'X-Requested-With': 'XMLHttpRequest',
                "User-Agent": "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/52.0.2743.116 Safari/537.36",
            }
    post_data = {
                "username": username,
                "password": password,
                "remember": 'checked',
            }
    html = session.post(login_url, data=post_data, headers=headers).text
    res = json.loads(html)#登录地址是一回事，提交数据地址是一回事，返回的地址是一回事，这里打开返回的地址
    html = session.get(res['msg']).text
    print('登录系统成功……')
  #  save_html(html)
    return session



def getinto_courseSite(session):
    url = "http://sep.ucas.ac.cn/portal/site/16/801"
    h_k = session.get(url)
 #   save_html(h_k.text)
    key = re.findall(r'"http://course.ucas.ac.cn/portal/plogin\?Identity=(.*)"', h_k.text)[0]
    #打开选课系统
    url = "http://course.ucas.ac.cn/portal/plogin/main/index?Identity=" + key
    h = session.get(url)
#    save_html(h.text)
    page = h
    return page
 
    
def get_courseInfo(session,courseSite):
    course_list = []
    mycourseBS = BeautifulSoup(courseSite.text,"lxml")
    mycourseBS.find_all('a',{"class":'Mrphs-toolsNav__menuitem--link'})
    url_mycourse = mycourseBS.find_all('a',{"class":'Mrphs-toolsNav__menuitem--link'})[0]
    url_mycourse = url_mycourse["href"]
    coursePage = session.get(url_mycourse)
    save_html(coursePage.text)
    coursePageBS = BeautifulSoup(coursePage.text,"lxml")
    Course_info = coursePageBS.find_all('li',{"class":"fav-sites-entry"})
    del Course_info[-1]
 #   del Course_info[1:-1]
    length = len(Course_info)
#    print(Course_info.attrs)
    for i in range(0,length-1):
##        print(s)
        info = Course_info[i]
        tag = info.div.a
        courseName = tag["title"]
        courseUrl = tag["href"]
        course_list.append((courseName,courseUrl))       
    return course_list

# 下载课件
def download(url, fileName, className, session):
    # \xa0转gbk会有错
    fileName = fileName.replace(u"\xa0", " ").replace(u"\xc2", "")
    # 去掉不合法的文件名字符
    fileName = re.sub(r"[/\\:*\"<>|?]", "", fileName)
    className = re.sub(r"[/\\:*\"<>|?]", "", className)

    dir = os.getcwd() + "/" + className
    file = os.getcwd() + "/" + className + "/" + fileName
    # 没有课程文件夹则创建
    if not os.path.exists(dir):
        os.mkdir(dir)
    # 存在该文件，返回
    if os.path.exists(file):
        print("%s已存在，就不下载了" % fileName)
        return 0
    print("开始下载%s..." % fileName)
    s = session.get(url)
    with open(file, "wb") as data:
        data.write(s.content)
    return 1
        
        


def getClass(currentClass, url, session, data):
    if data != None:
        s = session.post(url, data=data)
        s = session.get(url)
        
    else:
        s = session.get(url)
#    save_html(s.text)
    print('获取资源列表，寻找资源链接……')
    downloadfile = []
    resourceList = BeautifulSoup(s.text, "html.parser").findAll("tr",{"class":''}) 
    for  ress in resourceList:
        if ress.find("td") == None:
            continue
        if ress.find("input") == None:
            continue
#        def named_a_and_has_href(tag):
#            return  tag.has_attr('href')     
#  #      res = res.find_all("a")
#        res = res.find_all("named_a_and_has_href")
#        resUrl = res.get("href")[0]
#        # 文件夹
        flag = 0
        def parent_is_td(tag): 
            return  tag.parent.name=='td' and tag.name == 'a'     
        res = ress.find(parent_is_td)
        resUrl = res.get("href")
        if  res.get("title") == "打开此文件夹":
            print('文件夹展开……')
            path = ress.find("td", {"headers": "checkboxes"}).input.get("value")
            save_html(s.text)
            sct = BeautifulSoup(s.text,"lxml").find(attrs={"name":"sakai_csrf_token"})#
            sct = sct.get("value")
            data = {'source': '0', 'collectionId': path, 'navRoot': '', 'criteria': 'title', 'sakai_action': 'doNavigate',
                    'rt_action': '', 'selectedItemId': '', 'sakai_csrf_token': sct}
            # print (path)
            urlNew = BeautifulSoup(s.text, "html.parser").find("form").get("action")
#            testurl = session.get('http://course.ucas.ac.cn/portal/site/142481/tool/93af7d38-f125-4bd5-9200-bcd7b7824514')
#            save_html(testurl.text)
            print('文件夹展开链接为%s' %urlNew)
            getClass(currentClass, urlNew, session, data)
        # 有版权的文件，构造下载链接
        elif res.get("href") == "#":
            print('下载版权文件中……')
            jsStr = res.get("onclick")
            reg = re.compile(r"openCopyrightWindow\('(.*)','copyright")
            match = reg.match(jsStr)
            if match:
                resUrl = match.group(1)
                resName = resUrl.split("/")[-1]
                resName = unquote(resName)#, encoding="GBK")
                flag = download(resUrl, resName, currentClass, session)
        # 课件可以直接下载的
        else:
            print('课件直接下载中……')
            resName = resUrl.split("/")[-1]
            resName = unquote(resName)#, encoding="GBK")
            flag = download(resUrl, resName, currentClass, session)
        if flag:
            downloadfile.append(resName)
    return downloadfile
            
def download_courseware(courseInfo,session):
    summaryInfo = [] 
    for course in courseInfo:
        currentClassName = course[0]
        url = course[1]
        h = session.get(url)
      #  save_html(h.text)
        h_bs = BeautifulSoup(h.text,"lxml")
        url = h_bs.find_all(title="资源 - 上传、下载课件，发布文档，网址等信息")[0].get("href")
   #     h = session.get(url)
  #      save_html(h.text)
        print('进入%s课程主页中……' %currentClassName)
        downloadfile = getClass(currentClassName, url, session, None)
        if downloadfile!=[]:
            summaryInfo.append((currentClassName,downloadfile))
    return summaryInfo


if __name__ == '__main__':
    print(u"""#---------------------------------------  
#   程序：国科大课件批量自动下载程序 
#   版本：0.5  
#   作者：陆嵩  
#   日期：2018-4-15  
#   语言：Python 3.6  
#   操作：记事本第一行写入账号密码，空格隔开。
#   运行脚本即可，建议进行课件的全下载，再次运行程序即可知哪些科目新下载了文件  
#   功能：可以知道所上科目有些课程更新了课件，并自动下载本地端未有课件  
#---------------------------------------
""")
    session = UCAS_login()
    print('进入课程网站中……')
    courseSite = getinto_courseSite(session)
    print('获取假如课程网站的课程中……')
    courseInfo = get_courseInfo(session,courseSite)
    print('开始下载……')
    summary = download_courseware(courseInfo,session)
    if summary != []:
        print("\n\n\n\n\n课件更新情况汇总：\n")
        for s in summary:
            print("\n\n%s课程有更新："%s[0])
            files = s[1];
            for m in files:
                print("%s"%m)
            
    else:
        print("\n\n\n没有新课件下载……")
        

    
    
    
```


> 特别需要声明的是，该程序参考了李博伟代码的`getClass`方法，直接引用了其`download_courseware`方法，特此声明。
