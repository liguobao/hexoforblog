---
layout: post
title: .NET_58同城品牌公寓爬虫
category: .net
date: 2016-08-08 00:00:00
tags:
- 58City
- Crawler
---

源码地址：[https://github.com/liguobao/58HouseSearch](https://github.com/liguobao/58HouseSearch)

在线地址：[58公寓高德搜房(全国版)：http://codelover.link:8080/](http://codelover.link:8080/)

周末闲着无事刷知乎发现一个爬虫教程（[高德API+Python解决租房问题](https://zhuanlan.zhihu.com/p/21883516)
），正中最近想要换地方住的痛点。然后大早上懒觉都没睡就屁颠屁颠开始研究这个教程了。这样教程在实验楼网站里面有手把手步骤，有兴趣自取（[实验楼：高德API+Python解决租房问题](https://www.shiyanlou.com/courses/599)）。

整体项目主要分成两步：

第一步:python爬取数据，生成数据文件;

第二步：导入数据文件，在地图上显示房源，设定上班地点后自动计算出行路线和路程时间。

研究了一下这个教程之后发现这货做得实在有点粗糙，只能当教程用，完全没有通用实际价值。

而且这里面还有个更大的问题：教程是基于北京的数据来做的，而我在上海...

虽然说改改python数据源，改改导航页面JS完事。不过是在难用...

于是，开始自己动手了。先看原有的python代码。


```python

#-*- coding:utf-8 -*-
from bs4 import BeautifulSoup
from urlparse import urljoin
import requests
import csv

url = "http://bj.58.com/pinpaigongyu/pn/{page}/?minprice=2000_4000"

#已完成的页数序号，初时为0
page = 0

csv_file = open("rent.csv","wb") 
csv_writer = csv.writer(csv_file, delimiter=',')

while True:
    page += 1
    print "fetch: ", url.format(page=page)
    response = requests.get(url.format(page=page))
    html = BeautifulSoup(response.text)
    house_list = html.select(".list > li")

    # 循环在读不到新的房源时结束
    if not house_list:
        break

    for house in house_list:
        house_title = house.select("h2")[0].string.encode("utf8")
        house_url = urljoin(url, house.select("a")[0]["href"])
        house_info_list = house_title.split()

        # 如果第二列是公寓名则取第一列作为地址
        if "公寓" in house_info_list[1] or "青年社区" in house_info_list[1]:
            house_location = house_info_list[0]
        else:
            house_location = house_info_list[1]

        house_money = house.select(".money")[0].select("b")[0].string.encode("utf8")
        csv_writer.writerow([house_title, house_location, house_money, house_url])

csv_file.close()

```
整个代码基本思路就是，爬取http://bj.58.com/pinpaigongyu/pn/{page}/?minprice=2000_4000页面数据，然后扔到创建的csv文件里面作为下一步的数据源。
通过研究http://bj.58.com/pinpaigongyu/pn/{page}/?minprice=2000_4000这个页面的数据，我们可以很容易发现，在页面中，每条数据都是一个li标签。


如下图：

![](http://7xread.com1.z0.glb.clouddn.com/685849af-2ccc-4454-a26e-e2a1c3001378)

![](http://7xread.com1.z0.glb.clouddn.com/909df442-b1d9-4073-84d0-99a5b3cc9022)

#### li结构如下：

```html
<li logr="" class="">
  <a href="/pinpaigongyu/26851774057013x.shtml"
  target="_blank" onclick="clickLog('from=fcpc_list_gy_sh_tupian')" 
  tongji_label="listclick">
    <div class="img">
     <img lazy_src="" alt="" src="">
    </div>
        <div class="des">
            <h2>【合租】菊园新区 柳湖景庭 3室次卧</h2>
            <p class="room">
            3室1厅1卫&nbsp; &nbsp; 13m²&nbsp;&nbsp; 3/6层&nbsp; </p>
            <p class="dist"></p>
            <p class="spec">
            <span class="spec1">公共阳台</span>
            <span class="spec2">公共卫生间</span>
            <span class="spec3">离地铁近</span>
            <span class="spec4">厨房</span>
            </p>
        </div>
        <div class="money">
            <span><b>1100</b>元/月 </span>
         <p>租房月付</p>
        </div>
  </a>
</li>
```
照着python的思路，是把所有的li标签的数据提取出来的。

我自己研究的时候又看了下，其实数据都在一个属性为tongji_label="listclick"的a标签里面。

一般来说，字符匹配用正则表达式完事，奈何正则水平实在不佳。我还是选择直接上HtmlAgilityPack算了。
关于HtmlAgilityPack的介绍还是看官网算了。[HtmlAgilityPack](http://htmlagilitypack.codeplex.com/)

HtmlAgilityPack是.NET一个比较强大的HTML处理类库了，基本可以让你像JS来操作HTML标签。
安装这货很简单，直接在Nuget PM包管理工具里面输入下面命令就完事了。

```csharp
Install-Package HtmlAgilityPack

```
有需要使用教程可以看这个：[Html Agility Pack基础类介绍及运用](http://www.cnblogs.com/ITmuse/archive/2010/05/29/1747199.html)


下面直接贴control源码算了。

```csharp
 /// </summary>
 /// <param name="costFrom">价格区间起始值</param>
 /// <param name="costTo">价格区间终止值</param>
 /// <param name="cnName">城市拼音首字母</param>
 /// <returns></returns>
 public ActionResult Get58CityRoomData(int costFrom, int costTo, string cnName)
        {
            if (costTo<=0 || costTo < costFrom)
            {
                return Json(new { IsSuccess = false, Error = "输入数据有误，请重新输入。" });
            }

            if (string.IsNullOrEmpty(cnName))
            {
                return Json(new { IsSuccess = false, 
                Error = "城市定位失败，建议清除浏览器缓存后重新进入。" });
            }

            try
            {
                var lstHouse = new List<HouseInfo>();

                string tempURL = "http://" + 
                cnName + ".58.com/pinpaigongyu//pn/{0}/?minprice="
                + costFrom + "_" + costTo;

                Uri uri = new Uri(tempURL);

                var htmlResult = HTTPHelper.GetHTMLByURL(string.Format(tempURL, 1));

                HtmlDocument htmlDoc = new HtmlDocument();
                htmlDoc.LoadHtml(htmlResult);

                var countNodes = htmlDoc.DocumentNode.
                SelectSingleNode(".//span[contains(@class,'list')]");
                int pageCount = 10;

                if (countNodes != null && countNodes.HasChildNodes)
                {
                    pageCount = Convert.ToInt32(countNodes.ChildNodes[0].InnerText) / 20;

                    if(pageCount==0)
                    {
                        return Json(new { IsSuccess = false, 
                        Error =string.Format("没有找到价格区间为{0} - {1}的房子。",
                        costFrom,costTo)});
                    }
                    
                }
                for (int pageIndex = 1; pageIndex <= pageCount; pageIndex++)
                {
                    htmlResult = HTTPHelper.GetHTMLByURL(string.Format(tempURL, pageIndex));
                    htmlDoc.LoadHtml(htmlResult);
                    var roomList = htmlDoc.DocumentNode
                    .SelectNodes(".//a[contains(@tongji_label,'listclick')]");
                    foreach (var room in roomList)
                    {
                        var houseTitle = room.SelectSingleNode(".//h2").InnerHtml;
                        var houseURL = uri.Host + room.Attributes["href"].Value;
                        var house_info_list = houseTitle.Split(' ');
                        var house_location = string.Empty;
                        if (house_info_list[1].Contains("公寓") 
                        || house_info_list[1].Contains("青年社区"))
                        {
                            house_location = house_info_list[0];
                        }
                        else
                        {
                            house_location = house_info_list[1];
                        }
                        var momey = room.SelectSingleNode(".//b").InnerHtml;

                        lstHouse.Add(new HouseInfo()
                        {
                            HouseTitle = houseTitle,
                            HouseLocation = house_location,
                            HouseURL = houseURL,
                            Money = momey,
                        });
                    }
                }

                return Json(new { IsSuccess = true, HouseInfos = lstHouse });
            }
            catch (Exception ex)
            {
                return Json(new { IsSuccess = false,
                Error = "获取数据异常。" + ex.ToString() });
            }
        }
```

下面解释一下核心代码。

片段一：获取总数。

在观察58同城页面的时候，无意发现其实第一个加载的页面中有一个数据总条数，隐藏在页面里面的。
```html
 <span class="listsum"><em>1813</em>条结果</span>
```
这样一来，总页面就很清晰了。页面=总数/每页20条。然后我们根据已知的数据规则去循环请求页面，也就能拿到所有的搜索数据了。

核心代码，获取总条数。
```
var countNodes = htmlDoc.DocumentNode.
SelectSingleNode(".//span[contains(@class,'list')]");
int pageCount = 10;

if (countNodes != null && countNodes.HasChildNodes)
{
    pageCount = Convert.ToInt32(countNodes.ChildNodes[0].InnerText) / 20;

    if(pageCount==0)
    {
        return Json(new { IsSuccess = false, 
        Error =string.Format("没有找到价格区间为{0} - {1}的房子。",
        costFrom,costTo)});
    }
                    
}

```

在HTMLDoc里面找到一个span的class包含list的节点，获取它子节点（即em）的内容，强制转换成数字，也就是我们要找的总条数了。总条数除以20就得到了页数，下面就是开始循环请求页面了。

在最上面我们分析过公寓数据分布，数据是li里面套a标签，我们需要的地理位置、房间名称、价格都在a标签里面。

这样一来，我们这要获得到页面所有带有属性为tongji_label="listclick"的a标签数据，也就得到了我们所有需要的数据。

看一下a标签的数据组成：
```html

 <a href="/pinpaigongyu/26851774057013x.shtml"
  target="_blank" onclick="clickLog('from=fcpc_list_gy_sh_tupian')" 
  tongji_label="listclick">
    <div class="img">
     <img lazy_src="" alt="" src="">
    </div>
        <div class="des">
            <h2>【合租】菊园新区 柳湖景庭 3室次卧</h2>
            <p class="room">
            3室1厅1卫&nbsp; &nbsp; 13m²&nbsp;&nbsp; 3/6层&nbsp; </p>
            <p class="dist"></p>
            <p class="spec">
            <span class="spec1">公共阳台</span>
            <span class="spec2">公共卫生间</span>
            <span class="spec3">离地铁近</span>
            <span class="spec4">厨房</span>
            </p>
        </div>
        <div class="money">
            <span><b>1100</b>元/月 </span>
         <p>租房月付</p>
        </div>
  </a>
```

我们要的房间信息在一个h2的标签里面，公寓租金价钱在class="money"的div标签里面。

于是有了一下代码：

```csharp

 for (int pageIndex = 1; pageIndex <= pageCount; pageIndex++)
{
    htmlResult = HTTPHelper.GetHTMLByURL(string.Format(tempURL, pageIndex));
    htmlDoc.LoadHtml(htmlResult);
    //找到所有的带有属性为tongji_label="listclick"的a标签数据
    var roomList = htmlDoc.DocumentNode.SelectNodes(".//a[contains(@tongji_label,'listclick')]");
    foreach (var room in roomList)
    {
        //获取其中为h2的房间数据，然后用空格分割成数组
        var houseTitle = room.SelectSingleNode(".//h2").InnerHtml;
        var houseURL = uri.Host + room.Attributes["href"].Value;
        var house_info_list = houseTitle.Split(' ');
        var house_location = string.Empty;
        //分割出来的数组，第二个包含公寓或青年社区，则取第一个数据为所在地区，否则取第二个数据
        //【合租】菊园新区 柳湖景庭 3室次卧 
        // 所在地区为：菊园新区
        if (house_info_list[1].Contains("公寓") || house_info_list[1].Contains("青年社区"))
        {
            house_location = house_info_list[0];
        }
        else
        {
            house_location = house_info_list[1];
        }
        //获取标签为b的数据，价格就在里面了
        var momey = room.SelectSingleNode(".//b").InnerHtml;

        lstHouse.Add(new HouseInfo()
        {
            HouseTitle = houseTitle,
            HouseLocation = house_location,
            HouseURL = houseURL,
            Money = momey,
        });
    }
}

```

后端来说，基本就这些内容了。

还有一些前端高德地图接口调用下次再讲吧，要陪女票玩游戏去了...

^-^









