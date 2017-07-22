最近在入门mongodb，期间踩了不少坑，在这里记录一下。

## 系统及mongodb版本
1. 系统：OS X
2. mongodb版本： 3.4 Community Edition

## 安装
安装方式有两种，一种是直接在官网下载安装包，然后解压，最后还需要设置PATH，比较麻烦，这里推荐的是第二种按照方式：[Homebrew](https://brew.sh/)，安装成功之后会自动帮我们设置好PATH，非常方便

使用Homebrew安装
```javascript
brew install mongodb
```
安装完成之后我们需要创建一个提供给mongodb存储数据的文件夹，mongodb默认使用的文件夹是 /data/db，我们先来创建一个：
```javascript
mkdir -p /data/db
```
如果你想使用其他路径文件夹来存储数据可以在启动命令的时候加上dbpath来指定所需要使用的目录路径：
```javascript
mongod --dbpath <path to data directory>
```
## 启动mongodb服务器
接下来我们来启动mongodb的服务，在控制台输入
```javascript
mongod
```
![](/asserts/1.png)
yes，启动成功！
从控制台上打印出来的日志我们可以看到mongodb默认的端口号是27017，dbpath=/data/db，我们可以注意到日志中有一行这样的warning信息：
```
2017-04-02T18:46:54.942+0800 I CONTROL  [initandlisten] ** WARNING: Access control is not enabled for the database.
```
> 直接使用mongod命令启动的是 **没有进行访问控制** 的访问方式，即可以不用登录就能访问所有数据库

接下来我们使用需要进行访问控制的启动方式，在这之前我们需要先创建一个角色，mongodb提供了mongo来启动mongodb的客户端
> **mongod** 是mongodb的服务端启动命令
**mongo**是mongodb的客户端启动命令

## 启动mongodb客户端：
```
mongo
```
![](/asserts/2.png)
在这我们可以输入mongodb的命令来操作数据库，ok，我们现在来创建一个角色：
```
use admin // 使用admin数据库
db.createUser(
  {
    user: "myUserAdmin",
    pwd: "abc123",
    roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
  }
)
```
创建成功，关闭mongodb服务器和客户端，用访问控制的方式启动服务器：
```
mongod --auth // 以访问控制的方式启动服务器
mongo // 启动客户端
```
在客户端进行登录：
```
use admin // 使用admin数据库
db.auth("myUserAdmin", "abc123")
```
## 在NodeJs使用mongodb
首先我们需要在NodeJs里面安装mongodb的驱动
```javascript
npm install mongodb --save
```
使用方式很简单，在app.js里面添加以下代码
```javascript
const MongoClient = require('mongodb').MongoClient
  , assert = require('assert');

// Connection URL
const url = 'mongodb://myUserAdmin:abc123@127.0.0.1:27017/admin';

// Use connect method to connect to the server
MongoClient.connect(url, function(err, db) {
  assert.equal(null, err);
  console.log("Connected successfully to server");
  db.close();
});

// Connected successfully to server
```
ok，接下来就是看文档的事情了~