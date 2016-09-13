---
layout: post
title: .NET 类字段与类属性
category: .net
date: 2016-02-10 00:00:00
---
#.NET 类字段与类属性
@(.NET)[基础|字段|属性]

##字段
字段表示只读或可读/可写的数据值。
字段可以是静态的，这种字段被认为是类型状态的一部分。
字段也可以是实例（非静态），这种字段被认为是对象状态的一部分。 
强烈建议把字段声明为私有，防止类型或对象的状态被类型外部代码破坏。

##属性
属性允许用简单的、字段风格的语法设置或查询类型或对象的逻辑状态，同时保证状态不被破坏。
作用于类型称为静态属性，作用于对象称为实例属性。
属性可以无参，也可以有多个参数（相当少见，但集合类用的多）。 

	using System;
	
	public sealed class SomeType
	{                            //  1  
	// Nested class  
	   private class SomeNestedType { }                //  2  
	
	   // Constant, read­only, and static read/write field   
	   private const Int32 c_SomeConstant = 1;            //  3     
	
	   private readonly String m_SomeReadOnlyField = "2";     //  4    
	
	   private static Int32 s_SomeReadWriteField = 3;      //  5  
	
	   // Type constructor  
	   static SomeType() { }                                  //  6  
	
	   // Instance constructors  
	   public SomeType(Int32 x) { }                           //  7  
	
	   public SomeType() { }                                  //  8 
	
	   // Instance and static methods  
	   private String InstanceMethod() { return null; }       // 9   
	
	   public static void Main() { }                        // 10 
	
	   // Instance property  
	   public Int32 SomeProp
	   {                                // 11      
	       get { return 0; }                                // 12      
	       set { }                                          // 13  
	   }
	
	   // Instance parameterful property (indexer) 
	   public Int32 this[String s]
	   {                          // 14       
	       get { return 0; }                                // 15        
	       set { }                                          // 16  
	   }
	
	   // Instance event  
	   public event EventHandler SomeEvent;                  // 17  
	}

![enter image description here](https://kekaeq-ch3301.files.1drv.com/y3meEWWaK-o9SNfr_fT71Xr3YrPqO1LIswWqMvlHyUxWeH8P0PtXsQlfRkDnGshlMJIy2gPsxNet14efOPOuX-dHmZhCTg8PXyELJR9tnayye4LeEQ6F997b8pSI84wBR6nmmOF8IAr92oKWk36-f8alkEj9TrDQbiMoKGQwO5MTFY/20150105.jpg?psid=1)


![enter image description here](https://kekaeq-ch3301.files.1drv.com/y3mN8FseIEbAaQfT8ynEIc4nYsOUK0p0IN7Hp39imVSZXNQXlSfAYmvaC-9bDM3Hq6rPCV1XgrgvoST8wXejwARXaDmXptZpb_nyWwUzWK1rzaJ6fSsYfnP0icRKUclaVxnlltOJSQiuTFO7_fCfabmv0AsrgL5sLo6GdVJmpd-L10/QQ%E5%9B%BE%E7%89%8720151003165909.jpg?psid=1)