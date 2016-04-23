---
layout: post
title: mono webreques https exception
category: .net
---

前几天在做一个使用URL通过WebRequest请求HTML页面的功能的时候遇到了点坑，程序在开发环境没有任何的问题，部署到linux mono上之后就跪了。代码如下：

```csharp
public static string GetHTML(string url)
{
    string htmlCode;
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
                        Encoding ending = Encoding.GetEncoding(regex.Match(contentype).Groups[1].Value.Trim());
                        using (StreamReader sr = new System.IO.StreamReader(zipStream, ending))
                        {
                            htmlCode = sr.ReadToEnd();
                        }
                    }
                    else
                    {
                        using (StreamReader sr = 
                        new System.IO.StreamReader(zipStream, Encoding.UTF8))
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
                using (System.IO.StreamReader sr = new System.IO.StreamReader(streamReceive, Encoding.Default))
                {
                    htmlCode = sr.ReadToEnd();
                }
            }
        }
        return htmlCode;

    }catch(Exception ex)
    {
        LogHelper.WriteException("GetHTML", ex, new { Url = url });
        return "";
    }

}
```

开发环境在Windows10 + VS2013,整个代码跑起来没什么问题。

无论是HTTP还是HTTPS协议，网页HTML一样能获取得到。

网站部署到linux Jexus之后HTTP协议的网站同样可以获取到HTML，遇到HTTPS协议的网站的时候就跪了。

抓到的异常信息如下：

```csharp
   System.Net.WebException: Error: TrustFailure (The authentication or decryption has failed.) 
   ---> System.IO.IOException: The authentication or decryption has failed.
   ---> System.IO.IOException: The authentication or decryption has failed. 
   ---> Mono.Security.Protocol.Tls.TlsException:
   Invalid certificate received from server. Error code: 0xffffffff800b0109
   at Mono.Security.Protocol.Tls.RecordProtocol.EndReceiveRecord 
   (IAsyncResult asyncResult) <0x41b58d80 + 0x0013e> in <filename unknown>:0 
   at Mono.Security.Protocol.Tls.SslClientStream.SafeEndReceiveRecord
   (IAsyncResult ar, Boolean ignoreEmpty) <0x41b58cb0 + 0x00031> in <filename unknown>:0 
   at Mono.Security.Protocol.Tls.SslClientStream.NegotiateAsyncWorker
   (IAsyncResult result) <0x41b72a40 + 0x0023f> in <filename unknown>:0 
   --- End of inner exception stack trace ---
   at Mono.Security.Protocol.Tls.SslClientStream.EndNegotiateHandshake 
   (IAsyncResult result) <0x41ba07e0 + 0x000f3> in <filename unknown>:0 
   at Mono.Security.Protocol.Tls.SslStreamBase.AsyncHandshakeCallback 
   (IAsyncResult asyncResult) <0x41ba0540 + 0x00086> in <filename unknown>:0 
   --- End of inner exception stack trace ---
   at Mono.Security.Protocol.Tls.SslStreamBase.EndRead 
   (IAsyncResult asyncResult) <0x41b73fd0 + 0x00199> in <filename unknown>:0 
   at Mono.Net.Security.Private.LegacySslStream.EndAuthenticateAsClient 
   (IAsyncResult asyncResult) <0x41b73f30 + 0x00042> in <filename unknown>:0 
   at Mono.Net.Security.Private.LegacySslStream.AuthenticateAsClient (System.String targetHost, 
   System.Security.Cryptography.X509Certificates.X509CertificateCollection
   clientCertificates, SslProtocols enabledSslProtocols,
   Boolean checkCertificateRevocation) <0x41b6a660 + 0x00055> in <filename unknown>:0 
   at Mono.Net.Security.MonoTlsStream.CreateStream (System.Byte[] buffer) 
   <0x41b69c30 + 0x00159> in <filename unknown>:0 
   --- End of inner exception stack trace ---
   at System.Net.HttpWebRequest.EndGetResponse (IAsyncResult asyncResult) 
   <0x41b67660 + 0x001f9> in <filename unknown>:0 
   at System.Net.HttpWebRequest.GetResponse () <0x41b60920 + 0x0005a> in <filename unknown>:0 
   at WebBookmarkUI.Commom.HTTPHelper.GetHTML (System.String url) <0x41b59b70 + 0x00235> 
   in <filename unknown>:0 

```

有用的信息基本就是：

1. Invalid certificate received from server
2. The authentication or decryption has failed

一开始百思不得其解，为嘛好好的程序在开发环境跑得都好的，到了mono上就挂了，多疑的我还以为是mono的bug。
今天静下心来去找了一下资料，发现Mono的文档有这个问题的描述，认真读了一遍，又去请教了一番宇内流云大大，终于弄懂了是什么回事。

先贴一下相关资料：

1. [stackoverflow mono-webrequest-fails-with-https](http://stackoverflow.com/questions/4926676/mono-webrequest-fails-with-https)

2. [mono doc security](http://www.mono-project.com/docs/faq/security/)


这个问题是出现的原因是Windows自带了HTPPS的根证书，linux默认则是没有带有根证书的。我们的mono在调用WebRequest去请求HTTPS协议的网站的时候，找不到任何有效的根证书，所以抛出上面的异常了。

解决方案也很简单，为linux导入一下HTTPS根证书就好。

在linux服务器上面执行下面这条命令。
```
mozroots --import --ask-remove --machine

```


然后在网站的Application_Start()里面添加下面代码：

```csharp
 System.Net.ServicePointManager.ServerCertificateValidationCallback +=
 delegate(object sender, System.Security.Cryptography.X509Certificates.X509Certificate certificate,
 System.Security.Cryptography.X509Certificates.X509Chain chain,
 System.Net.Security.SslPolicyErrors sslPolicyErrors)
{
    return true; // **** Always accept
};

```

完事。

这个故事告诉我们，linux干活都是要亲力亲为呀。



