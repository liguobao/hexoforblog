---
layout: post
title: 58同城高德搜房项目JS相关知识
category: JavaScript
---

在线地址：[58同城品牌公寓高德搜房](http://codelover.link:8080/)

Github地址：[https://github.com/liguobao/58HouseSearch](https://github.com/liguobao/58HouseSearch)

知乎专栏(点赞用的)：[高德API+Python解决租房问题(.NET版)](https://zhuanlan.zhihu.com/p/21960329)

经过了一个星期的修补补，以及小伙伴奉献的代码，整个项目基本处于基本稳定运行的状态。

同时加入了一下新功能：

1. IP定位：调用高德地图IP定位功能实现
2. 移动地图中心定位：调用高德地图移动地图定位实现
3. 定位城市名转58同城城市名以获得准确58同城城市域名：抓取58同城城市分类信息
4. 优化数据源、去除广告数据：小伙伴奉献代码

今天主要简单讲解一下其中使用的一些高德地图API接口。

#####高德地图JavaScript API 主体为map对象，基本所有的操作都是通过map对象来实现的。
map对象实例化是通过 Amap类来做的。如以下代码：
```javascript
map = new AMap.Map("container", {
        resizeEnable: true,
        zoomEnable: true,
        center: [121.297428, 31.1345],//经纬度，此处为上海
        zoom: 11
    });

```





#### IP定位
调用Map.CitySearch()获得当前IP所在城市，直接将地图显示成当前城市。代码如下：
```JavaScript
function showCityInfo(map) {
    //实例化城市查询类
    var citysearch = new AMap.CitySearch();
    //自动获取用户IP，返回当前城市
    citysearch.getLocalCity(function (status, result) {
        if (status === 'complete' && result.info === 'OK') {
            if (result && result.city && result.bounds) {
                var cityinfo = result.city;//获得XX市
                var citybounds = result.bounds;//用于设置地图显示位置的实例
                cityName = cityinfo.substring(0, cityinfo.length - 1);//去掉市这个字
                ConvertCityCNNameToShortCut();//城市名转换成58同城城市域名字母，如上海->sh,苏州->su,
                                              //下面会有实现代码

                document.getElementById('IPLocation').innerHTML = '您当前所在城市：' + cityName;
                //地图显示当前城市
                map.setBounds(citybounds);
            }
        } else {
            document.getElementById('IPLocation').innerHTML = result.info;
        }
    });
}

```


#### 移动地图自动中心定位

之前有一版是让用户输入城市名，然后直接定位到输入的城市的。
这个功能卡在了设置地图显示位置上，如果是使用高德地图提供的搜索控件的话，又存在输入结果之后搜索结果可能是多个的问题。而且这点我只是要取到用户想要定位的城市而已，感觉没必要做得太复杂。
昨晚在看高德地图API的时候发现，有一个移动地图获得地图中心所在位置的样例，马上眼前一亮了。这个功能比我想要的还要好...果断上。

```JavaScript
function MapMoveToLocationCity()
{
    map.on('moveend', getCity);
    function getCity() {
        map.getCity(function (data) {
            if (data['province'] && typeof data['province'] === 'string') {

                var cityinfo = (data['city'] || data['province']);
                cityName = cityinfo.substring(0, cityinfo.length - 1);
                ConvertCityCNNameToShortCut();//城市名转58同城地区域名

                document.getElementById('IPLocation').innerHTML = '地图中心所在城市：' + cityName;

            }
        });
    }
}
```

整个代码的意思是，给map绑定一下移动时间，移动完了之后，调用getCity的方法获取当前地图中心所在城市信息。

这个时候要注意，城市名可能在city对象里面，也可能在province里面。

原因很简单：普通城市等级就是城市，我国还存在一个和省份一个等级的城市：直辖市。因此直辖市的城市名是在province里面的。


#### 城市名匹配58同城地区域名

这个是上个版本(两三天前)的一个bug引出来的新功能。

上个版本是让用户输入城市名，然后提取城市名的中文拼音首字母作为58同城地区域名。如上海 =sh，广州=gz，北京=bj，成都=cd。

这个功能使用的是网上别人写的一个JS库，通过汉字匹配实现的。转换出来的数据没什么问题，不过我国汉字实在奥妙。

广州=gz，赣州=gz；
遂宁=sn；绥宁=sn；
惠州=hz，杭州=hz。

这样一来，上面这个做法就没法玩了。

想了下怎么解决这个问题，灵机一动。反正是在爬58的数据，这个城市名和城市域名数据58同城肯定有啊，然后找到了这个。
[58同城城市分类导航](http://www.58.com/changecity.aspx?PGTID=0d100000-0007-a77b-4c4b-a28f725b8f5a&ClickID=1)

![](http://7xread.com1.z0.glb.clouddn.com/c01c293a-d5cc-4f58-80dc-13aa05d47b01)

很明显，我要的所有城市名和城市域名都是里面了。

晚上和衣衣说了下，衣衣一大早就把处理好的json数据给我了。

于是来了下面一段代码：
```JavaScript

//加载json文件
$.getJSON("DomainJS/city.json", function (data)
{
      allCityInfo = data;
});


function ConvertCityCNNameToShortCut()
{
    var filterarray = $.grep(allCityInfo, function (obj) {
        return obj.cityName == cityName;
    });//找到当前城市名对应的json对象
    //获取json对象的地区域名
    cityNameCNPY = filterarray instanceof Array ? 
    filterarray[0].shortCut : filterarray != null ? filterarray.shortCut : "";
}

```


#### 高德地图自动补全功能

```html
  <div class="control-input">
       <input id="work-location" type="text" style="width:60%">
  </div>
```

```JavaScript
 var auto = new AMap.Autocomplete({
        input: "work-location"
    });

    AMap.event.addListener(auto, "select", workLocationSelected);
    
```
看方法前面也知道，其实这就是把ID为work-location的input初始化为地图插件，然后给Amap增加了一个监听事件。
当其中选中某一个数据的时候，触发workLocationSelected函数。效果如下：

![](http://7xread.com1.z0.glb.clouddn.com/fe425992-e7c4-4cae-800e-319eff3b17e8)


在这里locationSelected是定位到所选位置，代码如下：
```
function workLocationSelected(e) {
    workAddress = e.poi.name;
    loadWorkLocation();
}

function loadWorkLocation() {
    delWorkLocation();
    var geocoder = new AMap.Geocoder({
        city: cityName,
        radius: 1000
    });

    geocoder.getLocation(workAddress, function (status, result) {
        if (status === "complete" && result.info === 'OK') {
            var geocode = result.geocodes[0];
            x = geocode.location.getLng();
            y = geocode.location.getLat();
            loadWorkMarker(x, y);
            loadWorkRange(x, y, 60, "#3f67a5", vehicle);
            map.setZoomAndCenter(12, [x, y]);
        }
    })
}

```

至于导航功能代码我没怎么动，没去研究就不献丑了...


最后来个效果图。

####北京
![](http://7xread.com1.z0.glb.clouddn.com/e7900aba-5a56-417c-9dd9-63527583e84b)



####成都
![](http://7xread.com1.z0.glb.clouddn.com/c9947d97-1b76-42bf-82b3-aca817e84e13)



####苏州
![](http://7xread.com1.z0.glb.clouddn.com/941809ae-aa37-4a3b-89c7-10646cc7e3e7)


###深圳
![](http://7xread.com1.z0.glb.clouddn.com/86712397-27ec-4b37-bbf8-81191e530ef6)










