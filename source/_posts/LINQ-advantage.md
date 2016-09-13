---
layout: post
title: C#LINQ 优点 总结(转载)
category: .net
date: 2016-03-15 00:00:00
---


原文链接：http://www.cnblogs.com/c-jquery-linq-sql-net-problem/archive/2011/01/15/LINQ_Merit.html

这几天在读一本LINQ方面的书《Essential LINQ》,在这里和大家分享下。

由于对LINQ的深入总结需要大量的篇幅，因此在这里分成几个部分来讲。

（*我看《Essential LINQ》是英文版的，有些名词不能翻译成正统的中文解释请给予谅解）

LINQ的优点：

LINQ基本有以下七个优点，让我来一一举例说明：

#####1.Integrated：所谓的Integrated（集成化），LINQ是从以下方面体现集成的：

(1):把查询语法融入了C#(VB)这些语言中，让他变成了一种语法。这样就能和C#中的其他语法一样支持：

语句高亮显示，类型检查，允许使用debugger调试

(2):把以前复杂的查询前的工作都集成封装起来，让开发人员侧重于查询。

(3):集成后的语法更加的清晰易懂，可读性较高。

  
```
比较： 
//原来的格式
SqlConnection sqlConn = new SqlConnection(connectionString);>
sqlConn.Open();
SqlCommand command = new SqlCommand();
command.Connection = sqlConn;
command.CommandText = "Select * From Customer";
SqlDataReader dataReader = command.ExecuteReader(CommandBehavior.CloseConnection);
 
//LINQ的格式
NORTHWNDDataContext dc = new NORTHWNDDataContext();
var query = from c in dc.Customers
            select c;
```



#####2.Unitive：所谓Unitive(统一化)就指不管对任何类型外部和内部数据源(对象集合,xlm,数据库数据)都使用统一的查询语法。

使用统一化查询语言的好处在于以下几点：

你不用因为要使用不太熟悉的数据源而花很多精力去了解它，你可以快速简单的使用LINQ语法对起查询。
由于使用了统一的语法，可以使代码维护变的更加简单。
以下代码体现了LINQ的统一化：
```
//数据源:对象集合
var query = from c in GetCustomers()
            select c;
 
//数据源:SQL
var query1 = from c in dc.Customers
             select c;
//数据源:XML
var query2 = from c in customers.Descendants("Customer")
             select c;
```

         
#####3.Extensible：所谓Extensible(可扩展)指以下2个方面:

(1).可查询数据源的扩展。 LINQ提供了个LINQ provider model，你可以为LINQ创建或提供provider让LINQ支持更多的数据源。

(2).可扩展查询方法。开发者可以根据自己的需求为LINQ重写和扩展查询方法。

以下是些第三方的LINQ provider：

LINQ Extender, LINQ to JavaScript, LINQ to JSON, LINQ to MySQL, LINQ to Flickr, LINQ to Google

 

#####4.Declarative：所谓Declarative(声明式)，简单的来说指的是开发人员只要告诉程序做什么，程序自己判断怎么做。

Declarative programming(声明式编程)的优点体现在以下2点：

(1).提高了开发速度。因为开发者不用书写大量的代码来具体化执行步骤，只许告诉程序做什么。

(2).提高代码优化空间。因为开发者不用参与干涉对程序执行的具体步骤，这样就提供给编译器更多的空间去优化代码。

举例SQL来说，LINQ生成的SQL语句往往比一对SQL水平一般的开发者能写出更好的SQL语句。

比较Declarative programming 与 Imperative programming：
```
//声明式编程
List<List<int>> lists = new List<List<int>> { new List<int> { 1, 2, 3 }, new List<int> { 4, 5 } };
var query = from list in lists
            from num in list
            where num % 3 == 0
            orderby num descending
            select num;
 
//命令式编程
List<int> list1 = new List<int>();
list1.Add(1);
list1.Add(2);
list1.Add(3);
List<int> list2 = new List<int>();
list2.Add(4);
list2.Add(5);
List<List<int>> lists1 = new List<List<int>>();
lists1.Add(list1);
lists1.Add(list2);
 
List<int> newList = new List<int>();
foreach (var item in lists1)
      foreach (var num in item)
        if (num % 3 == 0)
            newList.Add(num);
newList.Reverse();

```

#####5.Hierarchical：所谓Hierarchical(层次化)指使用面向对象的方式抽象数据。

SQL是关系型数据库，它以关系的方式描述数据以数据的联系，但我们的程序设计成面向对象的因此我们在程序里得到的数据库数据往往都是

rectangular grid（平面的显示数据）。但是LINQ通过所谓的O-R Mapping方式，把关系型转换成对象与对象方式描述数据。

这样带来的好处是：开发者能直接以对象的方式去操作数据，对习惯面向对象的开发者来说面向对象模型更易理解。

 

#####6.Composable：所谓Composable(可组成)指LINQ可以把一个复杂的查询拆分成多个简单查询。

LINQ返回的结果都是基于接口：IEnumerable<T>，因此能对查询结果继续查询，而且LINQ具有延迟执行的特性因此拆分执行不会影响效率。

优点在于：

(1).方便调试。把复杂的查询拆分成简单的查询，然后逐个调试。

(2).便于代码维护。把代码拆分后能使代码变的更易理解。

以下代码体现了可组成性：
```
//以下代码体现了Composable
List<List<int>> lists = new List<List<int>> { new List<int>
 { 1, 2, 3 }, new List<int> { 4, 5 } };
 
var query1 = from list in lists
             from num in list
             select num;
 
var query2 = from num in query1
             where num % 3 == 0
             select num;
 
var query3 = from num in query2
             orderby num descending
             select num;
```

 

######7.Transformative：所谓Transformative(可转换)指的是LINQ能把一种数据源的内容转换到其他数据源。

方便用户做数据移植。

以下代码体现了转换的特性：
```
//把关系型数据转换成XML型
	var query = new XElement("Orders",
            from c in dc.Customers
            where c.City == "Paris"
            select new XElement("Order",
                new XAttribute("Address", c.Address)));
```

以上就是LINQ的几大优点，很高兴能在这里和大家分享。有任何不足之处请给予补充和纠正，谢谢光临小舍。

//2011/1/28 补充(LINQ TO SQL)

在LINQ TO SQL 方面，如果使用LINQ TO SQL可以有效的防止SQL注入，LINQ TO SQL 会把注入的代码当做无用的参数处理。


http://www.cnblogs.com/c-jquery-linq-sql-net-problem/archive/2011/01/15/LINQ_Merit.html 