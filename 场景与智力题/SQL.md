# SQL



## 时间区间是否有交集

**1.给定时间区间（begin，end），数据库字段startTime与endTime，现在要判断它们之间是否有交集。**

```shell
SELECT * FROM xxx
WHERE NOT ((endTime < begin) OR (startTime > end))
```





## 课程学生系列

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





## 部门工资最高的员工

[184. 部门工资最高的员工](https://leetcode.cn/problems/department-highest-salary/)



### 题目描述

表： `Employee`

```
+--------------+---------+
| 列名          | 类型    |
+--------------+---------+
| id           | int     |
| name         | varchar |
| salary       | int     |
| departmentId | int     |
+--------------+---------+
在 SQL 中，id是此表的主键。
departmentId 是 Department 表中 id 的外键（在 Pandas 中称为 join key）。
此表的每一行都表示员工的 id、姓名和工资。它还包含他们所在部门的 id。
```

 

表： `Department`

```
+-------------+---------+
| 列名         | 类型    |
+-------------+---------+
| id          | int     |
| name        | varchar |
+-------------+---------+
在 SQL 中，id 是此表的主键列。
此表的每一行都表示一个部门的 id 及其名称。
```

 

查找出每个部门中薪资最高的员工。按 **任意顺序** 返回结果表。

查询结果格式如下例所示。

**示例 1:**

```
输入：
Employee 表:
+----+-------+--------+--------------+
| id | name  | salary | departmentId |
+----+-------+--------+--------------+
| 1  | Joe   | 70000  | 1            |
| 2  | Jim   | 90000  | 1            |
| 3  | Henry | 80000  | 2            |
| 4  | Sam   | 60000  | 2            |
| 5  | Max   | 90000  | 1            |
+----+-------+--------+--------------+
Department 表:
+----+-------+
| id | name  |
+----+-------+
| 1  | IT    |
| 2  | Sales |
+----+-------+
输出：
+------------+----------+--------+
| Department | Employee | Salary |
+------------+----------+--------+
| IT         | Jim      | 90000  |
| Sales      | Henry    | 80000  |
| IT         | Max      | 90000  |
+------------+----------+--------+
解释：Max 和 Jim 在 IT 部门的工资都是最高的，Henry 在销售部的工资最高。
```



### SQL

```sql
SELECT
    Department.name AS 'Department',
    Employee.name AS 'Employee',
    Salary
FROM
    Employee
        JOIN
    Department ON Employee.DepartmentId = Department.Id
WHERE
    (Employee.DepartmentId , Salary) IN
    (   SELECT
            DepartmentId, MAX(Salary)
        FROM
            Employee
        GROUP BY DepartmentId
    )
;
```





## 部门工资前三高的所有员工

### 题目描述

>表: `Employee`与表: `Department`和上方一样

公司的主管们感兴趣的是公司每个部门中谁赚的钱最多。一个部门的 **高收入者** 是指一个员工的工资在该部门的 **不同** 工资中 **排名前三** 。

编写解决方案，找出每个部门中 **收入高的员工** 。

以 **任意顺序** 返回结果表。

返回结果格式如下所示。

 

**示例 1:**

```
输入: 
Employee 表:
+----+-------+--------+--------------+
| id | name  | salary | departmentId |
+----+-------+--------+--------------+
| 1  | Joe   | 85000  | 1            |
| 2  | Henry | 80000  | 2            |
| 3  | Sam   | 60000  | 2            |
| 4  | Max   | 90000  | 1            |
| 5  | Janet | 69000  | 1            |
| 6  | Randy | 85000  | 1            |
| 7  | Will  | 70000  | 1            |
+----+-------+--------+--------------+
Department  表:
+----+-------+
| id | name  |
+----+-------+
| 1  | IT    |
| 2  | Sales |
+----+-------+
输出: 
+------------+----------+--------+
| Department | Employee | Salary |
+------------+----------+--------+
| IT         | Max      | 90000  |
| IT         | Joe      | 85000  |
| IT         | Randy    | 85000  |
| IT         | Will     | 70000  |
| Sales      | Henry    | 80000  |
| Sales      | Sam      | 60000  |
+------------+----------+--------+
解释:
在IT部门:
- Max的工资最高
- 兰迪和乔都赚取第二高的独特的薪水
- 威尔的薪水是第三高的

在销售部:
- 亨利的工资最高
- 山姆的薪水第二高
- 没有第三高的工资，因为只有两名员工
```



### SQL

```sql
SELECT
    d.Name AS 'Department', e1.Name AS 'Employee', e1.Salary
FROM
    Employee e1
        JOIN
    Department d ON e1.DepartmentId = d.Id
WHERE
    3 > (SELECT
            COUNT(DISTINCT e2.Salary)
        FROM
            Employee e2
        WHERE
            e2.Salary > e1.Salary
                AND e1.DepartmentId = e2.DepartmentId
        )
;
```















## 查询每个用户的点击总数、不同位置的点击次数以及不同订单数

我们可以假设有两张表：

1. `click_log` 表：
   - `user_id`: 用户ID
   - `click_count`: 点击数
   - `position`: 位置
   - `clickId`: 点击记录ID
2. `order_log` 表：
   - `clickId`: 点击记录ID（外键，关联到 `click_log` 表）
   - `order_id`: 订单ID

接下来，通过 `click_log` 和 `order_log` 的关联，可以查询每个用户的点击总数、不同位置的点击次数以及不同订单数。SQL 查询可以写成如下：

```sql
SELECT 
    c.user_id,
    SUM(c.click_count) AS total_clicks,
    COUNT(DISTINCT o.order_id) AS distinct_orders,
    COUNT(DISTINCT c.position) AS distinct_positions
FROM 
    click_log c
LEFT JOIN 
    order_log o ON c.clickId = o.clickId
GROUP BY 
    c.user_id;
```

解释：

- `SUM(c.click_count) AS total_clicks`：计算每个用户的总点击数。
- `COUNT(DISTINCT o.order_id) AS distinct_orders`：计算每个用户的不同订单数。
- `COUNT(DISTINCT c.position) AS distinct_positions`：计算每个用户点击时的不同位置数量。

通过这种方式，我们可以将两张表的数据进行关联，并计算用户的点击数、订单数和位置数。





## 第二高的薪水



### 题目描述

`Employee` 表：

```
+-------------+------+
| Column Name | Type |
+-------------+------+
| id          | int  |
| salary      | int  |
+-------------+------+
id 是这个表的主键。
表的每一行包含员工的工资信息。
```

 

查询并返回 `Employee` 表中第二高的 **不同** 薪水 。如果不存在第二高的薪水，查询应该返回 `null(Pandas 则返回 None)` 。

查询结果如下例所示。

 

**示例 1：**

```
输入：
Employee 表：
+----+--------+
| id | salary |
+----+--------+
| 1  | 100    |
| 2  | 200    |
| 3  | 300    |
+----+--------+
输出：
+---------------------+
| SecondHighestSalary |
+---------------------+
| 200                 |
+---------------------+
```





### SQL

```sql
SELECT
    (SELECT DISTINCT
            Salary
        FROM
            Employee
        ORDER BY Salary DESC
        LIMIT 1 OFFSET 1) AS SecondHighestSalary
;
```





