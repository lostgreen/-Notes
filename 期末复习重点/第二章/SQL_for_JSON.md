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

---

## 4. 更新 JSON 数据

可以使用 `JSON_SET`、`JSON_REMOVE` 等函数更新 JSON 数据。

### (1) 更新 JSON 中的值

```sql
-- 将 Alice 的城市改为 "Los Angeles"
UPDATE users 
SET profile = JSON_SET(profile, '$.city', 'Los Angeles') 
WHERE name = 'Alice';
```

### (2) 删除 JSON 中的属性

```sql
-- 删除 Bob 的技能信息
UPDATE users 
SET profile = JSON_REMOVE(profile, '$.skills') 
WHERE name = 'Bob';
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
