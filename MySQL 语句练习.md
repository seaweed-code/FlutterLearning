## MySQL 语句练习

数据表：

| 表   | 字段                         |
| ---- | ---------------------------- |
| 学生 | Student(SId,Sname,Sage,Ssex) |
| 课程 | Course(CId,Cname,TId)        |
| 教师 | Teacher(TId,Tname)           |
| 成绩 | SC(SId,CId,score)            |

 SQL语句

```sql
create table Student(SId varchar(10),Sname varchar(10),Sage datetime,Ssex varchar(10));
insert into Student values('01' , '赵雷' , '1990-01-01' , '男');
insert into Student values('02' , '钱电' , '1990-12-21' , '男');
insert into Student values('03' , '孙风' , '1990-12-20' , '男');
insert into Student values('04' , '李云' , '1990-12-06' , '男');
insert into Student values('05' , '周梅' , '1991-12-01' , '女');
insert into Student values('06' , '吴兰' , '1992-01-01' , '女');
insert into Student values('07' , '郑竹' , '1989-01-01' , '女');
insert into Student values('09' , '张三' , '2017-12-20' , '女');
insert into Student values('10' , '李四' , '2017-12-25' , '女');
insert into Student values('11' , '李四' , '2012-06-06' , '女');
insert into Student values('12' , '赵六' , '2013-06-13' , '女');
insert into Student values('13' , '孙七' , '2014-06-01' , '女');
 
create table Teacher(TId varchar(10),Tname varchar(10));
insert into Teacher values('01' , '张三');
insert into Teacher values('02' , '李四');
insert into Teacher values('03' , '王五');
 
 
create table Course(CId varchar(10),Cname nvarchar(10),TId varchar(10));
insert into Course values('01' , '语文' , '02');
insert into Course values('02' , '数学' , '01');
insert into Course values('03' , '英语' , '03');
 
create table SC(SId varchar(10),CId varchar(10),score decimal(18,1));
insert into SC values('01' , '01' , 80);
insert into SC values('01' , '02' , 90);
insert into SC values('01' , '03' , 99);
insert into SC values('02' , '01' , 70);
insert into SC values('02' , '02' , 60);
insert into SC values('02' , '03' , 80);
insert into SC values('03' , '01' , 80);
insert into SC values('03' , '02' , 80);
insert into SC values('03' , '03' , 80);
insert into SC values('04' , '01' , 50);
insert into SC values('04' , '02' , 30);
insert into SC values('04' , '03' , 20);
insert into SC values('05' , '01' , 76);
insert into SC values('05' , '02' , 87);
insert into SC values('06' , '01' , 31);
insert into SC values('06' , '03' , 34);
insert into SC values('07' , '02' , 89);
insert into SC values('07' , '03' , 98);
```

1. #####  **查询"01"课程比"02"课程成绩高的学生的信息及课程分数**

   ```mysql
   #同一个学生，01课程的分数 大于其 02课程的分数
   #条件语句中嵌套SQL子语句
   SELECT * FROM SC T0 WHERE T0.CId = '01' and T0.score > ( SELECT score FROM SC T1 WHERE T0.SId=T1.SId and T1.CId ='02' )
   
   #关联查询，在on语句中加入判断条件,
   #此处必须使用inner join,而不能使用left join or right join,因为我们只需要满足条件的数据
   SELECT * FROM SC T0 JOIN SC T1 on T0.SId = T1.SId and T0.CId = '01' and T1.CId='02' and T0.score > T1.score
   
   #关联查询，用两个已经缩小范围的表进行关联
   SELECT * FROM (SELECT * FROM SC T0 WHERE T0.CId = '01') T01 JOIN (SELECT * FROM SC T0 WHERE T0.CId = '02') T02 on T01.SId=T02.SId and T01.score > T02.score
   ```

2. **查询同时存在" 01 "课程和" 02 "课程的情况**

   ```mysql
   #条件语句中嵌套SQL子语句，使用Exist语句判断是否存在
   SELECT * FROM SC T0 WHERE EXISTS (SELECT * FROM SC T1 WHERE T1.SId=T0.SId and T1.CId='01')  and EXISTS (SELECT * FROM SC T2 WHERE T2.SId=T0.SId and T2.CId='02')
   
   #关联查询，在on中加入判断条件，和第一题中基本类似
   SELECT * FROM SC T0 JOIN SC T1 on T0.SId = T1.SId and T0.CId='01' and T1.CId='02'
   
   #利用Case end、IF增加2列分别表示该列是否是01，02，再利用group分组和聚合函数判断是否存在01、02
   SELECT T.SId FROM
   (SELECT *, IF(CId='01',1,0) Is01, #使用IF语句
    CASE CId
   	  WHEN '02' THEN 1
   	  ELSE  0
   END Is02
    FROM SC) T GROUP BY T.SId HAVING MAX(T.Is01) = 1 and MAX(T.Is02) = 1
    
   ```

3. **查询存在" 01 "课程但不存在" 02 "课程的情况**

   ```sql
   #在Where子语句中嵌套SQL语句
   SELECT * FROM SC T0 WHERE EXISTS (SELECT * FROM SC T1 WHERE T1.SId=T0.SId and T1.CId='01') and NOT EXISTS (SELECT * FROM SC T2 WHERE T2.SId=T0.SId and T2.CId='02')
   
   # (SELECT * FROM SC WHERE CId = '01')  查询存在01的
   # SELECT * FROM SC T0 WHERE NOT EXISTS (SELECT * FROM SC T1 WHERE T1.SId=T0.SId and T1.CId='02')查询不存在02的
   # A & B 所以只需要 A join B即可
   SELECT * FROM (SELECT * FROM SC WHERE CId = '01') T01 JOIN 
   (SELECT * FROM SC T0 WHERE NOT EXISTS (SELECT * FROM SC T1 WHERE T1.SId=T0.SId and T1.CId='02')) T02 on T01.SId=T02.SId
   ```

   

4. Fffddd

 