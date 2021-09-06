# sparksql_hajimei

1. 为 Spark SQL 添加一条自定义命令
   SHOW VERSION;->显示当前 Spark 版本和 Java 版本
   
   Add:
   ![8130C8F9-025E-4C33-901E-9094DB3DAFA8](https://user-images.githubusercontent.com/11592423/132137917-5f0a5fd5-eba2-4c34-bba4-be8824562346.png)
   ![B658866D-1395-4BF2-B7B1-53FDB79A6EE8](https://user-images.githubusercontent.com/11592423/132137957-aa5d8f35-d9a8-4f73-b74e-4a85d73f8886.png)
   ![07E3D439-9549-44A1-BA81-8F33C45F43A6](https://user-images.githubusercontent.com/11592423/132137971-fbbc8f2d-1218-49d8-aa19-95c4d5ff8630.png)

   
   Build:
   ![DEC3F9B1-2303-4226-B289-E69EF3DA92D3](https://user-images.githubusercontent.com/11592423/132137781-8810d70b-94c9-4381-b00d-5016ed736f0d.png)
   
   Result:
   ![534280B5-2C78-4695-B773-9E9D22610F95](https://user-images.githubusercontent.com/11592423/132138000-b9604dee-b41b-44e9-8b2f-6fcdd06669ab.png)



2. 构建 SQL 满足如下要求
   通过 set spark.sql.planChangeLog.level=WARN;查看
   - 2.1 构建一条 SQL，同时 apply 下面三条优化规则：
      - CombineFilters
      - CollapseProject
      - BooleanSimplification
   
   Commands:

   - bin/spark-sql
   - set spark.sql.planChangeLog.level=WARN;
   - create table sales(customerId string, productName string, amountPaid int);
   - select a.customerId from (select customerId, productName as pName, amountPaid from sales where 1 = “1” and amountPaid > 300) a where a.amountPaid < 500;

  Result:
  ![73BFAC86-6AE2-48D3-BF6A-C4F50E1F76D5](https://user-images.githubusercontent.com/11592423/132138146-6d489111-6b7c-4726-8424-f7a7ee116f1f.png)
  ![D7F7BAC8-C696-4766-96EB-C488F29733D3](https://user-images.githubusercontent.com/11592423/132138172-eeee58e5-ce85-4223-bc3c-7ec92b9f7e3d.png)
  
  我理解的CombineFilters:
  ![519615D7-8FE5-496D-AD3E-62BDB650A63C](https://user-images.githubusercontent.com/11592423/132138207-12f43e7f-c038-407c-9f33-8fc5e84176f2.png)



   - 2.2 构建一条 SQL，同时 apply 下面五条优化规则：
      - ConstantFolding
      - PushDownPredicates
      - ReplaceDistinctWithAggregate
      - ReplaceExceptWithAntiJoin
      - FoldablePropagation
         - Replace attributes with aliases of the original foldable expressions if possible.
   Other optimizations will take advantage of the propagated foldable expressions. For example,
   this rule can optimize
   {{{SELECT 1.0 x, 'abc' y, Now() z ORDER BY x, y, 3}}}
   to
   {{{SELECT 1.0 x, 'abc' y, Now() z ORDER BY 1.0, 'abc', Now()}}}
   and other rules can further optimize it and remove the ORDER BY operator.
  \*/
  object FoldablePropagation extends Rule[LogicalPlan] {
  def apply(plan: LogicalPlan): LogicalPlan = {
  CleanupAliases(propagateFoldables(plan).\_1)
  }

Commands:

- bin/spark-sql
- set spark.sql.planChangeLog.level=WARN;
- create table sales(customerId string, productName string, amountPaid int);
- (select a.productName , a.amountPaid + (10 + 20) , Now() z from (select distinct customerId, amountPaid , productName from sales) a where a.amountPaid>100 order by z) except (select customerId , amountPaid + (10 + 20) , Now() z from (select distinct customerId, amountPaid , productName from sales) );

Result:
- ConstantFolding
![E99C6C6A-B3E4-4A62-A60F-30F1EE946750](https://user-images.githubusercontent.com/11592423/132137420-b86caa5a-66f4-49e6-a02f-1eda2b7d724f.png)

- PushDownPredicates
![DD3B7DC9-5626-4B1D-B284-2529B981D04F](https://user-images.githubusercontent.com/11592423/132137451-cbfbc601-2bbb-4d12-bd5b-44f1dfefe366.png)

- ReplaceDistinctWithAggregate
![FC4DC033-2743-476A-A522-9D30E7743841](https://user-images.githubusercontent.com/11592423/132137461-f6652a86-84d8-4cfa-a558-2dbc6517da94.png)

- ReplaceExceptWithAntiJoin
![23A68847-C283-4B66-AC91-02C76179C2D3](https://user-images.githubusercontent.com/11592423/132137487-72680c41-b1fe-49a6-bdc6-e6fc61e944fc.png)

- FoldablePropagation
![059ECA84-0FA0-435F-989D-9E304CF3BF26](https://user-images.githubusercontent.com/11592423/132137502-a5871fbd-f290-4077-a22d-5b3154798277.png)


3. 实现自定义优化规则（静默规则） 
   - 第一步 实现自定义规则（静默规则，通过 set spark.sql.planChangeLog.level=WARN;确认执行到就行）
      - case class MyPushDown(spark: SparkSession) extends Rule[LogicalPlan] {
   def apply(plan: LogicalPlan): LogicalPlan = plan transform { …. }
   }
   - 第二步 创建自己的 Extension 并注入
      - class MySparkSessionExtension extends (SparkSessionExtensions => Unit) {
   override def apply(extensions: SparkSessionExtensions): Unit = {
   extensions.injectOptimizerRule { session =>
   new MyPushDown(session)
   }}}
   - 第三步 通过 spark.sql.extensions 提交
      - bin/spark-sql --jars my.jar --conf spark.sql.extensions=com.jikeshijian.MySparkSessionExtension
  
  Result:
  ![A89E8A16-DA89-4609-AE84-71F3E4134142](https://user-images.githubusercontent.com/11592423/132241246-86d20a22-421a-4c77-94b2-ae664b737ed3.png)

