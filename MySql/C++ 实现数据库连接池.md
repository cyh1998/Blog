## 概 述
本文介绍使用C++实现数据库连接池。  
连接池的基本思想就是在初始化时，将数据库连接对象保存在内存中，当用户需要访问数据库时，并非新建一个连接，而是从连接池中取出一个空闲的连接对象；使用完毕后，也并非直接断开连接，而是重新放到连接池中。以达到避免频繁连接断开数据库，提高数据库的处理效率。

## 实 现
根据连接池的基本思想，我们设计一个数据库连接池，如下：  
- 单例模式，保证进程唯一
- 使用queue来管理数据库连接对象(静态大小)
- 互斥锁实现线程安全

基于queue实现一个静态大小的简单数据库连接池，示例代码如下：
```
//SqlConnPool.h
#include <string>
#include <queue>
#include <mutex>
#include <mysql/mysql.h>

class SqlConnPool {
public:
    static SqlConnPool *GetInstance(); //单例

    void Init(const std::string& host, int port,
              const std::string& user, const std::string& pwd,
              const std::string& db_name, int conn_size);
    void ClosePool();

    MYSQL *GetConnObj(); //获取连接对象
    void FreeConnObj(MYSQL *conn); //释放连接对象

private:
    SqlConnPool();
    ~SqlConnPool();

private:
    std::queue<MYSQL *> m_connQue; //连接对象队列

    int m_max_conn; //最大连接数
    std::mutex m_mutex; //互斥量
};

//SqlConnPool.cpp
#include "../Log/Log.h"
#include "SqlConnPool.h"

SqlConnPool::SqlConnPool() {
}

SqlConnPool::~SqlConnPool() {
    ClosePool();
}

SqlConnPool *SqlConnPool::GetInstance() {
    static SqlConnPool sqlConnPool;
    return &sqlConnPool;
}

void SqlConnPool::Init(const std::string& host, int port, const std::string& user,
        const std::string& pwd, const std::string& db_name, int conn_size) {
    for (int i = 0; i < conn_size; i++) {
        MYSQL *sql = nullptr;
        sql = mysql_init(sql);
        if (!sql) {
            LOG_ERROR("MySql init error!");
        }

        sql = mysql_real_connect(sql, host.c_str(), user.c_str(), pwd.c_str(), db_name.c_str(), port, nullptr, 0);
        if (!sql) {
            LOG_ERROR("MySql Connect error!");
        }
        m_connQue.push(sql);
    }
    m_max_conn = conn_size;
}

MYSQL *SqlConnPool::GetConnObj() {
    MYSQL *sql = nullptr;
    if (m_connQue.empty()) {
        LOG_WARN("SqlConnPool busy!");
        return nullptr;
    }

    {
        std::lock_guard<std::mutex> locker(m_mutex);
        sql = m_connQue.front();
        m_connQue.pop();
    }

    return sql;
}

void SqlConnPool::FreeConnObj(MYSQL *conn) {
    if (conn != nullptr) {
        std::lock_guard<std::mutex> locker(m_mutex);
        m_connQue.push(conn);
    }
}

void SqlConnPool::ClosePool() {
    std::lock_guard<std::mutex> locker(m_mutex);
    while(!m_connQue.empty()) {
        auto item = m_connQue.front();
        m_connQue.pop();
        mysql_close(item);
    }
    mysql_library_end();
}
```
这个数据库连接池提供了连接对象的获取和释放接口，但是在实际使用中，需要使用者记得在使用完毕后去释放连接对象，这样的设计并不友好，因此我们添加一个数据库操作类，利用RAII来封装下数据库连接池，这样可以确保在使用完连接对象后，操作类会帮助我们析构释放对象。示例代码如下：
```
// SqlHandler.h
#include "SqlConnPool.h"

class SqlHandler {
public:
    SqlHandler(SqlConnPool *connpool);
    SqlHandler(MYSQL **sql, SqlConnPool *connpool);
    ~SqlHandler();

private:
    MYSQL *m_sql;
    SqlConnPool *m_connpool;
};

// SqlHandler.cpp
#include "SqlHandler.h"

SqlHandler::SqlHandler(SqlConnPool *connpool) {
    if (connpool) {
        m_sql = connpool->GetConnObj();
        m_connpool = connpool;
    }
}

SqlHandler::SqlHandler(MYSQL **sql, SqlConnPool *connpool) {
    if (connpool) {
        *sql = connpool->GetConnObj();
        m_sql = *sql;
        m_connpool = connpool;
    }
}

SqlHandler::~SqlHandler() {
    if (m_sql) {
        m_connpool->FreeConnObj(m_sql);
    }
}
```
更多内容，详见github [NetLib](https://github.com/cyh1998/NetLib)