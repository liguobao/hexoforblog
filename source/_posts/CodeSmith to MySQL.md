---
layout: post
title: CodeSmith for MySQL template
category: CodeSmith
---


对于.NET平台上的代码生成器来说，codesmith是一个非常好的选择。

<br> 
以前在学院实验室用的都是SQL server数据库，老师给的一套codesmith模板用来生成model/DAL/BLL很是方便。
<br> 
不过后来放弃SQL server 投入MySQL之后，刚开始都是手写SQL，还是很痛苦的。
<br> 
再后来又去找MySQL codesmith模板,这个对应的资料就不多了。不过最后还是找到了一套不错的，凑合能用。起初也懒，codesmith语法不熟，就没想过去修改一下了。
<br> 最近又要用到这套东西，于是决定还是去修改一番，更便于使用。

这个文章就主要讲一下修改过程，顺便说一下codesmith的简单语法。


先说一下操作步骤：

把模板的文件夹扔到codesmith模板文件的路径下，接着打开Codesmith，找到刚扔过去的文件夹，选择Main.cst,右键-execute-选择对应的MySQL库-选中表。
<br>（注：codesmith连接MySQL有问题的话，
<br>移步这里解决 [CodeSmith 连接MySQL数据库报“can't find .net framework data provider”](http://codelover.link/2016/02/25/CodeSmith%E8%BF%9E%E6%8E%A5MySQL%E6%8A%A5%E9%94%99%E2%80%9C%E6%89%BE%E4%B8%8D%E5%88%B0%E8%AF%B7%E6%B1%82%E7%9A%84%20.Net%20Framework%20Data%20Provider%E3%80%82%E5%8F%AF%E8%83%BD%E6%B2%A1%E6%9C%89%E5%AE%89%E8%A3%85%E3%80%82%E2%80%9D%E8%A7%A3%E5%86%B3%E6%96%B9%E6%B3%95/))

如下图：
![1](http://a1.qpic.cn/psb?/V13bZOxq1m5DxB/1XlywVcAyPW6y*6sO1QU.gkuidIqPkx.f70JsaijlU0!/b/dIwBAAAAAAAA&bo=MQIcAjECHAIDCSw!&rf=viewer_4)



然后点击Generate就能顺利生成model/dal/bll了。


生成代码结构如下：
![2](http://a3.qpic.cn/psb?/V13bZOxq1m5DxB/ikI1fSxAYklZgi1PmtPVTQqpHcZ0KTQHgN7LrPJaKNo!/b/dG4AAAAAAAAA&bo=fgLGAH4CxgADCSw!&rf=viewer_4)



<p>这样操作没什么问题，顺利生成了我们要的model/dal/bll了，然后....我懒嘛。
每次都要把表一个个选一次，麻不麻烦啊。然后就想了，能不能改一下模板呢。于是便开始google相关资料了。找到了几个相关文章，参考这就开始改造了。
先看看原来的Main.cst里面写了撒。

```C#

<%@ CodeTemplate Language="C#" ResponseEncoding="UTF-8" 
TargetLanguage="Text" Src="" Inherits="" Debug="False" 
Description="Template description here." 
 Output="None"%>
<%@ Register Name="Models" Template="DBMad.Models.cst" 
	MergeProperties="False" ExcludeProperties="" %>	
<%@ Register Name="DAL" Template="DBMad.DAL.cst" 
MergeProperties="False" ExcludeProperties="" %> 
<%@ Register Name="BLL" Template="DBMad.BLL.cst" 
MergeProperties="False" ExcludeProperties="" %>

<%@ Property Name="SourceTable" 
Type="SchemaExplorer.TableSchema" Optional="False"%>
<%@ Property Name="RootNamespace" Default="Net.Itcast.CN" 
Type="System.String" Optional="False"%>

<%@ Assembly Name="SchemaExplorer" %>
<%@ Assembly Name="System.Data" %>
<%@ Import Namespace="SchemaExplorer" %>
<%@ Import Namespace="System.Data" %>
<script runat="template">
	private string _outputDirectory = String.Empty;
	[Editor(typeof(System.Windows.Forms.Design.FolderNameEditor), 
	typeof(System.Drawing.Design.UITypeEditor))] 
	[Description("The directory to output the results to.")]
	public string OutputDirectory 
	{
		get
		{		
			return _outputDirectory;
		}
		set
		{
			if (value != null && !value.EndsWith("\\"))
			{
				value += "\\";
		    }
			_outputDirectory = value;
		} 
	}
</script>

```


这一段基本就是在声明选项以及引用命名空间，表现出来的便是我们看到的下图：

![1](http://a1.qpic.cn/psb?/V13bZOxq1m5DxB/1XlywVcAyPW6y*6sO1QU.gkuidIqPkx.f70JsaijlU0!/b/dIwBAAAAAAAA&bo=MQIcAjECHAIDCSw!&rf=viewer_4)


``` C#

<%
    Models model = this.Create<Models>();
	model.ModelsNamespace = this.RootNamespace+".Model";
	model.TargetTable = this.SourceTable;
	model.RenderToFile(this.OutputDirectory+"Model/"+model.GetFileName(),true);
	

   DAL dal = this.Create<DAL>();
   dal.TargetTable = this.SourceTable;
   dal.ModelsNamespace = model.ModelsNamespace;
   dal.DALClassNameSurfix = "DAL";
   dal.DALNamespace =this.RootNamespace+".DAL";
   dal.RenderToFile(this.OutputDirectory+"DAL/"
   +dal.GetFileName(),true);

   BLL bll = this.Create<BLL>();
   bll.ModelsNamespace = model.ModelsNamespace;
   bll.DALClassNameSurfix = dal.DALClassNameSurfix;
   bll.DALNamespace = dal.DALNamespace;
   bll.BLLClassNameSurfix = "BLL";
   bll.BLLNamespace = this.RootNamespace+".BLL";
   bll.TargetTable = this.SourceTable;
   bll.RenderToFile(this.OutputDirectory+"BLL/"
   +bll.GetFileName(),true);

   Response.Write("ok,see "+this.OutputDirectory);
%>
```

这一段就是我们点击Generate之后执行的代码，基本功能就是调用
DBMad.Models.cst,DBMad.DAL.cst,DBMad.BLL.cst。
因为在上面声明数据源的时候，使用了SchemaExplorer.TableSchema，导致我们选择表的时候不能多选。代码如下：

<%@ Property Name="SourceTable" Type="SchemaExplorer.TableSchema" Optional="False"%>

这样一想，这个Main.cst就是一个可以处理单表的生成模板了，我们只要自己写一个可以多选表的模板，然后循环调用这个模板去生成，不就完事了？

找了一下资料，发现只需要把上面的选项Type改一下，便可以多选表了。

如下：

<%@ Property Name="SourceTables" Type="SchemaExplorer.TableSchemaCollection" Default="" Optional="False" Category=""%> 


整体代码如下：

``` C#
<%@ CodeTemplate Language="C#" ResponseEncoding="UTF-8" 
TargetLanguage="Text" Src="" Inherits="" Debug="False" 
Description="Template description here." Output="None"%>
<%@ Property Name="SourceTables" 
Type="SchemaExplorer.TableSchemaCollection" Default="" 
Optional="False" Category=""%> 
<%@ Register Name="SE" Template="CreatSingleTable.cst" 
MergeProperties="False" ExcludeProperties="" %> 
<%@ Property Name="RootNamespace" Default="Net.Itcast.CN" 
Type="System.String" Optional="False"%>
<%@ Assembly Name="SchemaExplorer" %> 
<%@ Assembly Name="System.Data" %>
<%@ Import Namespace="SchemaExplorer" %> 
<%@ Import Namespace="System.Data" %> 
<%@ Import Namespace="System.Collections" %> 
<script runat="template">
	private string _outputDirectory = String.Empty;
	[Editor(typeof(System.Windows.Forms.Design.FolderNameEditor), 
	typeof(System.Drawing.Design.UITypeEditor))] 
	[Description("The directory to output the results to.")]
	public string OutputDirectory 
	{
		get
		{		
			return _outputDirectory;
		}
		set
		{
			if (value != null && !value.EndsWith("\\"))
			{
				value += "\\";
		    }
			_outputDirectory = value;
		} 
	}
</script>

<% 
foreach(TableSchema ts in SourceTables) 
{ 
SE s = new SE(); 
   s.SourceTable = ts; 
   s.RootNamespace = RootNamespace;
   s.OutputDirectory = OutputDirectory;
   s.Render(this.Response); 
} 
%> 
<script runat="template"> 
</script> 

```

前面一部分还是一样的声明，

<%@ Property Name="SourceTables" Type="SchemaExplorer.TableSchemaCollection" Default="" Optional="False" Category=""%> 

这一句把选项类型修改成可多选的（既 集合）。
效果如下图：
![3](http://a2.qpic.cn/psb?/V13bZOxq1m5DxB/CGQI8amVPz7K7Vyb05kp*0ti3mbnPWz4Q5xdUPBQSl0!/b/dHIBAAAAAAAA&bo=MQK0AjECtAIDCSw!&rf=viewer_4)



``` C#
<% 
foreach(TableSchema ts in SourceTables) 
{ 
SE s = new SE(); 
   s.SourceTable = ts; 
   s.RootNamespace = RootNamespace;
   s.OutputDirectory = OutputDirectory;
   s.Render(this.Response); 
} 
%> 
<script runat="template"> 
</script> 

```


这一段代码便是获取刚得到的表集合，遍历集合然后依次调用之前的单表生成模板。


到这里差不多已经完成了我要的效果，选择多表，实现一次生成所有的表对应的model/dal/bll。


这个效果基本就是我要的了，但是后来又发现，model里面的字段居然没有注释，我在建表的时候写了字段注释的呀。

打开model的cst文件之后发现，模板并没有做注释这个工作。
代码如下：

``` C#
<%@ CodeTemplate Language="C#" TargetLanguage="C#" 
Src="ToolsCodeTemplate.cs" Inherits="ToolsCodeTemplate"%>
<%@ Property Name="TargetTable" Type="SchemaExplorer.TableSchema" 
Category="Context" Description="TargetTable that the object is 
based on." %>
<%@ Property Name="ModelsNamespace" Default="Model" 
Type="System.String" Category="Context" Description="TargetTable 
that the object is based on." %>
<%@ Assembly Name="SchemaExplorer" %>
<%@ Assembly Name="System.Data" %>
<%@ Import Namespace="SchemaExplorer" %>
<%@ Import Namespace="System.Data" %>
<% PrintHeader(); %>
using System;
using System.Collections.Generic;
using System.Text;

namespace <%= ModelsNamespace %>
{	
	[Serializable()]
	public class <%= GetModelClassName() %>
	{
	    <% 
		foreach (ColumnSchema column in TargetTable.Columns)
	   {
		%>
			private <%= GetPropertyType(column) %>  _<%= 
			GetPropertyName(column) %>;			
		<%
		}
		%>
	    
		<% 
		foreach (ColumnSchema column in TargetTable.Columns)
		{
		%>
			public <%= GetPropertyType(column) %> <%= 
			GetPropertyName(column) %>
			{
				 get { return _<%= GetPropertyName(column) %>; }
	             set { _<%= GetPropertyName(column) %> = value; }
			}
		<%
		}
		%>	
	}
}
<script runat="template">
public string GetModelClassName()
{
	return GetModelClassName(TargetTable);
}

public override string GetFileName()
{
	return this.GetModelClassName(this.TargetTable) + ".cs";
}

</script>
```

获取表中字段名使用的是GetPropertyName(column)，咦，在哪实现了这个东西呢？回去翻一下文件，哦，还有一个ToolsCodeTemplate.cs文一直没管呢。

果然，GetPropertyName(column)在这里。

``` C#
public string GetPropertyName(ColumnSchema column)
{

    return GetNameFromDBFieldName(column);
}
public string GetNameFromDBFieldName(ColumnSchema column)
{
	return column.Name;
}

```
读取列名就是这么简单，那么我们对应写一个函数读取一下列注释，然后再model里面调用一下不好了。

又查了一下资料，

```C#
    public string GetColumnComment(ColumnSchema column)
    {
         return column.Description;
    }

```
嗯，理论上这样是可以的...
然而，我想多了。倒腾了好久，这个属性值都是空的...
google了一圈之后发现，原来是SchemaExplorer.MySQLSchemaProvider.dll 里面压根没实现读取列注释的实现....


不过也有对应的解决方法：

[完美解决CodeSmith无法获取MySQL表及列Description说明注释的方案](http://www.cnblogs.com/LonelyShadow/p/4147743.html)

把DLL替换一下就好了。

最后附上模板连接:[CodeSmith-for-MySQL-Template](https://github.com/liguobao/CodeSmith-for-MySQL-Template)



注：

1. 模板会把MySQL的表名前三个字符截取掉，建议把表明设置为tbl开头，或者自行修改模板文件。
2. 想让字段注释生效记得替换SchemaExplorer.MySQLSchemaProvider.dll(替换前记得备份！)

