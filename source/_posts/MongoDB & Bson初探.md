---
title: ET框架探索之初入MongoDB
date: 2019-11-21 15:41:15
categories: 
- Unity
- ET框架学习笔记
tags :
- ET
---
# 前言
这是我对于ET框架学习的第一个知识点的探索，因为要深入一个游戏框架要掌握的知识点非常多，幸好有方向就有了动力，首先了解ET框架所使用的MongoDB数据库
# 简介
- MongoDB是一种非关系型的数据库，对于NoSQL的解释是(Not Only SQL)也就是不仅仅只有SQL

- MongoDB的结构非常松散，他采取类似于Json的格式存储数据，这种格式叫做Bson采用键值对的方式存储

- MongoDB被称为是最像关系型数据库的非关系型数据库，因为他的一些语法非常类似sql语言，使得学过其他关系型数据库的人学习成本非常低
<!-- more -->
# ET为什么要使用MongoDB
至于为什么ET要使用MongoDB作为游戏数据库我找到了下面这一段话

>【群主】熊猫
mongobson做消息跟配置序列化，搭配数据库简直完美

>【群主】熊猫
mongo数据库天生完美配合游戏

>【群主】熊猫
网易内网消息也是bson，数据库也是mongodb，所以ET这点设计是没有啥问题的，甚至更加漂亮，因为python的mongo驱动实在尴尬，远远比不上C#驱动，网易游戏对于性能要求高的序列化还必须写到C++里面，C++驱动也很尴尬，导致必须自己实现一套类似C#驱动这种东西。用C#就不存在这个问题，性能完全不用担心，太方便了，你们捡到宝了

这是ET作者所说的话应该是很有含金量的，所以学习这个数据库对于以后游戏数据的存储应该用应该是巨大的，而且对于网络游戏的话，这种信息吞吐量巨大的项目使用非关系数据库进行读写是效率非常高的一种做法
# 为什么要学习MongoDB
面对业务需求的三高:
- High performance - 对数据库高并发读写的需求
- Huge Storage - 对海量数据的高效率存储和访问需求
- High Scalability && High Availability - 对数据库高可扩展性和高可用性的需求

# MongoDB Compare With MySql
![Mongo对比MySql2](https://s1.ax1x.com/2020/05/16/YgiMxH.png)
![Mongo对比MySql1](https://s1.ax1x.com/2020/05/16/YgilMd.png)


下面我会以Mongo的术语继续说明

# 对于MongoDB主键的一些认识
MongoDB自动会集合(表)生成主键，当插入一组数据时MongoDB自动生成一个ObjectID

例如我往数据库中的一个集合中插入这些数据
```
{
    title: 'MongoDB 教程', 
    description: 'MongoDB 是一个 Nosql 数据库',
    by: '菜鸟教程',
    url: 'http://www.runoob.com',
    tags: ['mongodb', 'database', 'NoSQL'],
    likes: 100
}

//当插入成功后查看 词条数据内容 就会变为

{ 
    "_id" : ObjectId("56064886ade2f21f36b03134"), 
    "title" : "MongoDB 教程", 
    "description" : "MongoDB 是一个 Nosql 数据库",
    "by" : "菜鸟教程", 
    "url" : "http://www.runoob.com", 
    "tags" : [ "mongodb", "database", "NoSQL" ],
    "likes" : 100 
}
```
我们可以看到他是一个函数传入了一串字符，这是MongoDB生成主键的方式

MongoDB中的主键以时间戳为基础，以进程编号，服务器名称为后缀，以此保证新数据填入时一定有一个独一无二的标识，从而免去与"用户注册表"的主键查询交互。
# MongoDB 特殊数据库
在我们安装完数据库后，它会自动生成3个数据库
- admin： 从权限的角度来看，这是"root"数据库。要是将一个用户添加到这个数据库，这个用户自动继承所有数据库的权限。一些特定的服务器端命令也只能从这个数据库运行，比如列出所有的数据库或者关闭服务器。
- local: 这个数据永远不会被复制，可以用来存储限于本地单台服务器的任意集合
- config: 当Mongo用于分片设置时，config数据库在内部使用，用于保存分片的相关信息。

# 创建数据库
安装数据库这里就不做介绍了，网上一大堆安装教程

MongoDB创建数据库的命令很简单，以命令行的方式创建只需要执行一条命令
```
//切换到当前数据库(不存在则创建一个新的)
use [数据库名称]
```
但是执行完这条命令之后，显示所有数据库并不会看到新创建的数据库
```
//显示所有数据库的命令
show dbs
```
为什么不会显示新创建的数据库呢，这就得谈到MongDB的创建机制，我们都知道存储空间 有速度快的内存，和用来持久化数据的硬盘，我们创建数据库的时候实际是创建在内存中，并没有写入到硬盘，只有我们数据库中有文档内容时，数据库才会真正存储到硬盘上，这时我们在使用show dbs命令就可以看到新创建的数据库了

其他命令
```
//显示当前数据库(可以看到在内存中没有被存储到硬盘的空数据库)
db
//删除数据库 (类似于js的方法)
db.dropDatabase()
```

# 语法简述 增删改查
直接给命令
```
//创建一个名为my的集合 非常类似函数很容易理解
db.createCollection("my")
//显示集合
show collections
//删除集合
db.[集合名称].drop()

//文档的插入 
//特别一提的是如果插入的集合不存在则会自动创建该名字的集合
db.[集合名称].save(document)//新版本中已废弃
//或者
db.[集合名称].insert(document)
//或者
//插入一条文档
db.[集合名称].insertOne(
   <document>,
   {
      writeConcern: <document>
   }
)
//插入多条文档
db.[集合名称].insertMany(
   [ <document 1> , <document 2>, ... ],
   {
      writeConcern: <document>,
      ordered: <boolean>
   }
)

//示例
db.col.insert
({
        "title" : "MongoDB 教程",
        "description" : "MongoDB 是一个 Nosql 数据库",
        "by" : "菜鸟教程",
        "url" : "http://www.runoob.com",
        "tags" : [
                "mongodb",
                "database",
                "NoSQL"
        ],
        "likes" : 100
})
//查询所有
db.[集合名称].find()
//条件查找
db.[集合名称].find({ title:"MongoDB 教程" }，...)

//更新
db.collection.update(
   <query>,
   <update>,
   {
     upsert: <boolean>,
     multi: <boolean>,
     writeConcern: <document>
   }
)

//删除
db.collection.remove(
   <query>,
   {
     justOne: <boolean>,
     writeConcern: <document>
   }
)
```
关于两种插入的方式
- save()：如果 _id 主键存在则更新数据，如果不存在就插入数据。该方法新版本中已废弃，可以使用 db.collection.insertOne() 或 db.collection.replaceOne() 来代替。
- insert(): 若插入的数据主键已经存在，则会抛 org.springframework.dao.DuplicateKeyException 异常，提示主键重复，不保存当前数据。

参数说明：

- document：要写入的文档。
- writeConcern：写入策略，默认为 1，即要求确认写操作，0 是不要求。
- ordered：指定是否按顺序写入，默认 true，按顺序写入
## 其他
插入语句可以使用try catch防止插入错误

# 结束语
其他语法和数据库的性质我就不一一介绍了，这篇文章主要的目的是介绍MongoDB的基本认识，为了ET框架学习做好基础，这会是一些列的知识体系

如果你拥有一种sql数据库的系统学习的过程，对于MongoDB的学习成本应该会很低，只要你多看多实践一定会有收获。
# 不应该使用MogoDB的场景
- 对于事务性强的数据
- 对于复杂查询等

# 参考链接
菜鸟教程:https://www.runoob.com/mongodb/mongodb-tutorial.html