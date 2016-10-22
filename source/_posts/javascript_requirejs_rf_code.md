---
layout: post
title: 使用requirejs编写模块化代码
category: javascript
date: 2016-10-22 00:00:00
tags:
- javascript
- requirejs
---

### 写在前面
最早接触javascript的时候，javascript代码直接扔在script标签里面就完事了。

反正代码不多，交互简单，逻辑不难，和HTML混在一起也未尝不可。

后来交互越来越复杂，代码越多越多了，我们就开始把JS代码独立到了单独的JS文件中。

公共的库引用在前，自己的逻辑代码引用在后，全局变量定义在HTML内部，在独立JS文件中直接使用变量就好。

我们会经常看到下面这种代码：

```
　　<script src="1.js"></script>
　　<script src="2.js"></script>
　　<script src="3.js"></script>
　　<script src="4.js"></script>
　　<script src="5.js"></script>
　　<script src="6.js"></script>
```
通过script标签顺序去js管理依赖关系。

阮一峰老师在[Javascript模块化编程（三）：require.js的用法](http://www.ruanyifeng.com/blog/2012/11/require_js.html)
一文中总结了这样写法的缺点：

首先，加载的时候，浏览器会停止网页渲染，加载文件越多，网页失去响应的时间就会越长；

其次，由于js文件之间存在依赖关系，因此必须严格保证加载顺序（比如上例的1.js要在2.js的前面），依赖性最大的模块一定要放到最后加载.

当依赖关系很复杂的时候，代码的编写和维护都会变得困难。

而requirejs的诞生便是为了解决这个问题。

### [requirejs](http://requirejs.org/docs/download.html)

在[官网](http://requirejs.org/docs/download.html)把requirejs 下载回来之后。使用一般的方法引入：
```
<script src="js/require.js"></script>
```
但是这样的方法，还是可能在加载require.js的时候导致网页失去响应。解决方案一般有两种：

1. 把上面的代码放到网页底部

2. 使用异步的方法加载，如下：

```
<script src="js/require.js" defer async="true" ></script>
```
[async属性](http://www.w3school.com.cn/html5/att_script_async.asp) 表明这个文件需要异步加载，避免网页失去响应。

不过IE下不支持这个属性，只支持defer，所以可以把defer也写上。



### 加载主模块
在上一步，我们已经引入了require了，那么require怎么知道我们究竟要加载什么东西呢？答案是使用data-main属性。
假设我们的主模块为js/home.js,引入代码应该如下：
```
　<script src="js/require.js" data-main="js/home"></script>
  //require.js默认文件后缀为js，所以home.js可以写成home。
```
接下来我使用[58HouseSearch](https://github.com/liguobao/58HouseSearch) 的代码来讲解一下require的使用。

在此项目里面，重构前大概就是JS变量漫天飞，js文件里面各种函数到处乱放。一开始用起来还没什么，后来加入了更多功能的时候，JS代码维护起来就疼不欲生了。因此托了个小伙伴帮忙使用模块化思想重构了一下JS代码。

上面说了，我们首先需要创建我们的模块，在这个项目里面，主模块叫home.js。

home.js中我们需要配置一下require.config.
```
require.config({
    baseUrl: '/DomainJS/',
    paths: {
        jquery: "lib/jquery-1.11.3.min",
        "AMUI": "lib/amazeui.2.7.1.min",
        "jquery.range": "lib/jquery.range",
        "es5": "lib/es5",
        "mapController": "mapController",
        "addToolbar": "addToolbar",
    },
    shim: {
        "addToolbar": {
            deps: ["jquery"]
        },
        "jquery.range": {
            deps: ["jquery"]
        }
    }
});

```
在这里我主要配置了一下baseURL(所有模块的查找根路径)，paths(名称映射)，shim(
为那些没有使用define()来声明依赖关系、设置模块的"浏览器全局变量注入"型脚本做依赖和导出配置。)

关于require.config的详细内容可以看下下面这些文章：

1. [RequireJS进阶:配置文件的学习](https://segmentfault.com/a/1190000002401665) 
2. [RequireJS进阶:模块的优化及配置的详解](https://segmentfault.com/a/1190000002403806)

配置做完了，我们也可以开始真正写我们的逻辑代码了,我们使用require来加载我们需要的库。
代码如下：

```
require(['domready!', 'jquery', 'AMUI', 'mapController', 'city', 'commuteGo'], 

function (doc, $, AMUI, mapController, city, commuteGo) {
    city.initAllCityInfo();
    mapController.init();

    $("input[name='locationType']").bind('click', 
    mapController.locationMethodOnChange)

    $("input[name='vehicle']").bind('click', commuteGo.go)

    $('#Get58Data').bind('click', function(e) {
        e.preventDefault();
     
        mapController.Get58DataClick();
        e.stopPropagation();
    });

 
    $.ajax({
        type: "post",
        url: "../Commom/GetPVCount",
        data: { },
        success: function (result)
        {
            if (result.IsSuccess){
                $("#lblPVCount").text(result.PVCount);
            }else {
                $("#lblPVCount").text(0);
                console.log(result.Error);
            }
        }
    });

    $('#search-offcanvas').offCanvas({ effect: 'overlay' });

    $(".amap-sug-result").css("z-index", 9999);
})

```

忽略function里面的具体逻辑，加载如下：
```
require(['domready!', 'jquery', 'AMUI', 'mapController', 'city', 'commuteGo'], 
function (doc, $, AMUI, mapController, city, commuteGo){

//todo

});

```

第一个参数为一个数组，表示所依赖的模块，此处为['domready!', 'jquery', 'AMUI', 'mapController', 'city', 'commuteGo']；

第二个参数为回调函数，当前面指定的模块都全部加载成功之后，便调用此函数。加载的模块会以参数形式传入此函数，从而在回调函数内部就可以使用这些模块啦。

require()异步加载所需模块的时候，此时浏览器并不会失去响应；当前面的模块加载成功之后，执行回调函数才会运行我们的逻辑代码，因此解决了依赖性问题。

讲完了模块加载，我们下面讲一下模块编写。

### AMD模块编写

require.js加载的模块的采用的AMD规范。所以我们的模块必须按照AMD的规定来写。

关于AMD规范详情可以看这个文章：[Javascript模块化编程（二）：AMD规范](http://www.ruanyifeng.com/blog/2012/10/asynchronous_module_definition.html)

模块有两个情况，不依赖其他模块和依赖其他模块。

#### 不依赖其他模块
直接define定义，使用function回调。

[58HouseSearch/DomainJS/helper.js](https://github.com/liguobao/58HouseSearch/blob/master/58HouseSearch/DomainJS/helper.js)
```
define(function () {

    //获取URL中的参数
    var getQueryString=  function (name) {
        var reg = new RegExp("(^|&)" + name + "=([^&]*)(&|$)");
        var r = window.location.search.substr(1).match(reg);
        if (r != null) return unescape(r[2]); return null;
    }
    return {
        getQueryString: getQueryString,
    };
})
```

#### 依赖其他模块
define中如同require一样，用数组表明需要加载的模块，function回调。

[58HouseSearch/DomainJS/marker.js](https://github.com/liguobao/58HouseSearch/blob/master/58HouseSearch/DomainJS/marker.js)
```
define(['mapSignleton', 'city', 'transfer'], 
function(mapSignleton, city, transfer) {
    var _map = mapSignleton.map;
    var _workMarker = null;
    var _markerArray = [];
    var load = function(x, y, locationName) {
        _workMarker = new AMap.Marker({
            map: _map,
            title: locationName,
            icon: 'http://webapi.amap.com/theme/v1.3/markers/n/mark_r.png',
            position: [x, y]
        });
    }

    var add = function(address, rent, href, markBG) {
        new AMap.Geocoder({
            city: city.name,
            radius: 1000
        }).getLocation(address, function(status, result) {

            if (status === "complete" && result.info === 'OK') {
                var geocode = result.geocodes[0];
                var rentMarker = new AMap.Marker({
                    map: _map,
                    title: address,
                    icon: markBG ? 'IMG/Little/' +
                    markBG : 'http://webapi.amap.com/theme/v1.3/markers/n/mark_b.png',
                    position: [geocode.location.getLng(), geocode.location.getLat()]
                });
                _markerArray.push(rentMarker);

                rentMarker.content = "<div><a target = '_blank' href='" 
                + href + "'>房源：" + address + "  租金：" + rent + "</a><div>"
                rentMarker.on('click', function(e) {
                    transfer.add(e, address);
                });
            }
        })
    };

    var clearArray = function() {
        if (_markerArray && _markerArray.length > 0) 
        _map.remove(_markerArray);
        _markerArray = [];
    }

    var clear = function() {
        if (_workMarker) {
            _map.remove(_workMarker);
        }
    }

    return {
        load: load,
        add: add,
        clearArray: clearArray,
        clear: clear
    };
});

```
这样的话，一个供require调用的模块也就写好了。


最后感谢小伙伴[Larry Sean](https://www.zhihu.com/people/piratf) 帮忙重构代码。

全文完。



















