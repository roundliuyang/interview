# SQL



**1.给定时间区间（begin，end），数据库字段startTime与endTime，现在要判断它们之间是否有交集。**

```shell
SELECT * FROM xxx
WHERE NOT ((endTime < begin) OR (startTime > end))
```





**2.Student、Course、Teacher、SC SQL查询练习题**

数据表

```sql
-- 创建库
CREATE DATABASE school;
-- 创建学生表
CREATE TABLE Student(
Sid VARCHAR(10) PRIMARY KEY, -- 学生编号
Sname VARCHAR(10),-- 学生姓名
Sage  DATETIME,-- 学生出生年月
Ssex  VARCHAR(10)-- 学生性别
)DEFAULT CHARSET = utf8;
INSERT INTO Student VALUES('01' , '赵雷' , '1990-01-01' , '男');
INSERT INTO Student VALUES('02' , '钱电' , '1990-12-21' , '男');
INSERT INTO Student VALUES('03' , '孙风' , '1990-05-20' , '男');
INSERT INTO Student VALUES('04' , '李云' , '1990-08-06' , '男');
INSERT INTO Student VALUES('05' , '周梅' , '1991-12-01' , '女');
INSERT INTO Student VALUES('06' , '吴兰' , '1992-03-01' , '女');
INSERT INTO Student VALUES('07' , '郑竹' , '1989-07-01' , '女');
INSERT INTO Student VALUES('09' , '张三' , '2017-12-20' , '女');
INSERT INTO Student VALUES('10' , '李四' , '2017-12-25' , '女');
INSERT INTO Student VALUES('11' , '李四' , '2017-12-30' , '女');
INSERT INTO Student VALUES('12' , '赵六' , '2017-01-01' , '女');
INSERT INTO Student VALUES('13' , '孙七' , '2018-01-01' , '女');

SELECT * FROM Student;

-- 创建教师表
CREATE TABLE Teacher(
Tid VARCHAR(10) PRIMARY KEY,-- 教师编号
Tname VARCHAR(10) -- 教师姓名
)DEFAULT CHARSET = utf8;
INSERT INTO Teacher VALUES('01' , '张三');
INSERT INTO Teacher VALUES('02' , '李四');
INSERT INTO Teacher VALUES('03' , '王五');

SELECT * FROM Teacher;

-- 创建课程表
CREATE TABLE Course(
Cid VARCHAR(10) PRIMARY KEY,-- 课程编号
Cname NVARCHAR(10),-- 课程名称
Tid  VARCHAR(10), -- 教师编号
FOREIGN KEY (Tid) REFERENCES Teacher(Tid)-- 外键约束
)DEFAULT CHARSET = utf8;
INSERT INTO Course VALUES('01' , '语文' , '02');
INSERT INTO Course VALUES('02' , '数学' , '01');
INSERT INTO Course VALUES('03' , '英语' , '03');
 SELECT * FROM Course;
 
-- 创建成绩表
CREATE TABLE SC(
Sid VARCHAR(10), -- 学生编号
Cid VARCHAR(10),-- 课程编号
score DECIMAL(18,1),-- 学生成绩
FOREIGN KEY (Sid) REFERENCES Student(Sid),-- 外键约束
FOREIGN KEY (Cid) REFERENCES Course(Cid) 
)DEFAULT CHARSET = utf8;
INSERT INTO SC VALUES('01' , '01' , 80);
INSERT INTO SC VALUES('01' , '02' , 90);
INSERT INTO SC VALUES('01' , '03' , 99);
INSERT INTO SC VALUES('02' , '01' , 70);
INSERT INTO SC VALUES('02' , '02' , 60);
INSERT INTO SC VALUES('02' , '03' , 80);
INSERT INTO SC VALUES('03' , '01' , 80);
INSERT INTO SC VALUES('03' , '02' , 80);
INSERT INTO SC VALUES('03' , '03' , 80);
INSERT INTO SC VALUES('04' , '01' , 50);
INSERT INTO SC VALUES('04' , '02' , 30);
INSERT INTO SC VALUES('04' , '03' , 20);
INSERT INTO SC VALUES('05' , '01' , 76);
INSERT INTO SC VALUES('05' , '02' , 87);
INSERT INTO SC VALUES('06' , '01' , 31);
INSERT INTO SC VALUES('06' , '03' , 34);
INSERT INTO SC VALUES('07' , '02' , 89);
INSERT INTO SC VALUES('07' , '03' , 98);
```





**练习题目**

1. 查询平均成绩大于等于60分的同学的学生编号和学生姓名和平均成绩

   ```sql
   SELECT 
       s.SId,
       s.Sname,
       AVG(sc.score) AS average_score
   FROM 
       Student s
   JOIN 
       SC sc ON s.SId = sc.SId
   GROUP BY 
       s.SId, s.Sname
   HAVING 
       AVG(sc.score) > 60;
   ```

   这个查询的步骤如下：

   1. 从学生表 `Student` 和成绩表 `SC` 中联接数据，基于学生编号 `SId`。
   2. 计算每个学生的平均分数。
   3. 通过 `GROUP BY` 将结果按学生编号和名字分组。
   4. 使用 `HAVING` 筛选出平均分数大于60的学生。

   >在这个查询中使用 `JOIN`（通常指 `INNER JOIN`）的原因是我们只关心那些有成绩记录的学生。如果使用 `LEFT JOIN`，则会包括没有成绩记录的学生，即使他们的平均分是 `NULL`，最终结果会不符合我们的要求（因为我们只想要平均分大于60的学生）。
   >
   >使用 `INNER JOIN` 只会返回在 `Student` 表和 `SC` 表中都有匹配记录的学生，因此确保只计算那些有成绩的学生的平均分，这样符合我们的需求。

2. 查询所有同学的学生编号、学生姓名、选课总数、所有课程的总成绩

   ```
   SELECT 
       s.SId,
       s.Sname,
       COUNT(sc.CId) AS course_count,
       SUM(sc.score) AS total_score
   FROM 
       Student s
   LEFT JOIN 
       SC sc ON s.SId = sc.SId
   GROUP BY 
       s.SId, s.Sname;
   ```

   这个查询的步骤如下：

   1. 从学生表 `Student` 和成绩表 `SC` 中联接数据，基于学生编号 `SId`。
   2. 使用 `COUNT(sc.CId)` 计算每个学生选课的总数。
   3. 使用 `SUM(sc.score)` 计算每个学生所有课程的总成绩。
   4. 通过 `GROUP BY` 将结果按学生编号和姓名分组，以确保每个学生只出现一次。

   这里使用 `LEFT JOIN` 是为了确保即使某些学生没有选课记录，他们的学生编号和姓名也能显示在结果中，选课总数和总成绩将显示为0。

3. 查询学过"张三"老师授课的同学的信息

   ```
   SELECT 
       s.SId,
       s.Sname,
       s.Sage,
       s.Ssex
   FROM 
       Student s
   JOIN 
       SC sc ON s.SId = sc.SId
   JOIN 
       Course c ON sc.CId = c.CId
   JOIN 
       Teacher t ON c.TId = t.TId
   WHERE 
       t.Tname = '张三';
   ```

   

4. 查询没学过"张三"老师授课的同学的信息

   ```sql
   SELECT 
       s.SId,
       s.Sname,
       s.Sage,
       s.Ssex
   FROM 
       Student s
   WHERE 
       s.SId NOT IN (
           SELECT 
               sc.SId
           FROM 
               SC sc
           JOIN 
               Course c ON sc.CId = c.CId
           JOIN 
               Teacher t ON c.TId = t.TId
           WHERE 
               t.Tname = '张三'
       );
   ```

   

5. 

