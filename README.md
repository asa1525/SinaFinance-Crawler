# 第二十一天有回想起来记录了

### 原先预定的爬虫工作终于引来了终点，尽管最后的去重工作还没有做到真正的完善，但从现在开始我会写下我所做到的内容。
> 我的任务就是爬取新浪财经每只股票资讯页面下所有的新闻，将日期，标题，来源，内容，关键词，地址和股票代码保存至csv和数据库里面。

# 爬取新浪财经资讯


```python
import requests
import time
import csv
import codecs
import re
import random
from selenium import webdriver
import pandas as pd
from bs4 import BeautifulSoup
from lxml import etree
import scrapy
from multiprocessing import cpu_count
from multiprocessing import Process, Queue
```

## 先使用pandas读取所有的深市沪市股市ID
```python
stock_id = pd.read_csv("stock_id.csv")
```

## 深市和沪市分别保存至sz, hs
```python
sz = stock_id[0: 1384] #[0: 1384]
hs = stock_id[2045: 3402] #[2045: 3402]
```

## 抓取深沪股市资讯列表


```python
def get_url():
    sz_url, hs_url = [], []
    for i in range(0, len(sz)):
        a = 'http://vip.stock.finance.sina.com.cn/corp/view/vCB_AllNewsStock.php?symbol=sz' + sz.SecuCode[i] # 新浪财经资讯后的每一个股票都对应的网址
        sz_url.append(a) #使用append() 方法在列表末尾添加新的地址到sz_url，下面的hs_url理由相同。
    for j in range(2045, 3402):
        b = 'http://vip.stock.finance.sina.com.cn/corp/view/vCB_AllNewsStock.php?symbol=hs' + hs.SecuCode[j]
        sz_url.append(b)
    return sz_url
```


```python
GetUrls = get_url() # 在爬取的时候总会遇到网不好的情况，所以当完整的爬取以后就再备份至GetUrls
```

### 只爬取深市(测试用)


```python
def get_sz_url():
    sz_url, SecuCode_list = [], []
    for i in range(0, len(sz)):
        a = 'http://vip.stock.finance.sina.com.cn/corp/view/vCB_AllNewsStock.php?symbol=sz' + sz.SecuCode[i]
        sz_url.append(a)
    return sz_url
```


```python
#GetUrls = get_sz_url()
```


```python
#GetUrls
```

### 只爬取沪市(测试用)


```python
def get_hs_url():
    hs_url = []
    for j in range(2047, 3402):
        b = 'http://vip.stock.finance.sina.com.cn/corp/view/vCB_AllNewsStock.php?symbol=hs' + hs.SecuCode[j]
        hs_url.append(b)
    return hs_url,SecuCode_list
```

##  获取Stock_id
> 该函数主要实现获取每个股票id，为什么做这件事后面会提到

```python
def get_id():
    sz_list, hs_list = [], []
    for i in range(0, len(sz)):
        a = 'sz' + sz.SecuCode[i]
        Sz_SecuCode = re.findall("(sz\d\d\d\d\d\d)", a) # 使用了正则来找到ID
        sz_list.append(Sz_SecuCode)
    for j in range(2045, 3402):
        b = 'hs' + hs.SecuCode[j]
        Hs_SecuCode = re.findall("(hs\d\d\d\d\d\d)", b)       
        hs_list.append(Hs_SecuCode)
    sz_list.extend(hs_list)
    return sz_list # 深市沪市都返回到sz_list
```

### 获取深市StockID(测试用)


```python
def get_id():
    sz_list, hs_list = [], []
    for i in range(0, len(sz)):
        a = 'sz' + sz.SecuCode[i]
        Sz_SecuCode = re.findall("(sz\d\d\d\d\d\d)", a)
        sz_list.append(Sz_SecuCode)
    return sz_list
```

### 获取沪市StockID(测试用)

## 爬取新闻链接


```python
from Download import request # Download包重新写了一个requests，参考了网上的大神，佩服！后面还会继续贴出来的。
def get_data_news_zhongxin(): #该函数专门负责每一页的超链接新闻
    news_list, URL_list = [], []
    for i in range(0, len(GetUrls)):                                      # 0 is first stock_id
        Pages = GetUrls[i]
        for j in range(0,1):                                                   # catch page1 - 10 firstly，这里只抓了每个股的第一页
            r = request.get(str(Pages) + '&Page=' + str(j), 3) #注意！这里的request就不是requests包里的了，是download包的
            r.encoding='gb2312'                                          #该页面的编码
            selector = etree.HTML(r.text)
            infos = selector.xpath('//div[@class="datelist"]') #使用etree的xpath来获取静态内容
            for info in infos:
                URL = info.xpath('//td/div[1]/ul/a/@href')
                #print(URL)
                URL_list.extend(URL)
    return URL_list
```


```python
gdnz = get_data_news_zhongxin() ###所有的新闻链接
```


```python
gdnz
```




    ['http://finance.sina.com.cn/stock/s/2018-09-14/doc-ihkahyhw8150416.shtml',
     'http://finance.sina.com.cn/chanjing/gsnews/2018-09-11/doc-ihiixyeu6288005.shtml',
     'http://finance.sina.com.cn/roll/2018-09-10/doc-ihivtsyk8575881.shtml',
     'http://finance.sina.com.cn/chanjing/gsnews/2018-09-10/doc-ihivtsyk8131280.shtml',
     'http://finance.sina.com.cn/chanjing/gsnews/2018-09-08/doc-ihivtsyi6893086.shtml',
     'http://finance.sina.com.cn/roll/2018-09-04/doc-ihiqtcap3371000.shtml',
     'http://finance.sina.com.cn/roll/2018-09-04/doc-ihiqtcan8351044.shtml',
     'http://finance.sina.com.cn/roll/2018-09-04/doc-ihiqtcan8059637.shtml',
     'http://finance.sina.com.cn/stock/s/2018-09-04/doc-ihiqtcan7724278.shtml',
     'http://finance.sina.com.cn/stock/s/2018-08-28/doc-ihifuvpi0153076.shtml',
     'http://finance.sina.com.cn/roll/2018-08-28/doc-ihifuvpi0104918.shtml',
     'http://finance.sina.com.cn/stock/s/2018-08-27/doc-ihifuvph3775430.shtml',
     'http://finance.sina.com.cn/stock/gujiayidong/2018-08-24/doc-ihicsiaw3191805.shtml',
     'http://finance.sina.com.cn/roll/2018-08-24/doc-ihicsiaw3121896.shtml',
     'http://finance.sina.com.cn/stock/gujiayidong/2018-08-24/doc-ihicsiaw2072623.shtml',
     'http://finance.sina.com.cn/roll/2018-08-24/doc-ihicsiaw2027530.shtml',
     'http://finance.sina.com.cn/stock/s/2018-08-23/doc-ihicsiav6266837.shtml',
     'http://finance.sina.com.cn/roll/2018-08-22/doc-ihhzsnec2188769.shtml',
     'http://finance.sina.com.cn/roll/2018-08-22/doc-ihhzsnec2329573.shtml',
     'http://finance.sina.com.cn/roll/2018-08-22/doc-ihhzsnec0647787.shtml',
     'http://finance.sina.com.cn/chanjing/gsnews/2018-08-20/doc-ihhxaafy9369754.shtml',
     'http://finance.sina.com.cn/roll/2018-08-19/doc-ihhxaafy3484484.shtml',
     'http://finance.sina.com.cn/roll/2018-08-18/doc-ihhvciix0123652.shtml',
     'http://finance.sina.com.cn/roll/2018-08-17/doc-ihhvciiw3092353.shtml',
     'http://finance.sina.com.cn/stock/s/2018-08-17/doc-ihhvciiw2518447.shtml',
     'http://finance.sina.com.cn/roll/2018-08-17/doc-ihhvciiw2406918.shtml',
     'http://finance.sina.com.cn/stock/s/2018-08-16/doc-ihhvciiv9068870.shtml',
     'http://finance.sina.com.cn/roll/2018-08-16/doc-ihhvciiv8164151.shtml',
     'http://finance.sina.com.cn/stock/s/2018-08-16/doc-ihhvciiv7096151.shtml',
     'http://finance.sina.com.cn/stock/jsy/2018-08-16/doc-ihhtfwqs0343821.shtml',
     'http://finance.sina.com.cn/stock/s/2018-08-16/doc-ihhtfwqs0246167.shtml',
     'http://finance.sina.com.cn/roll/2018-08-16/doc-ihhtfwqs0108893.shtml',
     'http://finance.sina.com.cn/stock/s/2018-08-16/doc-ihhtfwqr9548961.shtml',
     'http://finance.sina.com.cn/stock/s/2018-08-16/doc-ihhtfwqr9478736.shtml',
     'http://finance.sina.com.cn/stock/s/2018-08-15/doc-ihhtfwqr9124771.shtml',
     'http://finance.sina.com.cn/roll/2018-08-15/doc-ihhtfwqr9168400.shtml',
     'http://finance.sina.com.cn/roll/2018-08-15/doc-ihhtfwqr8653818.shtml',
     'http://finance.sina.com.cn/stock/ggzz/2018-08-15/doc-ihhtfwqr8287010.shtml',
     'http://finance.sina.com.cn/roll/2018-08-15/doc-ihhtfwqr8267017.shtml',
     'http://finance.sina.com.cn/roll/2018-08-15/doc-ihhtfwqr8144422.shtml']


## 后面的工作我踩了不少坑，由于也没有及时记录我忘记了一些，这就是告诉自己以后每天都要发文章！

### 保存爬取的新闻链接至csv（方便以后爬取作去重）


```python
###保存urls到csv遇到的一个坑就是csv不是以一行一行的保存的，而是一列一列的。
for i in range(0, len(gdnz)):
    a = gdnz[i]
    with open('sina_info_url.csv', 'a', newline='') as csvfile: # newline的作用就是不以空行保存。
        w = csv.writer(csvfile)
        #print(w)
        w.writerow([a]) ### 我就改了这个地方！！！！ 原本只能输出一行的， 现在终于可以一个一列了！
```

## 将文本与SrockID合并并输出

### 输出StockID至list


```python
gsid = get_id() 
```


```python
gsid
```




    [['sz000001']]



> 这里的print_id()是我遇到的难题之一
> **因为我还要输出股票id到csv，所以我只能通过根据抓取的链接数目来循环输出股票id N次**
```python
### 
def print_id(): 
    id_list = []
    nums = len(gdnz) // len(sz) # 链接总数除以股票总数可以得出需要几次的StockID
    for i in range(0, len(gsid)):
        for j in range(0, nums):
            id_list.extend(gsid[i])
    return id_list                        # 都保存到id_list
```


```python
pid = print_id() #同样保存至pid
```
    




### 输出文本并保存至csv


```python
def get_data_news_content():
    lists, InfoPublDate, InfoSource, InfoTitle, InfoContent = [], [], [], [], []
    
    for i in range(0, len(gdnz)):                                                                    #gdnz是要抓取的新闻链接
        r_content = request.get(gdnz[i], 3)
        r_content.encoding = 'utf-8'
        selector = etree.HTML(r_content.text)
        info_contents = selector.xpath("//div[@class='article']")

        for info in info_contents:
            ###解析时间来员标题关键词文本
            InfoPublDate = info.xpath('//div[@class="date-source"]/span/text()')
            InfoSource = info.xpath('//div[@class="date-source"]/a/text()')
            InfoTitle = info.xpath('//h1/text()')
            InfoKeyword = info.xpath('//div[@id="article-bottom"]/div[@class="keywords"]/a/text()')
            InfoContent = info.xpath('//div[@class="article"]//p//text()')


            InfoPublDate.extend(InfoSource)
            InfoPublDate.extend(InfoTitle)
            Join_Key = ' '.join(InfoKeyword) # 我按照空格的格式保存了连续三个的关键词
            InfoPublDate.append(Join_Key)
            Join_Con = ''.join(InfoContent) # 文本会出现换行的情况，抓取的时候则直接将文本全部连接起来。
            InfoPublDate.append(Join_Con)
            InfoPublDate.extend([gdnz[i]])
#             for j in range(3):
#                 InfoPublDate[j] = InfoPublDate[j].replace(u'\u3000',u'')

            ### 文本合并StockID
            ### 全部一并打包InfoPublDate
            InfoPublDate.append(pid[i])
            
            with open('news_sina_2.csv', 'a+', encoding='utf-8-sig', newline='') as csvfile: #这里的encoding=‘utf-8-sig’非常重要
                w = csv.writer(csvfile, dialect='excel')
                w.writerow(InfoPublDate)        
    return InfoPublDate
```


```python
gdnc = get_data_news_content()
```

## 保存csv至Mysql
> 这里的工作我只做到了python将csv导入到Mysql，一边抓取一边保存到MySQL有些问题还没有解决，以后有机会还会补上（逃

```python
import MySQLdb
import pymysql
import codecs
```


```python
csv = pd.read_csv('news_sina_2.csv', header=None)
```


```python
print(csv[0][1]) #代表着第一个属性，时间，的第二个值
```

    2018年09月11日 21:17
    


```python
conn = MySQLdb.connect(host='localhost', user='root', passwd='231111', db='mysql', charset='utf8')
cur = conn.cursor()
for i in range(0, len(gdnz)):
    try:
        cur.execute("insert into sinainfo(InfoPublDate,InfoSource, InfoTitle, InfoKeyword, InfoContent, Url, StockID) values ('%s', '%s', '%s', '%s', '%s', '%s', '%s')" % (csv[0][i], csv[1][i], csv[2][i], csv[3][i], csv[4][i], csv[5][i], csv[6][i]))
        #stockid = cur.execute("insert into sinainfo() values ()" % )
        #print('date:', date)
        conn.commit()
    except Exception as e:
        #print(e)
        conn.rollback()

cur.close()

```



    

## 去重（未完待续...继续跑...一边跑一边喊的那种


```python
def distinct():
    conn = MySQLdb.connect(host='localhost', user='root', passwd='231111', db='mysql', charset='utf8')
    cur = conn.cursor()
    sql_groupby = "DELETE from sinainfo where InfoPublDate NOT IN                                      \
                    (                                                                                  \
                        SELECT s.InfoPublDate from                                                     \
                            (                                                                          \
                            SELECT InfoPublDate from sinainfo where InfoPublDate GROUP BY InfoPublDate \
                        )AS s                                                                          \
                    )"
    try:
        cur.execute(sql_groupby)
        conn.commit()
        date = cur.fetchall()
        print('date:',date[4])
        
    except Exception as e:
        print('e:',e)
        conn.rollback()
cur.close()
```


```python
distinct()
```

    e: (1292, "Truncated incorrect INTEGER value: '2018年09月14日 07:21'")
    
