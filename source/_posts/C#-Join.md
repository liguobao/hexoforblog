---
layout: post
title: .NET join用法
category: .net
---


#.NET   Join
 连接：内连接、外连接、左连接、右连接。
SQL的Join这里就不多说了，
今天主要是看一下LINQ的Join用法，以及Enumerable.Join()的用法。

Join用于连接数据，首先就是数据之间有联系咯。
 
先说Enumerable.Join()。
参数类型如下：
			public static IEnumerable<TResult> Join<TOuter, TInner, TKey, TResult>
			(      
			         this IEnumerable<TOuter> outer,
			        IEnumerable<TInner> inner,
			        Func<TOuter, TKey> outerKeySelector,
			        Func<TInner, TKey> innerKeySelector,
			        Func<TOuter, TInner, TResult> resultSelector
			) 
类型参数
TOuter
第一个序列中的元素的类型。
TInner
第二个序列中的元素的类型。
TKey
键选择器函数返回的键的类型。
TResult
结果元素的类型。
参数
outer
类型：System.Collections.Generic.IEnumerable<TOuter>
要联接的第一个序列。
inner
类型：System.Collections.Generic.IEnumerable<TInner>
要与第一个序列联接的序列。
outerKeySelector
类型：System.Func<TOuter, TKey>
用于从第一个序列的每个元素提取联接键的函数。
innerKeySelector
类型：System.Func<TInner, TKey>
用于从第二个序列的每个元素提取联接键的函数。
resultSelector
类型：System.Func<TOuter, TInner, TResult>
用于从两个匹配元素创建结果元素的函数。
返回值
类型：System.Collections.Generic.IEnumerable<TResult>
IEnumerable&lt;T&gt; that has elements of type TResult that are obtained by performing an inner join on two sequences." xml:space="preserve">一个具有 TResult 类型元素的 IEnumerable<T>，这些元素是通过对两个序列执行内部联接得来的。
使用说明
在 Visual Basic 和 C# 中，可以在 IEnumerable<TOuter> 类型的任何对象上将此方法作为实例方法来调用。当使用实例方法语法调用此方法时，请省略第一个参数。有关详细信息，请参阅 扩展方法 (Visual Basic) 或 扩展方法（C# 编程指南）。
 
先上一个MSDN的例子。
       
         public static void JoinEx1()         
         {
	       Person magnus = new Person { Name = "Hedlund, Magnus" };
	       Pet barley = new Pet { Name = "Barley", Owner = terry };
	       Person terry = new Person { Name = "Adams, Terry" };
	       Person charlotte = new Person { Name = "Weiss, 
	       Charlotte" };
	       Pet boots = new Pet { Name = "Boots", Owner = terry };
	       Pet whiskers = new Pet { Name = "Whiskers", 
	       Owner = charlotte};
	       Pet daisy = new Pet { Name = "Daisy", Owner = magnus };

        List<Person> people = new List<Person> 
        { magnus, terry, charlotte };
        List<Pet> pets = new List<Pet> 
        { barley, boots, whiskers, daisy };

        // Create a list of Person-Pet pairs where 
        // each element is an anonymous type that contains a
        // Pet's name and the name of the Person that owns the Pet.
        var query =
            people.Join(
                        pets,//需要Join的另一个数据源
              person => person,//自己用来比较的key， lambda 表达式
         pet => pet.Owner,//另一个数据源用来比较的key， lambda 表达式
    (person, pet) =>new { OwnerName = person.Name, Pet = pet.Name } 
                   //想要取出来的数据，支持匿名对象， lambda 表达式);
        foreach (var obj in query)
        {
          Console.WriteLine( "{0} - {1}",obj.OwnerName,obj.Pet);
        }
    }

 LINQ的Join
同样是上面的数据，如果换成LINQ的Join，写法如下：
            var query = from person in people // 第一个数据源
                      join pet in pets            //第二个数据源
                      on person equals pet.Owner  //Join条件
                      select  new { OwnerName = person.Name, Pet = pet.Name };  
                      //要到得到的数据。


上面两种写法得到的结果都是内链接结果，至于左连接、右连接、外连接....
等我下次有心情再更新吧，再不走家里的键盘又要坏了。 

一些资料链接：
https://msdn.microsoft.com/zh-cn/library/bb311040.aspx
https://msdn.microsoft.com/zh-cn/library/bb534675%28v=vs.110%29.aspx?f=255&MSPPError=-2147217396
http://www.cnblogs.com/Ivony/archive/2008/08/18/1270555.html
http://www.cnblogs.com/Ivony/archive/2008/08/28/1278643.html
http://www.cnblogs.com/Ivony/archive/2008/10/14/1309807.html ; 


LINQ GroupJoin 实现左连接
var queryGroup = from person in people // 第一个数据源
                                join pet in pets //第二个数据源
                                on person equals pet.Owner into ps //加了into,华丽变身GroupJoin
                                select new { OwnerName = person.Name, Pet = ps }; //要到得到的数据。 