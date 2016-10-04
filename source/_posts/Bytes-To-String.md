---
layout: post
title: bytes to string
category: .net
date: 2016-01-20 00:00:00
tags:
- 加密编码
---

有时候我们会遇到需要把数据加密之后再网络上传输的需求，这样的话一般使用AES256之类的算法，经过运算之后得到一个byte数组，接着转换成string，就扔出去了。对方拿到之后，用密钥解密之后便得到了对应的数据。

在C#里面，Byte数组转String字符串我们一般用Convert.ToBase64()完成。

代码如下：
```csharp
    public string BytesToString(byte[] buff)
    {
       return Convert.ToBase64String(buff);
    }

    public byte[] StringToBytes(string input)
    {
        return Encoding.UTF8.GetBytes(input);
    }
```
 
 一般来说这样也没撒问题了，不过，如果这个数据是通过URL的方式给出去的，这时候就要考虑一下特殊字符编码问题了。+、空格、%之类的特殊字符可能会导致切断URL传参的数据，导致得到的数据不一致。这样的话，解密也做不下去了。
 
 相关资料：
 <br>1、[关于URL编码](http://www.ruanyifeng.com/blog/2010/02/url_encoding.html)
 <br>2、[URL编码----url参数中有+、空格、=、%、&、#等特殊符号的问题解决](http://blog.csdn.net/luo_deng/article/details/12186535)
 
 
 不过也好在，C#提供了一个HttpUtility.UrlEncode(input)和HttpUtility.UrlEncode(input)这两个函数，让我们直接把上面的特殊字符转换成URL可识别的转义字符。
 数据出去之后先Encode一下，回来之后Decode一下，好像问题都解决了吧。
 
 
然而我们都忘了一件事情，URL到了浏览器之后，自然会对URL里面的东西Decode一次。
我实现的时候，在后台验证的时候又Decode一次,这就出问题了。

问题在哪呢？一个encode的字符被decode两次，内容已经被改掉了...
这就导致解密的时候直接挂了....



这样看来，
Convert.ToBase64String()这个不够靠谱，出来的数据可能会有特殊字符的问题。
怎么解决呢？那天晚上和老大/CTO都在看这个bug。一下子都没撒好办法....

后来CTO想了一下，说byte不就是最大不久255么？直接转16进制字符就是嘛。
于是有了下面的代码：
 
 
```csharp
	/// <summary>
	/// byte数组转string
	/// </summary>
	/// <param name="bytes"></param>
	/// <returns></returns>
	private static string BytesToString(byte[] bytes)
	{
	    if (bytes == null)
	        return string.Empty;
	   return string.Join(string.Empty, 
	   bytes.Select(b => string.Format("{0:x2}", b)).ToArray());
	}
	
	/// <summary>
	/// string转byte数组
	/// </summary>
	/// <param name="str"></param>
	/// <returns></returns>
	private static byte[] StringToBytes(string str)
	{
	    if (string.IsNullOrEmpty(str))
	        return null;
	    byte[] bytes = new byte[str.Length / 2];
	    for (int i = 0; i < str.Length; i += 2)
	    {
	        bytes[i / 2] = Convert.ToByte("0x" + str[i] + str[i + 1], 16);
	    }
	    return bytes;
	}
```


问题解决。

关于base64的实现，下面这个链接是相关资料，有兴趣自己看啦。

[C#/ASP.NET Base64编码原理及实现](http://www.hejingzong.cn/blog/ViewBlog_36.aspx)

[C＃实现Base64编码与解码](http://www.cnblogs.com/tuyile006/archive/2008/01/17/1043178.html)


