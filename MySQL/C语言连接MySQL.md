[TOC]

# C语言连接MySQL

## 一、库的安装

1. 在[官网](https://downloads.mysql.com/archives/c-c/)下载对应平台的接口包(Linux平台选择Linux-Generic通用版)；

2. 解压后，接口包内包含两个主要目录：

![img](https://typora-1307604235.cos.ap-nanjing.myqcloud.com/typora_img/202208171703250.jpg) 

我们需要使用include目录下的`"mysql.h"`头文件以及lib目录下的这些动态库或静态库：

![img](https://typora-1307604235.cos.ap-nanjing.myqcloud.com/typora_img/202208171703273.jpg) 

3. 可以选择将头文件和库添加至PATH下，如果害怕污染系统，则可以选择将它们添加至项目所在路径。

4. API官方文档可以[戳这里](https://dev.mysql.com/doc/c-api/8.0/en/c-api-basic-function-reference.html)。

5. 编译项目时记得链接动静态库。

 

## 二、基本API介绍

### 1、建立连接

```C
MYSQL* mysql_init(MYSQL *mysql);
MYSQL *mysql_real_connect(MYSQL *mysql, const char *host, const char *user, const char *passwd, 
       const char *db, unsigned int port, const char *unix_socket, unsigned long clientflag);
```

其中`mysql_init()`用来获取一个MySQL实例，`mysql_real_connect()`的参数介绍如下：

1. mysql：mysql_init()函数的返回值

2. host：主机IP

3. user：用户名

4. passwd：密码

5. db：要连接的数据库名

6. port：mysql服务器端口

7. unix_socket：一般设为空，表示不指定连接使用的套接字或管道；

8. clientflag：一般设为0。

9. RetVal：失败则返回NULL，成功则返回第一个参数。

### 2、设置字符集

```C
int mysql_set_character_set(MYSQL *mysql, const char *csname)
```

为当前连接设置字符集，如utf8，以**避免中文乱码**等问题。

### 3、写数据库

```C
int mysql_query(MYSQL *mysql, const char *stmt_str)
```

1. mysql：`mysql_init()`初始化的MySQL实例。

2. stmt_str：要执行的SQL语句。

3. RetVal：成功执行则返回0，否则返回非0值。

> 注：如果是增删改语句，则直接通过返回值查看执行结果，如果是查找语句，还需要进一步调用API获取数据。
>



### 4、读取结果

```C
MYSQL_RES *mysql_store_result(MYSQL *mysql)
```

将上一条查询语句的结果存储在`MYSQL_RES`类型的结构体中，以指针的形式返回。

```C
void mysql_free_result(MYSQL_RES *result)
```

获取查询结果后必须将`MYSQL_RES`结构体释放，避免内存泄漏。

```C
uint64_t mysql_num_rows(MYSQL_RES *result)
```

获取结果的行数。

```C
unsigned int mysql_num_fields(MYSQL_RES *result)
```

获取结果的列数。

```C
MYSQL_FIELD * mysql_fetch_fields(MYSQL_RES *result)
```

将列的相关信息存储在`MYSQL_FIELD`类型的结构体中返回，用户可以通过遍历fields[i].name获取列名。

```c
MYSQL_ROW mysql_fetch_row(MYSQL_RES *result)
```

获取下一行的记录。每一行记录都是`MYSQL_ROW`类型，通过col次遍历获取当前行的每一列的值。

**使用示例：**

```C++
MYSQL_FIELD *fields = mysql_fetch_fields(res);
for (int i = 0; i < col; ++i)
{
  cout << fields[i].name << "\t";
}

for (int i = 0; i < row; ++i)
{
  MYSQL_ROW record = mysql_fetch_row(res);
  for (int j = 0; j < col; ++j)
  {
     cout << (record[j] ? record[j] : "NULL") << "\t";
  }
  cout << endl;
}
```

> 注意：需要判断列值是否为空，避免对空指针解引用。



### 5、释放连接

```C
void mysql_close(MYSQL *mysql);
```

使用完毕后一定要关闭mysql实例！

