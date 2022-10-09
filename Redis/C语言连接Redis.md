[TOC]

# Hiredis

## 一、简介

> Hiredis库封装了Redis的C语言接口，专门用来操作Redis数据库。
>
> [Hiredis库下载](https://github.com/redis/hiredis)

库下载完成之后，利用`make`和`make install`将库安装到服务器，之后就可以通过`-lhiredis`选项将库链接到可执行文件了。

> 注：如果报出"...Not Found..."的错误，可以在root下编辑`/etc/ld.so.conf`，加上一行`/usr/local/lib`，再使用执行`ldconfig`命令刷新一下即可。



## 二、API介绍

### 1、建立连接

```C
redisContext *redisConnect(const char *ip, int port);
```

两个参数分别对应服务器的ip与端口。

**示例**

```C++
redisContext* redis = redisConnect("127.0.0.1", 6379);
if (redis->err || redis == nullptr)
{
    cout << "连接失败" << endl;
    return -1;
}
```

> 注：如果是云服务器，需要在防火墙开放端口。

------

### 2、写数据库

```C
void *redisCommand(redisContext *c, const char *format, ...);
void freeReplyObject(void *reply);
```

c是`redisConnect`返回的Redis实例，后面的可变参数是需要执行的命令，使用起来类似于`printf()`函数。

> 注：该函数的返回值可以被`redisReply*`指针接收，用来获取命令执行结果，在使用结束后必须调用`freeReplyObject()`进行释放！

**示例**

```C++
char value[] = "hello redis";
redisReply *reply = (redisReply *)redisCommand(redis, "ping");
redisReply *reply = (redisReply *)redisCommand(redis, "set key %s", value);
```

#### redisReply结构体

```c
typedef struct redisReply
{
    int type;                    // 返回的信息类型
    long long integer;           // 当type为REDIS_REPLY_INTEGER时有效
    double dval;                 // 当type为REDIS_REPLY_DOUBLE时有效
    size_t len;                  // 字符串长度
    char *str;                   // 当字段为REDIS_REPLY_ERROR, REDIS_REPLY_STRING
                                 // REDIS_REPLY_VERB, REDIS_REPLY_DOUBLE (in additional to dval),
                                 // 和 REDIS_REPLY_BIGNUM时有效
    char vtype[4];               // 返回值为REDIS_REPLY_VERB时有效
    size_t elements;             // element数组的元素个数
    struct redisReply **element; // 当type为REDIS_REPLY_ARRAY时有效
} redisReply;
```

**其中，信息类型有：**

```C
#define REDIS_REPLY_STRING 1
#define REDIS_REPLY_ARRAY 2
#define REDIS_REPLY_INTEGER 3
#define REDIS_REPLY_NIL 4
#define REDIS_REPLY_STATUS 5
#define REDIS_REPLY_ERROR 6
#define REDIS_REPLY_DOUBLE 7
#define REDIS_REPLY_BOOL 8
#define REDIS_REPLY_MAP 9
#define REDIS_REPLY_SET 10
#define REDIS_REPLY_ATTR 11
#define REDIS_REPLY_PUSH 12
#define REDIS_REPLY_BIGNUM 13
#define REDIS_REPLY_VERB 14
```

### 3、释放连接

```c
void redisFree(redisContext *c);
```

释放`redisConnect()`创建的redis实例。



## 三、应用示例

```C
int main()
{
    redisContext *redis = redisConnect("101.35.158.44", 6379);
    if (redis->err || redis == nullptr)
    {
        cout << "连接失败" << endl;
        return -1;
    }
    else
    {
        cout << "连接成功" << endl;
    }

    string key = "testkey";
    string value = "hello redis!";
    // 密码验证
    redisCommand(redis, "auth redis123456");
    // 设置键值对
    redisReply *reply = (redisReply *)redisCommand(redis, "set %s %s", key.c_str(), value.c_str());
    // 获取键值对
    reply = (redisReply *)redisCommand(redis, "get %s", key.c_str());
    cout << reply->str << endl;
    // 利用 keys * 获取所有键值对
    reply = (redisReply *)redisCommand(redis, "keys *");
    size_t n = reply->elements;
    for (int i = 0; i < n; ++i)
    {
        cout << reply->element[i]->str << endl;
    }

    freeReplyObject(reply);
    redisFree(redis);
    return 0;
}
```

