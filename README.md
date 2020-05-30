# LevelDB之旅
**LevelDB1.21版本**
## 编译
```
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Release .. && cmake --build .
```
注意：如果遇到 **CMake错误No CMAKE_CXX_COMPILER could be found.** 可以事先进行更新
```
sudo apt-get update
sudo apt-get install -y build-essential
```
## 链接
```
sudo cp build/libleveldb.a /usr/local/lib
sudo cp -R include/* /usr/local/include
```
## 测试
### 代码
```
#include <iostream>
#include "leveldb/db.h"

using namespace std;
using namespace leveldb;

int main() {
    DB *db ;
    Options op;
    op.create_if_missing = true;
    Status s = DB::Open(op,"/tmp/testdb",&db);

    if(s.ok()){
        cout << "创建成功" << endl;
        s = db->Put(WriteOptions(),"abcd","1234");
        if(s.ok()){
            cout << "插入数据成功" << endl;
            string value;
            s = db->Get(ReadOptions(),"abcd",&value);
            if(s.ok()){
                cout << "获取数据成功,value:" << value << endl;
            }
            else{
                cout << "获取数据失败" << endl;
            }
        }
        else{
            cout << "插入数据失败" << endl;
        }
    }
    else{
        cout << "创建数据库失败" << endl;
    }
    delete db;
    return 0;
}
```
### 编译运行
```
g++ test.cpp -pthread -lleveldb -lsnappy -o test && ./test
```

## 重点：如何使用YSCB测试LevelDB
> YCSB 是一种测试数据库的benchmark，它的使用原理是：
A. 目标数据库（待测试的数据库）作为**服务端**运行起来，并提供数据库操作相关的restful api，比如
http://localhost:8080/put
http://localhost:8080/get
http://localhost:8080/del
B.YCSB Client 作为**客户端**，通过restful api调用数据库，从而测试数据库的性能。

### 1.安装JSON依赖库和snappy依赖库
```
sudo apt install libjson-c-dev
sudo apt install libsnappy-dev
```

### 2.安装event依赖库
**需下载1.41版本**
编译安装：
```
./configure && make
sudo make install
sudo ln -s /usr/local/lib/libevent-1.4.so.2 /usr/lib/libevent-1.4.so.2
```

### 3.安装simpleleveldb
**simpleleveldb文件**
 simpleleveldb可以提供一套restful api，然后与leveldb数据库对接起来。
这样的话，ycsb 调用restful api，就会先调用simpleleveldb的相关操作，然后simpleleveldb再调用leveldb的相关操作。 完成一个衔接功能，即一个中间件。
```
git clone https://github.com/bitly/simplehttp.git
```

### 4.修改一些文件
由于源码中，include的路径有误，因此我们需要修改源码：
```
打开 simplehttp/simpleleveldb/str_list_set.c修改
#include <json/json.h> 为 #include <json-c/json.h>
打开 simplehttp/simpleleveldb/simpleleveldb.c修改
#include <json/json.h> 为 #include <json-c/json.h>
```
由于makefile中，对json库的引用也是错误的，-ljson应改为-ljson-c：
将
```
LIBS = -L. -L$(LIBSIMPLEHTTP_LIB) -L$(LIBEVENT)/lib -L/usr/local/lib -L$(LIBLEVELDB)/lib -levent -ljson -lsimplehttp -lleveldb -lm -lstdc++ -lsnappy -lpthread
```
改为（2处！！！）
```
LIBS = -L. -L$(LIBSIMPLEHTTP_LIB) -L$(LIBEVENT)/lib -L/usr/local/lib -L$(LIBLEVELDB)/lib -levent -ljson-c -lsimplehttp -lleveldb -lm -lstdc++ -lsnappy -lpthread
```

### 5.安装
安装simplehttp，进入simplehttp/simplehttp：
```
cd simplehttp
make
sudo make install
```
安装simpleleveldb，进入到simplehttp/simpleleveldb：
```
cd simpleleveldb
env LIBLEVELDB=/usr/local make
sudo make install
```

### 6.测试
打开终端，输入：
```
simpleleveldb --address=localhost --port=8080 --db-file=test
```
打开浏览器，输入：
> http://localhost:8080/put?key=name&value=Niko

显示：
> { "status_txt": "OK", "status_code": 200, "data": "" }

再输入：
> http://localhost:8080/get?key=name

显示：
> { "data": "Niko", "status_txt": "OK", "status_code": 200 }

注意：默认的数据库文件存储在**home**目录下

### 7.修改leveldb后的重编译
重编译leveldb：
```
cd build
make clean
cmake -DCMAKE_BUILD_TYPE=Release .. && cmake --build .
sudo cp libleveldb.a /usr/local/lib
sudo cp -R ../include/* /usr/local/include
```
重编译simpleleveldb：
```
cd simpleleveldb
make clean
env LIBLEVELDB=/usr/local make
sudo make install
```

### 运行YSCB客户端
安装python2：
```
sudo apt install python
```
编译yscb：
```
mvn -pl com.yahoo.ycsb:leveldb-binding -am clean package
```
测试运行：
```
./ycsb load leveldb -P workloads/workloada
./ycsb run leveldb -P workloads/workloada
```
