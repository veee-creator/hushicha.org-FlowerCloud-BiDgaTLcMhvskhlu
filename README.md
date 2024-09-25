
前言
最近比较闲,(项目要转Java被分到架构组,边缘化人员,无所事事 哈哈哈哈)


记录一下前段时间用到的.NET框架下**采用并行策略充分利用多核CPU**进行优化的一个方法


起因是项目中有个结算的方法,需要汇总一个月的数据在内存中进行计算,统计,分组 ,然后产生新的数据


在某个客户那部署后发现,这个方法执行的效率很低,监控发现数据从数据库查询出来 很快(因为数据库单独一台服务器)


然后通过top查看服务器的CPU就跑到了100%.内存正常,查了下CPU的型号 emm...很烂 但是好在核心很多(毕竟服务器级的U)..


查看服务器核心数 是在16个. Linux用top命令看的话,理论上CPU跑到1600%才算吃满,但是程序只吃了单个核.


等于1人干活 15人在吃瓜呀...如图:


![](https://img2024.cnblogs.com/blog/653851/202409/653851-20240924153017559-2054029548.png)


 然后查看了代码,发现结算的计算这一块代码是在单个foreach中进行顺序计算,所以决定用.NET提供的并行任务库(TPL)进行优化.


优化完成后,从之前的结算直接导致线程超时异常 变成 大概在20秒左右就结算完成.获得了巨大的提升.


正文
# 1 .NET 中的并行编程简介


在硬件发展迅速的今天.有太多的个人电脑和服务器级CPU都拥有多个 CPU 内核，为了方便多个线程能够同时执行。 充分利用硬件，就可以利用并行编程对代码进行并行化，以将工作分摊在多个处理器上。


以前，并行化需要自行开启子线程,维护锁等各种繁琐操作。但是从 .NET Framework 4 中引入的TPL简化了并行开发。 我们只需要通过简单的修改,就可以编写高效、细化且可伸缩的并行代码，而不必直接处理线程或线程池。


下图是官方文档的截图,简单的说明了 .NET 中的并行编程体系结构:


我们可以看到Parallel 就是在线程处理上加了一层封装好的算法,让我们处理并行多线程更简单


![](https://img2024.cnblogs.com/blog/653851/202409/653851-20240924154058563-1181101347.png)


 


# 2\. 并行任务库(TPL)


任务并行库 (TPL) 是 [System.Threading](https://learn.microsoft.com/zh-cn/dotnet/api/system.threading) 和 [System.Threading.Tasks](https://learn.microsoft.com/zh-cn/dotnet/api/system.threading.tasks):[westworld加速](https://tianchuang88.com) 空间中的一组公共类型和 API。


TPL 的目的是通过简化将并行和并发添加到应用程序的过程来提高开发人员的工作效率。


TPL 动态缩放并发的程度以最有效地使用所有可用的处理器。


此外，TPL 还处理工作分区、[ThreadPool](https://learn.microsoft.com/zh-cn/dotnet/api/system.threading.threadpool) 上的线程调度、取消支持、状态管理以及其他低级别的细节操作。


通过使用 TPL，你可以在将精力集中于程序要完成的工作，同时最大程度地提高代码的性能。


(以上来自于官方文档,我觉得已经讲的很详细了)


那么接下来,我们就编写一个并行任务的示例,来看看效果:


首先,并行任务库提供了两个方法 一个Parallel.ForEach  一个Parallel.For 用法都差不多,这里我们用Parallel.For做实验


先创建两个方法,代码如下:




```
 //创建顺序执行方法
 static List<dynamic> AddModelSequential(int modelCount)
 {
     var list = new List<dynamic>();
     //为了增加循环复杂性,里面嵌套一个循环
     for (int i = 0; i < modelCount; i++)
     {
         int f = 0;
         for (int j = 0; j < 5000; j++)
         {
             f++;
         }
         list.Add(new { bbb = i, aaa = "1", ccc = f });
     }
     return list;
 }
 //创建并行执行方法
 static List<dynamic> AddModelParallel(int modelCount)
 {
     var list = new List<dynamic>();
     Parallel.For(0, modelCount, i =>
     {
         int f = 0;
         //为了增加循环复杂性,里面嵌套一个循环
         for (int j = 0; j < 5000; j++)
         {
             f++;
         }
         list.Add(new { bbb = i, aaa = "1",ccc= f});
     });
     return list;
 }
```


 


接着执行两个方法,都跑10W条数据,并记录执行时间.如下:




```
 static void Main(string[] args)
 {

     Console.Error.WriteLine("执行顺序循环...");
     Stopwatch stopwatch = new Stopwatch();
     stopwatch.Start();

     AddModelSequential(1000000);
     stopwatch.Stop();
     Console.Error.WriteLine("顺序循环时间（毫秒）: {0}",
                             stopwatch.ElapsedMilliseconds);

     stopwatch.Reset();
     Console.Error.WriteLine("执行并行循环...");
     stopwatch.Start();
     var paradata= AddModelParallel(1000000);
     stopwatch.Stop();
     Console.Error.WriteLine("并行循环时间（毫秒）: {0}",
                             stopwatch.ElapsedMilliseconds);
     Console.ReadLine();
 }
```


本人是I9 12代CPU 逻辑处理器有20个,得到结果如图:


![](https://img2024.cnblogs.com/blog/653851/202409/653851-20240924161430802-1131939300.png)


性能提升20倍..


由于在开发机上跑的东西比较多,对于CPU的使用情况,监控不是很清楚,我们掏出..阿里云99元包邮的2核2G的服务器..来看看效果.


![](https://img2024.cnblogs.com/blog/653851/202409/653851-20240924161729585-1891136718.png)


我们可以明显看到在2核机上 性能大概也有接近一倍的提升


通过top命令,可以明显的监听到CPU的使用情况


在跑第一个循环的时候,CPU 100%,单核吃满,如图:


![](https://img2024.cnblogs.com/blog/653851/202409/653851-20240924161939715-1393793363.png)


跑第二个循环的时候,第2颗CPU就开始参与进来了,如图:


![](https://img2024.cnblogs.com/blog/653851/202409/653851-20240924162027478-777083548.png)


所以在合适的情况下(注意,这里是合适的情况)


程序中采用并行任务库充分的利用服务器的多核性能可以使运行效率有很大的提升.


 


 


# 3\. 并行PLINQ


PLINQ 是 LINQ 的一组扩展


它允许在运行代码的计算机上使用多个处理器或内核对支持 IEnumerable 接口的集合并行执行查询。


这可以显著减少处理大型数据集或执行复杂计算所需的时间


**注意,这里可以看到 PLINQ只支持 IEnumerable的接口,所以linq to sql时的表达式树是不支持的,如果使用则会导致全表查询到内存中**


使用方式也很简单,在数据集处理之前加上AsParallel方法即可,如下:




```
//LINQ
var results = from item in dataSource
              where item.SomeCondition()
              select item.SomeTransformation();
//PLINQ
var parallelResults = from item in dataSource.AsParallel()
                      where item.SomeCondition()
                      select item.SomeTransformation();
```


 


PLINQ的使用场景比较特殊,**目前demo中我还没反映出来比LINQ要快(甚至LINQ比PLINQ要快很多).**


所以我们在用的时候一定要考虑到以下几点:


* 并不总是更快：虽然 PLINQ 可以说是可以提高某些复杂查询的性能，但并非所有操作都会有明显收益。线程管理和同步产生的开销有时会使 PLINQ 查询比其顺序查询慢，尤其是对于小型数据集或计算复杂度较低的操作。
* 开销：并行化会带来开销，例如任务调度和线程之间的切换。对非 CPU 密集型的小型集合或操作，这些开销可能会抵消并行化的好处，从而使 PLINQ 查询比标准 LINQ 查询慢。
* 排序：默认情况下，PLINQ 不保证结果的顺序。如果排序很重要，则可以使用 AsOrdered 或 OrderBy 方法，但这可能会进一步降低并行化带来的性能提升。


综上所述,如果要用PLINQ一定要充分的进行测试与性能评估,一定要确定PLINQ有较大的提升时,才去使用.


 


.


 


