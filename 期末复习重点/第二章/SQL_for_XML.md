# 关系数据模型与SQL

## SQL for XML

| 模式名 | 功能 |
| --- | --- |
| RAW | 返回的行作为元素，列值作为元素的属性 |
| AUTO | 返回表名对应节点名称的元素，每列的属性作为元素的属性输出输出，可形成简单嵌套结构 |
| EXPLICIT | 通过SELECT语法定义输出XML结构 |
| PATH | 列名或列别名作为XPATH表达式来处理 |

SQL Server中的`FOR XML`子句允许开发者将查询结果转换成XML格式。这一特性非常实用，特别是在需要将关系型数据以XML格式交换或存储时。`FOR XML`子句支持四种模式：`AUTO`、`RAW`、`EXPLICIT`和`PATH`。每种模式都有其特点和适用场景。下面我们将详细介绍这四种模式的基本语法和用法，结合例子进行说明。
我们先设计一个简单的关系表，并基于这个表创建四种 SQL for XML 模式的语句示例： **RAW**, **AUTO**, **EXPLICIT**, 和 **PATH**。

### 示例关系表：`Employees`

| EmployeeID | Name         | Position   | Department    |
|------------|--------------|------------|---------------|
| 1          | Alice        | Engineer   | IT            |
| 2          | Bob          | Manager    | HR            |
| 3          | Charlie      | Analyst    | Finance       |

以下是 SQL for XML 的四种模式的例子：

### 1. **RAW 模式**

直接将每一行封装为一个 `<row>` 元素：

```sql
SELECT EmployeeID, Name, Position, Department
FROM Employees
FOR XML RAW;
```

**输出示例**：

```xml
<row EmployeeID="1" Name="Alice" Position="Engineer" Department="IT" />
<row EmployeeID="2" Name="Bob" Position="Manager" Department="HR" />
<row EmployeeID="3" Name="Charlie" Position="Analyst" Department="Finance" />
```

---

### 2. **AUTO 模式**

根据查询中指定的列和表的关系，自动生成嵌套结构：

```sql
SELECT EmployeeID, Name, Position, Department
FROM Employees
FOR XML AUTO;
```

**输出示例**：

```xml
<Employees EmployeeID="1" Name="Alice" Position="Engineer" Department="IT" />
<Employees EmployeeID="2" Name="Bob" Position="Manager" Department="HR" />
<Employees EmployeeID="3" Name="Charlie" Position="Analyst" Department="Finance" />
```

如果包含表的层级关系，例如部门和员工：

```sql
SELECT Department, EmployeeID, Name
FROM Employees
FOR XML AUTO, ELEMENTS;
```

**输出示例**（嵌套结构）：

```xml
<Employees>
  <Department>IT</Department>
  <EmployeeID>1</EmployeeID>
  <Name>Alice</Name>
</Employees>
<Employees>
  <Department>HR</Department>
  <EmployeeID>2</EmployeeID>
  <Name>Bob</Name>
</Employees>
```

---

### 3. **EXPLICIT 模式**

需要通过特定的列结构明确指定 XML 层级。规则：**元数据列**是SELECT查询必须先生成的满足规定要求的前两列，是Tag列和Parent列，作用是为结果提供层次信息：

- 第1列，列名固定为Tag，值是一个对应当前元素的标记号（整数类型）。 查询必须为从行集构造的每个元素提供标记号。
- 第2列，列名固定为Parent，值是父元素的标记号。Parent 列值为0或NULL表明相应的元素没有父级，该元素将作为顶级元素添加到 XML。

其他数据列名的指定方式**ElementName!TagNumber!AttributeName!Directive**，其中Directive 是可选的，提供有关 XML 构造的其他信息
**注意**：这种模式要求一个复杂的 `UNION ALL` 查询来构建 XML 的层级。

#### 示例

```sql
SELECT 1 AS Tag,
       NULL AS Parent,
       EmployeeID AS [Employees!1!EmployeeID],
       Name AS [Employees!1!Name],
       NULL AS [Department!2!Department]
FROM Employees
UNION ALL
SELECT 2 AS Tag,
       1 AS Parent,
       NULL AS [Employees!1!EmployeeID],
       NULL AS [Employees!1!Name],
       Department AS [Department!2!Department]
FROM Employees
ORDER BY [Employees!1!EmployeeID]
FOR XML EXPLICIT;
```

**输出示例**：

```xml
<Employees EmployeeID="1" Name="Alice">
  <Department>IT</Department>
</Employees>
<Employees EmployeeID="2" Name="Bob">
  <Department>HR</Department>
</Employees>
<Employees EmployeeID="3" Name="Charlie">
  <Department>Finance</Department>
</Employees>
```

---

### 4. **PATH 模式**

允许自定义 XML 的元素和属性的路径，并且可以更灵活地指定输出。

#### 示例 1：直接生成简单路径

```sql
SELECT EmployeeID AS "Employee/@ID",
       Name AS "Employee/Name",
       Position AS "Employee/Position"
FROM Employees
FOR XML PATH;
```

**输出示例**：

```xml
<Employee ID="1">
  <Name>Alice</Name>
  <Position>Engineer</Position>
</Employee>
<Employee ID="2">
  <Name>Bob</Name>
  <Position>Manager</Position>
</Employee>
<Employee ID="3">
  <Name>Charlie</Name>
  <Position>Analyst</Position>
</Employee>
```

#### 示例 2：嵌套路径

```sql
SELECT Department AS "Department",
       (SELECT EmployeeID AS "ID", Name AS "Name"
        FROM Employees E2
        WHERE E1.Department = E2.Department
        FOR XML PATH('Employee'), TYPE) AS "Employees"
FROM Employees E1
GROUP BY Department
FOR XML PATH('Department');
```

**输出示例**：

```xml
<Department>
  <Department>IT</Department>
  <Employees>
    <Employee>
      <ID>1</ID>
      <Name>Alice</Name>
    </Employee>
  </Employees>
</Department>
<Department>
  <Department>HR</Department>
  <Employees>
    <Employee>
      <ID>2</ID>
      <Name>Bob</Name>
    </Employee>
  </Employees>
</Department>
```

让我们通过一个具体的例子来说明 **RAW** 和 **AUTO** 模式的区别，尤其是 **AUTO 模式如何形成层次关系**。

### 假设有两张表：`Departments` 和 `Employees`

#### `Departments` 表

| DepartmentID | DepartmentName |
|--------------|----------------|
| 1            | IT             |
| 2            | HR             |

#### `Employees` 表

| EmployeeID | Name  | Position   | DepartmentID |
|------------|-------|------------|--------------|
| 1          | Alice | Engineer   | 1            |
| 2          | Bob   | Manager    | 2            |
| 3          | Eve   | Analyst    | 1            |

---

### 1. 使用 **RAW 模式**

RAW 模式会将每一行数据封装为 `<row>`，不形成嵌套关系：

#### SQL 查询

```sql
SELECT D.DepartmentName, E.EmployeeID, E.Name, E.Position
FROM Departments D
JOIN Employees E ON D.DepartmentID = E.DepartmentID
FOR XML RAW;
```

#### 输出 XML

```xml
<row DepartmentName="IT" EmployeeID="1" Name="Alice" Position="Engineer" />
<row DepartmentName="HR" EmployeeID="2" Name="Bob" Position="Manager" />
<row DepartmentName="IT" EmployeeID="3" Name="Eve" Position="Analyst" />
```

- 每一行数据以 `<row>` 包裹。
- **没有层次结构**：`Departments` 和 `Employees` 的信息在同一级别。

---

### 2. 使用 **AUTO 模式**

AUTO 模式根据表之间的关系自动生成层次结构，从左到右的表按嵌套顺序排列：

#### SQL 查询

```sql
SELECT D.DepartmentName, E.EmployeeID, E.Name, E.Position
FROM Departments D
JOIN Employees E ON D.DepartmentID = E.DepartmentID
FOR XML AUTO;
```

#### 输出 XML

```xml
<Departments DepartmentName="IT">
  <Employees EmployeeID="1" Name="Alice" Position="Engineer" />
  <Employees EmployeeID="3" Name="Eve" Position="Analyst" />
</Departments>
<Departments DepartmentName="HR">
  <Employees EmployeeID="2" Name="Bob" Position="Manager" />
</Departments>
```

- **层次结构**：`Departments` 成为父节点，`Employees` 成为子节点。
- 嵌套关系由表的连接顺序（从左到右）决定：`Departments → Employees`。

---

### 总结 RAW 与 AUTO 的区别

| 特性            | RAW 模式                             | AUTO 模式                                      |
|-----------------|-------------------------------------|-----------------------------------------------|
| 输出结构        | 每行数据一个 `<row>` 元素            | 自动根据表关系生成层次结构                   |
| 节点名称        | 固定为 `<row>`                      | 节点名称根据表名自动生成                     |
| 多表嵌套支持    | 不支持，所有数据在同一层级           | 支持，通过表连接顺序生成嵌套                 |
| 使用场景        | 简单的平面数据输出                   | 需要自动生成父子关系、层次结构的场景         |
