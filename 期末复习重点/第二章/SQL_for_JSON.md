# SQL for JSON

MySQL 提供了强大的 JSON 支持，使开发者能够高效地存储和查询结构化数据。下面，我们通过实例来介绍 MySQL 的 JSON 基本用法，包括如何存储、查询、更新和操作 JSON 数据。

---

## 1. 创建一个带有 JSON 列的表

我们可以将 JSON 数据类型用于表中的字段，例如：

```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50),
    profile JSON
);
```

**解释**：  

- `profile` 列的类型为 `JSON`，专门用于存储 JSON 数据。
- `name` 列用于存储用户的名称。

---

## 2. 插入 JSON 数据

可以直接插入符合 JSON 格式的数据：

```sql
INSERT INTO users (name, profile) 
VALUES 
('Alice', '{"age": 25, "city": "New York", "skills": ["MySQL", "Python"]}'),
('Bob', '{"age": 30, "city": "San Francisco", "skills": ["Java", "C++"]}');
```

**注意**：插入的数据必须是有效的 JSON 格式，否则 MySQL 会报错。

---

## 3. 查询 JSON 数据

MySQL 提供了很多内置函数来操作 JSON 数据。

### (1) 提取 JSON 数据 (`->` 和 `->>`)

- `->` 用于获取 JSON 对象中的子对象。
- `->>` 用于获取 JSON 数据的字符串值。

```sql
-- 获取 Alice 的城市
SELECT name, profile->>'$.city' AS city 
FROM users 
WHERE name = 'Alice';
```

**结果**：

```
+-------+-----------+
| name  | city      |
+-------+-----------+
| Alice | New York  |
+-------+-----------+
```

---

### (2) 查询嵌套属性

JSON 路径支持深层嵌套查询：

```sql
-- 查询技能中的第一个值
SELECT name, profile->'$.skills[0]' AS first_skill 
FROM users;
```

**结果**：

```
+-------+-------------+
| name  | first_skill |
+-------+-------------+
| Alice | "MySQL"     |
| Bob   | "Java"      |
+-------+-------------+
```

也可以使用`JSON_EXTRACT`获取指定的数据

```sql
例1：mysql> SELECT JSON_EXTRACT('[10, 20, [30, 40]]', '$[1]');
结果： 20 
例2：mysql> SELECT JSON_EXTRACT('[10, 20, [30, 40]]', '$[1]', '$[0]'); 
结果： [20, 10] --多个值拼接为数组
例3：mysql> SELECT JSON_EXTRACT('[10, 20, [30, 40]]', '$[2][*]'); 
结果： [30, 40]
```

JSON不同于字符串，如果直接和整个JSON字段比较，不会查询到结果

```sql
SELECT * FROM muscleape WHERE category = '{"id": 1,"name": "muscleape"}';----结果查询不到数据。
```

这时候需要利用`CAST`把字符串转化成为`JSON`

```sql
SELECT * FROM muscleape WHERE category = CAST('{"id": 1,"name": "muscleape"}' AS JSON);
这样就可以正常找到结果了
```

还可以利用`JSON_CONTAINS`查找内部值

```sql
SELECT * FROM muscleape 
WHERE JSON_CONTAINS(category,  '1',  '$.id');-- 可以查询到数据
注意第二个参数只能是字符串
```

---

## 4. 更新 JSON 数据

可以使用 `JSON_SET`、`JSON_REMOVE` 等函数更新 JSON 数据。

### (1) 更新 JSON 中的值

```sql
-- 将 Alice 的城市改为 "Los Angeles"
UPDATE users 
SET profile = JSON_SET(profile, '$.city', 'Los Angeles') 
WHERE name = 'Alice';

JSON_SET( )函数——插入新值，并覆盖已存在的值
UPDATE muscleape SET category = JSON_SET(category,
'$.host’,  'localhost’,   '$.url',  'www.muscleape.com') 
WHERE id = 1;

JSON_REPLACE( )函数——只替换已存在的值
UPDATE muscleape SET category = JSON_REPLACE(category,
'$.host',  '127.0.0.1',   '$.address',  'shandong') 
WHERE id = 1;

```

### (2) 删除 JSON 中的属性

```sql
-- 删除 Bob 的技能信息
UPDATE users 
SET profile = JSON_REMOVE(profile, '$.skills') 
WHERE name = 'Bob';
```

### (3) 插入值

```sql
UPDATE muscleape SET category = JSON_INSERT(category,
    '$.name', 'muscleape_new',  '$.url', 'muscleape.com')  WHERE id = 1;
-----当JSON数据中已经存在name属性而没有url属性时，name值不会被修改，而url的值被添加进去。

```

---

## 5. 查询包含特定 JSON 数据的行

使用 `JSON_CONTAINS` 函数检查 JSON 中是否包含特定值。

```sql
-- 查询拥有 "Python" 技能的用户
SELECT name 
FROM users 
WHERE JSON_CONTAINS(profile->'$.skills', '"Python"');
```

**结果**：

```
+-------+
| name  |
+-------+
| Alice |
+-------+
```

---

## 7. 聚合和复杂查询

MySQL 的 JSON 函数可以与其他 SQL 语法结合，进行复杂的数据分析。

```sql
-- 统计用户的年龄总和
SELECT SUM(profile->>'$.age') AS total_age 
FROM users;
```

在 MySQL 中，`JSON_ARRAYAGG` 和 `JSON_OBJECTAGG` 是两个用于聚合数据的函数，它们的输出形式分别为 JSON 数组和 JSON 对象。这两个函数常用于将关系型数据转化为 JSON 格式，便于前端处理或其他用途。

---

### 1. **`JSON_ARRAYAGG`**：聚合为 JSON 数组

#### 功能  

将查询结果中的每一行的数据聚合成一个 JSON 数组。

#### 语法

```sql
JSON_ARRAYAGG(expression)
```

- **`expression`**: 要放入数组的值，可以是列、表达式或函数的返回值。

---

#### 示例 1：简单使用

假设有一个表 `employees`：

```sql
CREATE TABLE employees (
    id INT,
    name VARCHAR(50),
    department VARCHAR(50)
);

INSERT INTO employees VALUES 
(1, 'Alice', 'HR'),
(2, 'Bob', 'Engineering'),
(3, 'Charlie', 'HR');
```

我们希望将所有员工的姓名聚合成一个 JSON 数组：

```sql
SELECT JSON_ARRAYAGG(name) AS employee_names 
FROM employees;
```

**结果**：

```
+----------------------+
| employee_names       |
+----------------------+
| ["Alice", "Bob", "Charlie"] |
+----------------------+
```

---

#### 示例 2：按部门分组

将员工的姓名按部门聚合：

```sql
SELECT 
    department,
    JSON_ARRAYAGG(name) AS employee_names 
FROM employees
GROUP BY department;
```

**结果**：

```
+-------------+--------------------+
| department  | employee_names     |
+-------------+--------------------+
| HR          | ["Alice", "Charlie"] |
| Engineering | ["Bob"]            |
+-------------+--------------------+
```

---

### 2. **`JSON_OBJECTAGG`**：聚合为 JSON 对象

#### 功能  

将查询结果中的每一行的两列数据聚合为键值对形式的 JSON 对象。

#### 语法

```sql
JSON_OBJECTAGG(key_expression, value_expression)
```

- **`key_expression`**: 对象的键。
- **`value_expression`**: 对象的值。

---

#### 示例 1：简单使用

继续使用 `employees` 表，我们希望创建一个以员工 ID 为键、姓名为值的 JSON 对象：

```sql
SELECT JSON_OBJECTAGG(id, name) AS employee_map 
FROM employees;
```

**结果**：

```
+-------------------------------+
| employee_map                 |
+-------------------------------+
| {"1": "Alice", "2": "Bob", "3": "Charlie"} |
+-------------------------------+
```

---

#### 示例 2：按部门分组

将每个部门的员工 ID 和姓名聚合为 JSON 对象：

```sql
SELECT 
    department,
    JSON_OBJECTAGG(id, name) AS employee_details 
FROM employees
GROUP BY department;
```

**结果**：

```
+-------------+-----------------------------------+
| department  | employee_details                 |
+-------------+-----------------------------------+
| HR          | {"1": "Alice", "3": "Charlie"}   |
| Engineering | {"2": "Bob"}                     |
+-------------+-----------------------------------+
```

---

### 3. **比较与应用场景**

| **函数**       | **输出格式**                 | **适用场景**                                               |
|-----------------|-----------------------------|-----------------------------------------------------------|
| `JSON_ARRAYAGG` | JSON 数组                   | 将多行数据组合成一个 JSON 数组，适用于无键值对的简单场景。 |
| `JSON_OBJECTAGG`| JSON 对象（键值对）         | 生成键值对结构的 JSON 数据，例如主键-值映射或配置表等。     |

---

### 4. **结合使用的场景**

假设我们有一个包含员工及其技能的表 `employee_skills`：

```sql
CREATE TABLE employee_skills (
    employee_id INT,
    skill VARCHAR(50)
);

INSERT INTO employee_skills VALUES 
(1, 'Management'),
(1, 'Recruitment'),
(2, 'Java'),
(2, 'MySQL'),
(3, 'Communication');
```

我们希望为每个员工生成一个以员工 ID 为键、技能列表为值的 JSON 对象：

```sql
SELECT 
    JSON_OBJECTAGG(employee_id, skills) AS employee_skills_map
FROM (
    SELECT 
        employee_id,
        JSON_ARRAYAGG(skill) AS skills
    FROM employee_skills
    GROUP BY employee_id
) AS aggregated_skills;
```

**结果**：

```
+-----------------------------------------+
| employee_skills_map                     |
+-----------------------------------------+
| {"1": ["Management", "Recruitment"],    |
|  "2": ["Java", "MySQL"],                |
|  "3": ["Communication"]}                |
+-----------------------------------------+
```

---

### `JSON_TABLE`的使用

**exp1**:将数组中的每个对象元素转为一个关系表中的一行，表中的列对应了每个对象中的成员，其中for ordinality定义了自增长计数器列。

```sql
SELECT  * FROM  JSON_TABLE(
    '[{"x": 10, "y": 11}, {"x": 20, "y": 21}]','$[*]'   //表示数组中的所有元素
    COLUMNS (   id FOR ORDINALITY,
                x INT PATH '$.x',
                y INT PATH '$.y'  
            )    
        ) AS t;
```

**exp2**:提取数组中第2行，设置值为空时的缺省值

```sql
SELECT  * FROM  JSON_TABLE(  
    '[{"x": 10, "y": 11}, {"x": 20}]','$[1]'   //取数组中的第2个元素
            COLUMNS (  id FOR ORDINALITY,
            x INT PATH '$.x',
            y INT PATH '$.y'  DEFAULT '100' ON EMPTY 
            )    
        ) AS t;
```

**exp3**:拉平内嵌的数组

```sql
SELECT *  FROM  JSON_TABLE(   
    '[{"x":10,"y":[11, 12]},{"x":20,"y":[21, 22]}]',
        '$[*]'  COLUMNS (  
            x INT PATH '$.x',
            NESTED PATH '$.y[*]' COLUMNS (y INT PATH '$') 
         //展开 y 对应的数组，并将 y 数组中的每个元素放入名称为 y 的列中  
         )
) AS t;

res:
x     y
10    11
10    12
20    21
20    22
```

**exp4**:拉平内嵌的对象

```sql
SELECT  *   FROM   JSON_TABLE(    
    '[{"x":10,"y":{"a":11,"b":12}},{"x":20,"y":{"a":21,"b":22}}]',
        '$[*]'  COLUMNS (   x INT PATH '$.x',
            NESTED PATH '$.y' COLUMNS (   
                ya INT PATH '$.a',  
                yb INT PATH '$.b'  
                )          
            )
      ) AS t; 

res:
x   ya  yb
10  11  12
20  21  22
```

**exp5**:实际查询时应用

```sql
SELECT c1, c2, t.at, tt.bt, tt.ct, JSON_EXTRACT(c3, '$.*') 
FROM t1 AS m 
JOIN 
JSON_TABLE(   m.c3, 
  '$.*'  COLUMNS(
    at VARCHAR(10) PATH '$.a' DEFAULT '1' ON EMPTY, 
    bt VARCHAR(10) PATH '$.b' DEFAULT '2' ON EMPTY, 
    ct VARCHAR(10) PATH '$.c' DEFAULT '3' ON EMPTY
  )
) AS tt
ON m.c1 > tt.at;

```

**例题**：设MySQL数据库中有包含json数据列的关系R1（sno int, cno int , eval json）记录学生学习具体课程的学习记录的学号、课号、教师评语，完成下列小题：
（1）若要插入记录：1号学生学了3号课程，评语列的json对象由属性名“evaluation”和体教师评语的记录数组作为“evaluation”属性的值构成，该数组包含三条记录，分别是`{ “stage”：1, “result”:”ok”}、{ “stage”：2, “result”:”good”}、{ “stage”：3, “result”:” excellent”}`，描述该生在该课程的3个阶段的教师评语。请写出MySQL中插入该记录的面向json拓展的SQL语句。

```sql
INSERT INTO R1 (sno, cno, eval) VALUES
 (1, 3, '{"evaluation":  [{"stage":1, "result":"ok"},
 {"stage":2, "result":"good"}, {"stage":3, "result":" excellent"}]  }');
```

（2）如果要在关系R1中查出上述第（1）小题插入的记录的前两个stage的信息，要求结果属性列表为“学号、课号、阶段号stage值及该阶段的教师评语result值”，并且不同的阶段输出在不同的行，请写出MySQL中插入该记录的面向json拓展的SQL语句。

```sql
因为记录只有一条，那么需要用where筛选时需要进行导出
SELECT sno，cno, tt.stage, tt.result FROM R1, 
JSON_TABLE (R1.eval,  '$.evauation[*]'   COLUMNS(
   stage int PATH '$.stage',
   result VARCHAR(20) PATH '$.result')
) AS tt  
WHERE tt.stage<3;

```

**注释**：
SQL查询中，`R1` 表与 `JSON_TABLE` 函数生成的虚拟表 `tt` 之间看似没有进行显式的连接操作，但实际上，`JSON_TABLE` 函数本身就是一个连接点。`JSON_TABLE` 函数用于解析 JSON 格式的数据，并将解析出来的数据转换为关系表的形式，以便与 SQL 语句中的其他表进行联合查询。在这个过程中，`JSON_TABLE` 会自动将 `R1` 表中的每一行与 JSON 数据中的每一个元素进行关联，从而实现了隐式的连接。

让我们更详细地看看这个查询的各个部分：

1. **`SELECT sno, cno, tt.stage, tt.result FROM R1,`**:
   - 这部分定义了查询的字段来源。`sno` 和 `cno` 来自 `R1` 表，而 `tt.stage` 和 `tt.result` 来自 `JSON_TABLE` 生成的虚拟表 `tt`。

2. **`JSON_TABLE (R1.eval, '$.evauation[*]' COLUMNS(stage int PATH '$.stage', result VARCHAR(20) PATH '$.result')) AS tt`**:
   - `JSON_TABLE` 函数接收两个参数：`R1.eval` 和一个 JSON 路径表达式 `'$.evauation[*]'`。
   - `R1.eval` 是 `R1` 表中的一个 JSON 字段，包含了需要解析的 JSON 数据。
   - `'$.*evaluation[*]'` 是一个 JSON 路径表达式，用于遍历 `eval` 字段中的 `evaluation` 数组中的每一个元素。
   - `COLUMNS` 子句定义了从每个 JSON 元素中提取的列及其路径。`stage` 和 `result` 分别对应 JSON 元素中的 `stage` 和 `result` 字段。
   - `AS tt` 将 `JSON_TABLE` 函数生成的虚拟表命名为 `tt`。

3. **`WHERE tt.stage < 3`**:
   - 这个条件用于过滤 `tt` 表中的行，只保留 `stage` 值小于 3 的记录。

### 隐式连接的实现

尽管在 `FROM` 子句中没有显式地使用 `JOIN` 关键字，`JSON_TABLE` 函数实际上已经将 `R1` 表中的每一行与 `eval` 字段中的 JSON 数组中的每一个元素进行了关联。具体来说，`JSON_TABLE` 会为 `R1` 表中的每一行生成一个或多个虚拟行，每个虚拟行对应 `eval` 字段中的一个 JSON 元素。这些虚拟行与 `R1` 表中的原始行一起构成了最终的查询结果。

### 示例解释

假设 `R1` 表的数据如下：

| sno | cno | eval                    |
|-----|-----|-------------------------|
| 1   | 101 | {"evaluation": [{"stage": 1, "result": "pass"}, {"stage": 2, "result": "fail"}]} |
| 2   | 102 | {"evaluation": [{"stage": 1, "result": "pass"}, {"stage": 3, "result": "pass"}]} |

执行上述查询后，`JSON_TABLE` 会生成如下的虚拟表 `tt`：

| sno | cno | stage | result |
|-----|-----|-------|--------|
| 1   | 101 | 1     | pass   |
| 1   | 101 | 2     | fail   |
| 2   | 102 | 1     | pass   |
| 2   | 102 | 3     | pass   |

然后，`WHERE tt.stage < 3` 条件会过滤掉 `stage` 值为 3 的行，最终结果为：

| sno | cno | stage | result |
|-----|-----|-------|--------|
| 1   | 101 | 1     | pass   |
| 1   | 101 | 2     | fail   |
| 2   | 102 | 1     | pass   |

### 总结

虽然在 `FROM` 子句中没有显式的 `JOIN` 关键字，但 `JSON_TABLE` 函数实际上已经实现了 `R1` 表与 JSON 数据的隐式连接。这种连接方式使得可以从 JSON 格式的数据中提取结构化信息，并与表中的其他字段一起进行查询和过滤。
