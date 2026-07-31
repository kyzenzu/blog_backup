---
title: C和C++中的数组是什么
date: 2024-8-23
tags: 
  - C
  - C/C++
---



[MySQL API 使用详解 | 爱编程的大丙 (subingwen.cn)](https://subingwen.cn/mysql/mysql-api/)



~~~C
/* 头文件 */
#include <mysql.h>

以上api对应的MySQL动态库
    Windows: libmysql.dll
    Linux: libmysqlclient.so
~~~

`MySQL`提供给`C`的库中所有的函数围绕一个结构体`MYSQL`，也就是说程序与`mysql`服务器就是靠这个对象维持连接的，下面给出定义：

~~~C
typedef struct st_mysql {
    NET         net;                /* 网络通信的相关信息 */
    gptr        connector_fd;       /* 连接器文件描述符 */
    char        *host,*user,*passwd,*unix_socket,*server_version,*host_info;
    char        *info, *db;
    struct charset_info_st *charset;/* 字符集信息 */
    MYSQL_FIELD *fields;
    MEM_ROOT    field_alloc;
    my_ulonglong affected_rows;     /* 受影响的行数 */
    my_ulonglong insert_id;         /* 自增ID */
    my_ulonglong extra_info;
    unsigned long thread_id;        /* 线程ID */
    unsigned long packet_length;
    unsigned int port;
    unsigned long client_flag, server_capabilities;
    unsigned int protocol_version;
    unsigned int field_count;
    unsigned int server_status;
    unsigned int server_language;
    unsigned int warning_count;
    struct st_mysql_options options;
    enum mysql_status status;
    my_bool free_me;                /* 是否需要释放 */
    my_bool reconnect;              /* 是否重新连接 */
    char scramble_buff[SCRAMBLE_LENGTH+1];
    my_bool unused1;
    void *unused2,*unused3,*unused4;
    LIST *stmts;                    /* SQL语句列表 */
    const struct st_mysql_methods *methods;
    void *thd;                      /* 线程数据 */
    my_bool *unbuffered_fetch_owner;
    char *info_buffer;
    void *extension;                /* 扩展字段 */
} MYSQL;
~~~

### 主要字段说明

- **`NET net`**: 处理与服务器之间的网络通信。包含套接字描述符、输入/输出缓冲区、超时设置等信息。
- **`char *host`**: 连接的服务器主机名或IP地址。
- **`char *user`**: 数据库用户名。
- **`char *passwd`**: 用户密码。
- **`char *unix_socket`**: Unix域套接字的路径（如果使用）。
- **`char *db`**: 当前使用的数据库名称。
- **`struct charset_info_st *charset`**: 指向当前使用字符集的信息结构。
- **`my_ulonglong affected_rows`**: 最近一次执行的SQL语句影响的行数（适用于 `INSERT`、`UPDATE`、`DELETE` 操作）。
- **`my_ulonglong insert_id`**: 最近一次插入操作产生的自增ID。
- **`unsigned long thread_id`**: 当前连接的线程ID。
- **`struct st_mysql_options options`**: 连接选项的相关结构体，保存了各种连接参数和配置。
- **`enum mysql_status status`**: 当前连接的状态，如连接中、已关闭等。
- **`my_bool reconnect`**: 表示是否在连接丢失时自动重连。
- **`LIST *stmts`**: 保存预处理的SQL语句。
- **`const struct st_mysql_methods *methods`**: 指向操作MySQL对象的函数指针表，提供了多态性的支持。
- **`void *thd`**: 线程数据指针，用于存储每个线程的上下文信息。
- **`void *extension`**: 供未来扩展使用的字段。

### 备注

- `NET` 结构体用于管理网络通信，如读取和写入数据包。
- `MYSQL_FIELD` 结构体用于保存列的信息，如列名、类型等。
- 该结构体使用了许多 `typedef` 类型和复杂的数据结构，如 `MEM_ROOT`、`LIST`、`my_ulonglong` 等，这些都是MySQL内部定义的类型。
- `methods` 字段使用了指向函数的指针表，这使得 `MYSQL` 结构体可以通过方法表执行不同的操作，实现了一定的面向对象的设计。

---

最开始调用的函数**`MYSQL* mysql_init(MYSQL* mysql)`**，就是为创建一个`MYSQL`对象在堆区开辟一块空间，并返回这块空间的地址。

~~~C
// 参数 mysql 可设置为 NULL
// 返回值: 该函数将分配、初始化、并返回新对象
// 			通过返回的这个对象去连接MySQL的服务器
MYSQL* mysql_init(MYSQL* mysql) ;
~~~

然后调用**`mysql_real_connect()`**函数，让上面创建的`MYSQL`对象与数据库连接。如果连接成功的话，函数返回的是刚刚传入的想要与数据库建立连接的`MYSQL`对象的地址，表示我们传入的`MYSQL`对象与数据库连接成功了。

~~~C
/*
返回值: 
    成功: 返回MYSQL*连接句柄, 对于成功的连接，返回值与第1个参数的值相同。返回值指向的内存和第一个参数指针指向的内存一样
    失败，返回NULL。
    句柄: 是windows中的一个概念, 句柄可以理解为一个实例(或者对象)
*/ 
MYSQL* mysql_real_connect(
    MYSQL *mysql,           // mysql_init() 函数的返回值
    const char *host,       // mysql服务器的主机地址, 写IP地址即可
                            // localhost, null -> 代表本地连接
    const char *user,       // 连接mysql服务器的用户名, 默认: root 
    const char *passwd,     // 连接mysql服务器用户对应的密码, root用户的密码
    const char *db,         // 要使用的数据库的名字
    unsigned int port,      /* 连接的mysql服务器监听的端口
                             如果==0, 使用mysql的默认端口3306, !=0, 使用指定的这个端口*/
    const char *unix_socket,  // 本地套接字, 不使用指定为 NULL
    unsigned long client_flag // 通常指定为0
);
~~~

成功建立连接后，我们就可以通过这个`MYSQL`对象用**`mysql_query()`**函数向`mysql`服务器发送一些`sql`语句让服务器去执行，函数执行完后，`sql`语句的执行结果集仍然存储在远端服务器中等待我们收取。

~~~C
// 执行一个sql语句, 添删查改的sql语句都可以
int mysql_query(MYSQL* mysql, const char* query);
参数:
    - mysql: 成功与某mysql服务器建立连接的MYSQL对象的地址
    - query: 一个可以执行的sql语句, 语句结尾不需要加 ';'
返回值: 
    - 如果查询成功，返回0。
    - 如果出现错误，返回非0值。 
~~~

---

服务器成功执行`sql`语句后，执行的结果集需要我们调用**`mysql_store_result()`**函数将结果集从远端接收到客户端内存中。这个函数会创建一个`MYSQL_RES`对象来存储收到的结果集，大概可以想成一个表格吧。

~~~C
// 参数是刚刚执行查询语句的 MYSQL对象地址
// MYSQL_RES 对应一块内存, 里边保存着这个查询之后得到的结果集
// 如何将行和列的数据从结果集中取出, 需要使用其他函数
// 返回值: 具有多个结果的MYSQL_RES结果集合。如果出现错误，返回NULL。 
MYSQL_RES* mysql_store_result(MYSQL* mysql);
~~~

下面的函数就是用来处理`MYSQL_RES`对象的函数

**`mysql_num_fields()`**

获取字段的个数

~~~C
// 从结果集中列的个数
// 参数: 调用 mysql_store_result() 得到的返回值
// 返回值: 结果集中的列数
unsigned int mysql_num_fields(MYSQL_RES *result)
~~~

**`mysql_fetch_fields()`**

获取所有字段的名称

~~~C
// 通过这个函数得到结果集中所有列(字段)的名字
// 参数: 调用 mysql_store_result() 得到的返回值
// 返回值: MYSQL_FIELD* 指向一个结构体数组的起始地址
// 通过查询官方文档, 返回是一个结构体的数组
MYSQL_FIELD* mysql_fetch_fields(MYSQL_RES* result);
~~~

`FIELD`表示一个字段，用**`MYSQL_FIELD`**结构体描述一个字段

~~~C
// mysql.h
// 结果集中的每一个列对应一个 MYSQL_FIELD
typedef struct st_mysql_field {
  char *name;                 /* 列名 或者说 字段的名字 */
  char *org_name;             /* Original column name, if an alias */
  char *table;                /* Table of column if column was a field */
  char *org_table;            /* Org table name, if table was an alias */
  char *db;                   /* Database for table */
  char *catalog;              /* Catalog for table */
  char *def;                  /* Default value (set by mysql_list_fields) */
  unsigned long length;       /* Width of column (create length) */
  unsigned long max_length;   /* Max width for selected set */
  unsigned int name_length;
  unsigned int org_name_length;                                                                                        
  unsigned int table_length;
  unsigned int org_table_length;
  unsigned int db_length;
  unsigned int catalog_length;
  unsigned int def_length;
  unsigned int flags;         /* Div flags */
  unsigned int decimals;      /* Number of decimals in field */
  unsigned int charsetnr;     /* Character set */
  enum enum_field_types type; /* Type of field. See mysql_com.h for types */
  void *extension;
} MYSQL_FIELD;
~~~

打印结果集的所有字段

~~~C
// 得到存储头信息的数组的地址
MYSQL_FIELD* fields = mysql_fetch_fields(res);
// 得到列数
int num = mysql_num_fields(res);
// 遍历得到每一列的列名
for(int i = 0; i < num; i++) {
    printf("当前列的名字: %s\n", fields[i].name);
}
~~~

**`mysql_fetch_row()`**

获取下一条记录

函数在调用时，最初指向的行是结果集的第一行之前。也就是说，当你第一次调用 mysql_fetch_row() 时，它会移动到结果集的第一行并返回该行的数据。

~~~C
typedef char** MYSQL_ROW;
/* 
        "value1"       "value2"
           ^               ^
           |               |
 ROW     char* value1   value2
           ^
           |
        MYSQL_ROW
*/

/* 函数在调用时，最初指向的行是结果集的第一行之前。也就是说，当你第一次调用 mysql_fetch_row() 时，它会移动到结果集的第一行并返回该行的数据。 */
MYSQL_ROW mysql_fetch_row(MYSQL_RES* result);
参数: 
    - result: 通过查询得到的结果集
返回值: 
    - 成功: 得到了当前记录字符串数组，每个字符串是对应字段的值
    - 失败: NULL, 说明数据已经读完了
~~~

**`mysql_fetch_lengths()`**

获取当前指向的记录所有字段值的长度

~~~C
/* 
返回结果集内当前行的列的长度:
    1. 如果打算复制字段值，使用该函数能避免调用strlen()。
    2. 如果结果集包含二进制数据，必须使用该函数来确定数据的大小，原因在于，对于包含'\0'字符的任何字段，strlen()将返回错误的结果。
*/
unsigned long *mysql_fetch_lengths(MYSQL_RES *result);
参数: 
    - result: 通过查询得到的结果集
返回值:
    - 无符号长整数的数组表示各列的大小。如果出现错误，返回NULL。
~~~

对记录的操作

~~~C
MYSQL_ROW row;
unsigned long* lengths;
unsigned int num_fields;

// 打印当前记录所有字段值的长度,按字节算
row = mysql_fetch_row(result);
if (row != NULL) {
    num_fields = mysql_num_fields(result);
    lengths = mysql_fetch_lengths(result);
    for(int i = 0; i < num_fields; i++) {
         printf("Column %u is %lu bytes in length.\n", i, lengths[i]);
    }
}

// 获取并打印每一行
int num_fields = mysql_num_fields(res);
while ((row = mysql_fetch_row(res))) {
     for(int i = 0; i < num_fields; i++) {
         printf("%s ", row[i] ? row[i] : "NULL");
     }
     printf("\n");
}
~~~

---

**资源回收/关闭连接**

~~~C
// 因为 MYSQL_RES 对象也是malloc出来的，所以不用的时候就释放掉
void mysql_free_result(MYSQL_RES* result);

// 关闭连接，释放在堆区的 MYSQL 对象的内存
void mysql_close(MYSQL* mysql);
~~~

**字符编码**

~~~C
// 查看某个MYSQL对象对结果集的字符编码
// 返回字符编码的字符串，比如说 "utf8"
const char* mysql_character_set_name(MYSQL* mysql)

// 设置MYSQL对象的字符编码
// 第一个参数指定MYSQL对象，第二个参数指定字符集编码类型，比如说 "utf8"
int mysql_set_character_set(MYSQL* mysql, char* csname)
~~~

**事务操作**

我们处理数据不是说添加一次就算完了，可能需要添加、删除、查找等一系列操作。这一系列操作序列在`mysql`中就称作**事务**，`mysql`中还提供对事务的回滚(**roll back**)，就是说事务当中的某次操作失败了，就可以回滚到事务处理前的状态，回到对数据库什么也没动的状态。

~~~
事务 ---- SELECT ---- INSERT ---- DELETE
    ^
roll back
~~~

~~~C
// mysql中默认会进行事务的提交
// 因为自动提交事务, 会对我们的操作造成影响
// 如果我们操作的步骤比较多, 集合的开始和结束需要用户自己去设置, 需要改为手动方式提交事务
// 这个函数就会发送关闭自动提交指令给mysql服务器
my_bool mysql_autocommit(MYSQL *mysql, my_bool mode) 
参数:
    如果模式为“1”，启用autocommit模式；如果模式为“0”，禁止autocommit模式。
返回值
    如果成功，返回0，如果出现错误，返回非0值。

// 事务提交
// 发送提交指令给mysql服务器
my_bool mysql_commit(MYSQL* mysql);
返回值: 成功: 0, 失败: 非0
    
// 数据回滚
// 发送数据回滚指令给mysql服务器
my_bool mysql_rollback(MYSQL* mysql) 
返回值: 成功: 0, 失败: 非0
~~~

**打印错误信息**

~~~C
// 返回错误信息
const char* mysql_error(MYSQL* mysql);
// 返回错误编号
unsigned int mysql_errno(MYSQL* mysql);
~~~

### 示例

~~~C
#include <stdio.h>
#include <mysql/mysql.h>

int main() {
    const char* host = "192.168.66.1";
    const char* user = "root";
    const char* passwd = "root";
    const char* db = "test";
    unsigned int port = 3306;
    const char* unix_socket = NULL;
    unsigned long client_flag = 0;

    MYSQL* mysql = mysql_init(NULL);
    mysql = mysql_real_connect(mysql, host, user, passwd, db, port, unix_socket, client_flag);
    if (mysql != NULL) {
        printf("连接成功\n");
    } else {
        perror("连接失败");
        return -1;
    }
    if (mysql_query(mysql, "SELECT * FROM user") == 0) {
        printf("查询成功\n");
        MYSQL_RES* result = mysql_store_result(mysql);
        int num = mysql_num_fields(result);
        printf("字段数为: %d\n", num);
        MYSQL_FIELD* fields = mysql_fetch_fields(result);
        MYSQL_ROW row;
        while ((row = mysql_fetch_row(result))) {
            for (int i = 0; i < num; i++)
            printf("%s: %s\n", fields[i].name, row[i] ? row[i] : "NULL");
        }
        
    } else {
        perror("查询失败");
    }
    mysql_close(mysql);
    return 0;
}
~~~



---

## 封装

