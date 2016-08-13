---
layout: post
title: .NET-lmabda避免修改绑定变量
category: .net
---

先看一段代码

		#region test1 闭包

        public static void test1()
        {
            int index = 0;
            Func<IEnumerable<int>> sequence =()=>GetEnumrableInt(index);

            index = 20;
            foreach(int n in sequence())
                Console.WriteLine(n);

            Console.WriteLine("Done");

            index = 100;
            foreach (int n in sequence())
                Console.WriteLine(n);
        }


        public static IEnumerable<int> GetEnumrableInt(int index)
        {
            List<int> l = new List<int>();
            for(int i=index;i<index+30;i++)
            {
                l.Add(i);
            }
            return l;
        }

        #endregion
上面一坨代码演示了在闭包中使用了外部变量，随即又在外部修改了这些变量的情况，得到的结果是输出了20-50的数，然后又输出了100-130之间的数。这种行为有点诡异，但是确实有存在的意义...(书本这样说的，我到觉得很少会用到。)

为了将查询表达式转换成可执行代码，C#编译器做了很多工作。一般而言，C#编译器将查询和lambda表达式转换成 "静态委托"、"实例委托" 或 "闭包"。编译器将根据lambda表达式中的代码选择一种实现方式。选择哪种方式依赖于lambda表达式的主体（body）。这看上去似乎是一些语言上的实现细节，但它却会显著地影响到我们的代码。编译器选择何种实现将可能导致diamante行为发生微妙的变化。

并不是任何的lambda表达式都会生成同样结构的代码。

对于编译器来说，最简单的一种行为是为以下形式的代码生成委托。  

        //我们的lambda表达式
        public static void test2()
        {
            int[] someNum = {0,1,2,3,4,5,6,7,8,9,10 };

            IEnumerable<int> ans = from n in someNum
                                   select n * n;

            foreach (int i in ans)
                Console.WriteLine(i);

        }

编译器将使用静态委托来实现n*n的lambda表达式，其为上面代码生成的代码如下：

         //编译器为我们的lambda生成的代码
        #region 等价于 test2()
        private static int HiddenFunc(int n)
        {
            return n * n;
        }
        
        //静态委托
        private static Func<int, int> HiddenDelegate;

        public void test2_1()
        {

            int[] someNum = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

            if(HiddenDelegate==null)
            {
                HiddenDelegate = new Func<int, int>(HiddenFunc);
            }
            IEnumerable<int> ans = someNum.Select<int, int>(HiddenDelegate);

          foreach(int i in ans)
              Console.WriteLine(i);

        }
        #endregion

这个lambda表达式主体部分并没有访问任何的实例变量或者局部变量。lambda表达式仅仅访问了它的参数。对于这种情况，C#编译器将创建一个静态方法，作为委托的目标。这也是编译器执行的最简单的一种处理方式。若表达式可以通过私有的静态方法实现，那么编译器将生成该私有的静态方法以及相对应的委托定义。对于上面的代码例子中的情况以及仅访问了静态变量的表达式，编译器都会采用这样的方案。


接下来介绍另一种较为简单的情况：
lambda表达式需要访问类型的实例变量，但无需访问外层方法中的局部变量。
				
    public class ModFilter
    {
        private readonly int modules;

        public ModFilter(int mod)
        {
            modules = mod;
        }


        public IEnumerable<int> FindValues(IEnumerable<int> sequence)
        {

            return from n in sequence
                   where n % modules == 0 //新添加的表达式
                   select n * n;  //和前面的例子是一样的
        }
    }



/* 

在这种情况下，编译器将为表达式创建一个实例方法来包装该委托。
其基本概念和前一种情况一致，只是这里使用了实例方法，以便读取并修改当前对象的状态。
与静态委托的例子一样，这里编译器将把lambda表达式转换成我们熟悉的代码。其中包含委托的定义以及方法调用。
如下：

*/



    public class ModFilter_Other
    {
        private readonly int modules;


        //实例方法
        private bool WhereClause(int n)
        {
            return ((n%this.modules) ==0);
        }


        private static int SelectClause(int n)
        {
            return n * n;
        }

        private static Func<int, int> SelectDelegate;




        public ModFilter_Other(int mod)
        {
            modules = mod;
        }


        public IEnumerable<int> FindValues(IEnumerable<int> sequence)
        {
            if(SelectDelegate==null)
            {
                SelectDelegate = new Func<int, int>(SelectClause);
            }

            return sequence.Where<int>(
                new Func<int, bool>(this.WhereClause)).
                Select<int, int>(SelectClause);
        }
    }

概括来说便是：lambda表达式中的代码访问了对象实例中的成员变量，那么编译器将生成实例方法来表示lambda表达式中的代码。其实这并没有什么奇特之处——编译器省去了我们的一些代码输入工作，代码也变得整洁很多，本质来说这还是普通的方法调用。





不过若是lambda表达式中访问到了外部方法中的局部变量或者方法参数，那么编译器将帮你完成很多工作。


这里会用到闭包。编译器将生成一个私有的嵌套类型，以便为局部变量实现闭包。

局部变量必须传入到实现了lambda表达式主体部分的委托里。

此外，所有由该lambda表达式执行的对这些局部变量所作的修改都必须能够在外部访问到。

当然，代码中内层和外层中共享的可能不止有一个变量，也可能不止一个的查询表达式。

我们来修改一下该实例方法，让其访问一个局部变量。


		  public class ModFilter
		  {
		        private readonly int modules;
		
		        public ModFilter(int mod)
		        {
		            modules = mod;
		        }
		
		
		        public IEnumerable<int> FindValues(IEnumerable<int> sequence)
		        {
		            int numValues = 0;
		
		            return from n in sequence
		                   where n % modules == 0 //新添加的表达式
		                   select n * n / ++ numValues; //访问局部变量
		        }
	      }

注意，select字句需要访问numValues这个局部变量。编译器为了创建这个闭包，需要使用嵌套类型来实现你所需要的行为。下面展示的是编译器为你生成的代码。




 	 public class ModFilter
     {
        private sealed class Closure
        {
            public ModFilter outer;

            public int numValues;

            public int SelectClause(int n)
            {
                return ((n * n) / ++this.numValues);
            }
        }



        private readonly int modules;


        //实例方法
        private bool WhereClause(int n)
        {
            return ((n % this.modules) == 0);
        }

        public ModFilter(int mod)
        {
            modules = mod;
        }


        public IEnumerable<int> FindValues(IEnumerable<int> sequence)
        {
            Closure c = new Closure();
            c.outer = this;
            c.numValues = 0;

            return sequence.Where<int>(
                new Func<int, bool>(this.WhereClause)).
                Select<int, int>(c.SelectClause);
        }
    }

在上面这段代码中，编译器专门创建了一个嵌套类，用来容纳所有将在lambda表达式中访问或修改的变量。实际上，这些局部变量将完全被嵌套类的字段所代替。lambda表达式内部的代码以及表达式外部(但仍在当前方法内)的代码访问的均是同一个字段，lambda表达式中的逻辑也被编译成了内部类的一个方法。

对于lambda表达式中将要用到的外部方法的参数，编译器也会以对待局部变量的方式实现：编译器将这些参数复制到表示该闭包的嵌套类中。

回到最开始的那个示例，这是我们应该可以理解这种看似怪异的行为了。变量index在传入闭包后，但在查询开始执行前曾被外部代码修改。也就是说，你修改了闭包的内部状态，然后还期待其能够回到从前的状态开始执行，显然这是不可能实现的。

考虑到延迟执行中的交互以及编译器实现闭包的方式，修改查询与外部代码之间绑定的变量将可能会引发错误的行为。
因此，我们应该尽量避免在方法中修改哪些将要传入到闭包中，并将在闭包中使用的变量。




