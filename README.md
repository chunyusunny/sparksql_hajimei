# sparksql_hajimei

1. 为 Spark SQL 添加一条自定义命令
   SHOW VERSION;->显示当前 Spark 版本和 Java 版本

2. 构建 SQL 满足如下要求
   通过 set spark.sql.planChangeLog.level=WARN;查看
   2.1 构建一条 SQL，同时 apply 下面三条优化规则：
   CombineFilters
   CollapseProject
   BooleanSimplification

   2.2 构建一条 SQL，同时 apply 下面五条优化规则：
   ConstantFolding
   PushDownPredicates
   ReplaceDistinctWithAggregate
   ReplaceExceptWithAntiJoin
   FoldablePropagation
   /\*\*

- Replace attributes with aliases of the original foldable expressions if possible.
- Other optimizations will take advantage of the propagated foldable expressions. For example,
- this rule can optimize
- {{{
- SELECT 1.0 x, 'abc' y, Now() z ORDER BY x, y, 3
- }}}
- to
- {{{
- SELECT 1.0 x, 'abc' y, Now() z ORDER BY 1.0, 'abc', Now()
- }}}
- and other rules can further optimize it and remove the ORDER BY operator.
  \*/
  object FoldablePropagation extends Rule[LogicalPlan] {
  def apply(plan: LogicalPlan): LogicalPlan = {
  CleanupAliases(propagateFoldables(plan).\_1)
  }

Commands:

> bin/spark-sql
> set spark.sql.planChangeLog.level=WARN;
> create table sales(customerId string, productName string, amountPaid int);
> create table sales(customerId string, amountPaid int, productName string);
> (select a.productName , a.amountPaid + (10 + 20) , Now() z from (select distinct customerId, amountPaid , productName from sales) a where a.amountPaid>100 order by z) except (select customerId , amountPaid + (10 + 20) , Now() z from (select distinct customerId, amountPaid , productName from sales) );

3. 实现自定义优化规则（静默规则）
   第一步 实现自定义规则（静默规则，通过 set spark.sql.planChangeLog.level=WARN;确认执行到就行）
   case class MyPushDown(spark: SparkSession) extends Rule[LogicalPlan] {
   def apply(plan: LogicalPlan): LogicalPlan = plan transform { …. }
   }
   第二步 创建自己的 Extension 并注入
   class MySparkSessionExtension extends (SparkSessionExtensions => Unit) {
   override def apply(extensions: SparkSessionExtensions): Unit = {
   extensions.injectOptimizerRule { session =>
   new MyPushDown(session)
   }}}
   第三步 通过 spark.sql.extensions 提交
   bin/spark-sql --jars my.jar --conf spark.sql.extensions=com.jikeshijian.MySparkSessionExtension
