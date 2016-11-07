---
layout: post
title: ASP.NET MVC 微信JS-SDK认证
category: .net
date: 2016-11-01 00:00:00
tags:
- .net
- javascript
- 
---

### 写在前面
前阵子因为有个项目需要做微信自定义分享功能，因而去研究了下微信JS-SDK相关知识。

此文做个简单的记(tu)录(cao)...

### 开始
所有的东西都从文档开始:[微信JSSDK说明文档](http://mp.weixin.qq.com/wiki/11/74ad127cc054f6b80759c40f77ec03db.html)

![分享接口](http://7xread.com1.z0.glb.clouddn.com/44b1232e-9311-4abb-9200-6dd4936d7c47)


项目需要用到的是[分享接口](http://mp.weixin.qq.com/wiki/11/74ad127cc054f6b80759c40f77ec03db.html#.E5.88.86.E4.BA.AB.E6.8E.A5.E5.8F.A3) 不过使用微信JS-SDK之前，需要做JS接口认证。

认证如下：

步骤一：绑定域名

步骤二：引入JS文件

步骤三：通过config接口注入权限验证配置

步骤四：通过ready接口处理成功验证

步骤五：通过error接口处理失败验证

步骤一中允许使用域名/子域名，只要xx.com/xxx.txt或者xx.com/mp/xxx.txt能访问就好。域名认证通过之后，此域名下的所有端口的网站都可以使用JS-SDK。

步骤二没什么问题，略过。

步骤三最磨人，下面单独讲解。


### config接口注入权限验证配置

先来一段说明：
```
所有需要使用JS-SDK的页面必须先注入配置信息，否则将无法调用
（同一个url仅需调用一次，对于变化url的SPA的web app可在每次url变化时进行调用,
目前Android微信客户端不支持pushState的H5新特性，
所以使用pushState来实现web app的页面会导致签名失败，此问题会在Android6.2中修复）。
```

```javascript
wx.config({
    debug: true, // 开启调试模式,调用的所有api的返回值会在客户端alert出来，
    //若要查看传入的参数，可以在pc端打开，参数信息会通过log打出，仅在pc端时才会打印。
    appId: '', // 必填，公众号的唯一标识
    timestamp: , // 必填，生成签名的时间戳
    nonceStr: '', // 必填，生成签名的随机串
    signature: '',// 必填，签名，见附录1
    jsApiList: [] // 必填，需要使用的JS接口列表，所有JS接口列表见附录2
});
```

看到这里肯定懵逼了，这是都什么鬼...怎么玩啊。

提示我们去看附录1...看完之后总结如下：

1. 使用config接口注入权限验证配置，重点是生成合法的signatrue
2. 生成signature需要通过appid和secret获取token
3. 时间戳和调用接口URL必不可少
4. 此操作需要服务端完成，不能使用客户端实现

整个过程变成：

1. 通过appid和secret获取access_token，接着使用token获取jsapi_ticket；

2. 拿到jsapi_ticket之后，把jsapi_ticket、时间戳、随机字符串、接口调用页面URL 拼接成完整字符串，使用sha1算法加密得到signature。

3. 最后返回至页面，在wx.config里面填入appid，上一步的时间戳timestamp，上一部的随机字符串、sha1拿到的signature，想要使用的JS接口。


废话少说，直接上代码吧。

### 代码时间

```csharp
    public class WeiXinController : Controller
    {
        public static readonly string appid =
        System.Web.Configuration.WebConfigurationManager.AppSettings["wxappid"];

        public static readonly string secret =
        System.Web.Configuration.WebConfigurationManager.AppSettings["wxsecret"];

        public static readonly bool isDedug =
        System.Web.Configuration.WebConfigurationManager.AppSettings["IsDebug"] =="true";


        public static string _ticket = "";

        public static DateTime _lastTimestamp;


        public ActionResult Info(string url,string noncestr)
        {
            if (string.IsNullOrEmpty(_ticket) || 
            _lastTimestamp == null || (_lastTimestamp - DateTime.Now).Milliseconds > 7200)
            {
                var resultString = HTTPHelper.GetHTMLByURL
                ("https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid="
                    + appid + "&secret=" + secret);
                dynamic resultValue = JsonConvert.DeserializeObject<dynamic>(resultString);
                if (resultValue == null || resultValue.access_token == null 
                || resultValue.access_token.Value == null)
                {
                    return Json(new { issuccess = false, 
                    error = "获取token失败" });
                }
                var token = resultValue.access_token.Value;

                resultString = HTTPHelper.GetHTMLByURL
                ("https://api.weixin.qq.com/cgi-bin/ticket/getticket?access_token=" + 
                token + "&type=jsapi");
                dynamic ticketValue = JsonConvert.DeserializeObject<dynamic>(resultString);
                if (ticketValue == null || ticketValue.errcode == null
                || ticketValue.errcode.Value != 0 || ticketValue.ticket == null)
                    return Json(new { issuccess = false,
                    error = "获取ticketValue失败" });
                _ticket = ticketValue.ticket.Value;
                _lastTimestamp = DateTime.Now;
                var timestamp = GetTimeStamp();
                var hexString = string.Format("jsapi_ticket={0}&noncestr={3}&timestamp={1}&url={2}",
                _ticket, timestamp, url,noncestr);

                return Json(new {
                    issuccess = true, 
                    sha1value = GetSHA1Value(hexString), 
                    timestamp = timestamp, 
                    url = url, 
                    appid = appid, 
                    debug=isDedug,
                    tiket=_ticket
                });
                
            }
            else
            {
                var timestamp = GetTimeStamp();
                var hexString = string.Format("jsapi_ticket={0}&noncestr=1234567890123456&timestamp={1}&url={2}",
                   _ticket, timestamp, url);
                return Json(new { 
                    issuccess = true, sha1value = GetSHA1Value(hexString),
                    timestamp = timestamp, url = url,
                    appid = appid, debug = isDedug,tiket = _ticket
                });
            }
        }


        private string GetSHA1Value(string sourceString)
        {
            var hash = SHA1.Create().ComputeHash(Encoding.UTF8.GetBytes(sourceString));
            return string.Join("", 
            hash.Select(b => b.ToString("x2")).ToArray());
        }

        private static string GetTimeStamp()
        {

            TimeSpan ts = DateTime.Now - new DateTime(1970, 1, 1, 0, 0, 0, 0);

            return Convert.ToInt64(ts.TotalSeconds).ToString();

        }

    }
    
    
    
    
    public class HTTPHelper
    {
        public static string GetHTMLByURL(string url)
        {
            string htmlCode = string.Empty;
            try
            {
                HttpWebRequest webRequest = (System.Net.HttpWebRequest)System.Net.WebRequest.Create(url);
                webRequest.Timeout = 30000;
                webRequest.Method = "GET";
                webRequest.UserAgent = "Mozilla/4.0";
                webRequest.Headers.Add("Accept-Encoding", "gzip, deflate");
                HttpWebResponse webResponse = (System.Net.HttpWebResponse)webRequest.GetResponse();
                //获取目标网站的编码格式
                string contentype = webResponse.Headers["Content-Type"];
                Regex regex = new Regex("charset\\s*=\\s*[\\W]?\\s*([\\w-]+)", RegexOptions.IgnoreCase);
                if (webResponse.ContentEncoding.ToLower() == "gzip")//如果使用了GZip则先解压
                {
                    using (System.IO.Stream streamReceive = webResponse.GetResponseStream())
                    {
                        using (var zipStream = new System.IO.Compression.GZipStream(streamReceive, 
                        System.IO.Compression.CompressionMode.Decompress))
                        {
                            //匹配编码格式
                            if (regex.IsMatch(contentype))
                            {
                                Encoding ending = Encoding.GetEncoding
                                (regex.Match(contentype).Groups[1].Value.Trim());
                                using (StreamReader sr = new System.IO.StreamReader(zipStream, ending))
                                {
                                    htmlCode = sr.ReadToEnd();
                                }
                            }
                            else
                            {
                                using (StreamReader sr = new System.IO.StreamReader(zipStream, Encoding.UTF8))
                                {
                                    htmlCode = sr.ReadToEnd();
                                }
                            }
                        }
                    }
                }
                else
                {
                    using (System.IO.Stream streamReceive = webResponse.GetResponseStream())
                    {
                        var encoding = Encoding.Default;
                        if (contentype.Contains("utf"))
                            encoding = Encoding.UTF8;
                        using (System.IO.StreamReader sr = new System.IO.StreamReader(streamReceive, encoding))
                        {
                            htmlCode = sr.ReadToEnd();
                        }

                    }
                }
                return htmlCode;
            }
            catch (Exception ex)
            {
                return "";
            }
        }
    }

```

PS：这里要注意缓存一下_ticket（即access_token），照微信文档说的，access_token两个小时内有效，不需要频繁调用。而且获取access_token的接口有调用次数的限制，如果超过了次数，就不允许调用了。

PPS:建议noncestr和URL由前台传入比较适合，使用 var theWebUrl = window.location.href.split('#')[0] 获取URL，noncestr就随意了。

PPPS:遇到诡异的invalid signature的时候，首先检查url参数，然后检查noncestr，再不行重启一下程序获取一个新的token回来继续玩。












