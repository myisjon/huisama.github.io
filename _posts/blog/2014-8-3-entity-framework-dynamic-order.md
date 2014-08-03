---
layout: default
title: Entity Framework 动态排序
---
# Entity Framework 动态排序	

## 背景
在EF中, 需要对查询结果排序时, 我们可以如下操作:

	var q = from it in dc.StudentInfo
		orderby it.name, it.type
		select it;

或者是:

	var q = dc.StudentInfo.OrderBy(it => it.name).ThenBy(it => it.type);

这是常规的情况, 但是很多时候, 我们需要在不确定排序字段的名字/类型/数目的情况下给查询排序.
例如表格中, 我们允许用户通过点击表头的标题栏给结果自定义排序.
我们希望能够有类似如下的操作:

	var sortField = textBox.Text;
	var q = dc.StudentInfo.OrderBy(sortField);

但是EF并不允许直接通过字符串列名进行排序. 
之后通过网上搜索, 查询, 以及自己的尝试, 做出了大致的解决方案.

## 准备
测试环境是 .NET 4.0 控制台程序 + Entity Framework 6 (Code First) + SQL Server

	public class BlogContext : DbContext
    {
        public virtual DbSet<StudentInfo> StudentInfo { get; set; }
    }

	public class StudentInfo
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public int Height { get; set; }
    }

1个简单的 Context 类和 StudentInfo (string类型的Name 和 int类型的 Height)作为模型.

## 前奏
略去楼主 google 的辛苦过程, 先上一段简短的代码.

	static void Main(string[] args)
	{
		using (var dc = new BlogContext())
		{
			// initDb();
			var query = from it in dc.StudentInfo
					select it;

			var param = Expression.Parameter(typeof(StudentInfo), "o");
			Expression body = Expression.PropertyOrField(param, "Name");
			var lambda = Expression.Lambda<Func<StudentInfo, string>>(body, param);

			var orderedQuery = query.OrderBy(lambda);
			Console.WriteLine(orderedQuery.ToString());
		}
	}

	// output
	///////////////////

	SELECT
    [Extent1].[Id] AS [Id],
    [Extent1].[Name] AS [Name],
    [Extent1].[Height] AS [Height]
    FROM [dbo].[StudentInfoes] AS [Extent1]
    ORDER BY [Extent1].[Name] ASC

这段代码, 核心在于 `param` `body` `lambda` 3个变量组成了1个动态的 `o => o.Name` 表达式. 其作为 OrderBy的参数, 实现"Name"的排序.
底下输出的SQL告诉我们, 它已经很好的完成自己的任务(按照Name排序), 之后我们的修改都将在此基础上进行.

## 完善

以上代码的问题在于`Expression.Lambda<Func<StudentInfo, string>>(body, param);` 中第二个泛型参数 `string`, 它是 Name字段的类型, 但是既然是 **动态**排序, 那么字段名称和类型应该是**未知** (我们不应该知道"Name"的类型是 `string`).

1. 使用`dynamic`代替`string`, 代码依然能够编译和正常执行. 但是测试发现, 使用`dynamic` 在字段的类型为**值类型**时无法正常运行(但是可以通过编译), 会得到类型为“System.XX”的表达式不能用于返回类型“System.Object” 的错误, 例如说, 上例排序字段"Name" 改为"Height"(int 类型)时.

2. 修改为 `var lambda = Expression.Lambda(body, param)`, 删除了 Expression.Lambda的泛型参数, lambda 变量的**静态类型**也因此退化为 `LambdaExpression` (之前是 `Expression<Func<StudenInfo, X>>`, 但是动态类型, 还是保持原样), 这样不论引用类型还是值类型都能够被支持. 但因为不符合 OrderBy的调用参数声明, 代码无法通过编译.

最终解决方案是使用 第2种处理 配合 反射机制调用OrderBy, 用反射来调用OrderBy是因为反射能够跳过编译时的类型检查, 让代码能够正常编译.

以下是新的代码

	static void Main(string[] args)
	{
		using (var dc = new BlogContext())
		{
			// initDb();
			var query = from it in dc.StudentInfo
						select it;

			var param = Expression.Parameter(typeof(StudentInfo), "o");
			Expression body = Expression.PropertyOrField(param, "Name");
			var lambda = Expression.Lambda(body, param);

			MethodInfo orderbyMethod = GetOrderbyMethodInfo();
			orderbyMethod = orderbyMethod.MakeGenericMethod(typeof(StudentInfo), body.Type);
			var orderedQuery = (IOrderedQueryable<StudentInfo>)orderbyMethod.Invoke(null, new object[] { query, lambda });

			Console.WriteLine(orderedQuery.ToString());
		}
	}

	// output
	////////////////////////

	SELECT
    [Extent1].[Id] AS [Id],
    [Extent1].[Name] AS [Name],
    [Extent1].[Weight] AS [Weight],
    [Extent1].[Height] AS [Height]
    FROM [dbo].[StudentInfoes] AS [Extent1]
    ORDER BY [Extent1].[Name] ASC

和第一版的 代码改变在于:

1. 去除了 `Expression.Lambda`的泛型参数. 
2. 由`query.OrderBy` 改为了通过反射调用函数 `OrderBy`.

因为 反射获取 Queryable.OrderBy 还是一个麻烦事儿, 所以暂用 `GetOrderbyMethodInfo()` 代替的其中的过程. 这之后会说.

至此, 我们的解决方案大致就完成了, 可以使用 "Name" "Height" "Id" 这样的字符串名称来对我们的查询排序.

## 结尾

方案已经能够工作, 但我们还需要更加简洁. 类似于以下:
	
	var query = from it from dc.StudentInfo;
				select it;

	var rules = new OrderingRule[] {
		new OrderingRule("Name", OrderingRuleType.Asc),
		new OrderingRule("Height", OrderingRuleType.Desc)};

	query.ApplyOrdering(rules); // 

这还需要更多的封装, 涉及到很多的细节. 例如`OrderBy` 和 `ThenBy`, 以及 各自的正序和倒序(一共4个方法, 还有4个重载). 因为它们都是泛型静态方法, 我找了很久都没有找到简单的反射方法. 只能够遍历`Queryable`所有的方法通过 方法名和参数个数筛选出来. 如果有朋友们知道更好的方法可以告诉我~

此外, 因为 EF 的查询对象同时实现了`IQueryable`和`IOrderedQueryable`(即使未曾调用过OrderBy), 因此无法知道应该调用`OrderBy`还是`ThenBy`, 我也使用了一些变通的方法, 这一点如果有朋友知道也请告诉我~

整体代码请访问 [https://github.com/huisama/EntityFrameworkDynamicOrder](https://github.com/huisama/EntityFrameworkDynamicOrder)
里面包含一个 DynamicOrder类, 以及示例代码. 

这是我第一次使用Git和写博客, 评论和联系方式会随后加上~

---
### 2014-8-3
---