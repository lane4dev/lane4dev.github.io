---
layout: post
title:  "MongoDB Cheat Sheet"
date:   2017-01-17 12:45:00 +0800
categories: Code
tags: [Database]
comments: true
---

MongoDB 是一个用 C++ 语言编写，指在为 Web 应用提供高性能数据存储访问的分布式文件存储数据库。它采用 JSON 的格式存储数据，并且兼容 JavaScript 语法， 常用于 Node.js 的运行环境下为用户提供数据访问与存储服务。

---

## 安装与配置

### 安装

- Windows 用户在[MongoDB 官方网站](https://www.mongodb.com/)直接下载`.exe`文件安装即可。

- Linux 用户用各自的包管理命令也能很方便的下载。

- Mac 用户...

### 运行配置

- Windows 用户需要在 C:\ 盘新建一个 Data 文件夹，然后在 Data 文件夹再中新建一个名为名为 db 的文件夹。然后可能需要设置一下环境变量，并且需要在管理员权限下运行 Mongo 的服务。

（ 反正我不在 Windows 下写代码 ╮（╯＿╰）╭ ）

- Linux ( Mac 一样 )用户同样需要在 / 目录下设置这样的文件夹：

```bash
$sudo mkdir -p /data/db
```

顺便还要将拥有者改为当前用户：

```bash
$sudo chown [User] /data/db
```

### 使用

Mongo 在使用过程中需要先运行 `mongod` 打开一个本地服务器，然后再运行 `mongo` 客户端，执行相关操作。

---

## MongoDB 的基本特点

在 Mongo 中的数据都是用 JSON 格式存储的，像下面这样：

```json
{
    "first_name" : "Albert",
    "last_name" : "Einstein",
    "fields" : ["Physics","philosophy"],
    "info" : {
        "born" : "14 March 1879",
        "died" : "18 April 1955",
        "age"  : 76
    }
}
```

Mongo 中可以运行 JavaScript 的程序：

```javascript
> function add( num1, num2){
    return num1 + num2;
}
> add(1, 1);
2
```

---

## 初始化操作

使用`db`命令查看当前所在的数据库

```shell
> db
test
```

使用`show dbs`查看所有数据库

```shell
> show dbs
admin            0.000GB
local            0.000GB
```

使用`use`命令建立一个新的数据库

```shell
> use person
switched to db person
> db
person
```

使用 `createCollection` 在当前数据库中新建一个数据集

```shell
> db.createCollection("physicist");
{ "ok" : 1 }
```

显示所有的数据集。

```shell
> show collections
physicist
```

---

## 添加数据

用`insert`函数添加一条JSON格式的数据

```shell
> db.physicist.insert({first_name : "Albert", last_name : "Einstein", age : 70 });
WriteResult({ "nInserted" : 1 })
```

用`find`函数( 不带参数的情况下 )查看当前数据集中所有数据，注意每次传入一条新的数据，系统都会自动生成一个独特的 id 。

```shell
> db.physicist.find();
{ "_id" : ObjectId("58d22d87bceb6b112a171303"), "first_name" : "Albert", "last_name" : "Einstein" }

```

`insert`函数同样可以一次性传入多条数据，但是注意必需要将它们放入一个数组中。

```shell
> db.physicist.insert([{first_name:"Isaac", last_name : "Newton",age : 84}, {first_name : "Galileo", last_name : "Galilei", gender : "male"}]);
BulkWriteResult({
    "writeErrors" : [ ],
    "writeConcernErrors" : [ ],
    "nInserted" : 2,
    "nUpserted" : 0,
    "nMatched" : 0,
    "nModified" : 0,
    "nRemoved" : 0,
    "upserted" : [ ]
})
```

---

## 查找与修改数据

使用 `pretty` 函数可以将 JSON 数据以更规整的格式输出

```shell
> db.physicist.find().pretty();
{
    "_id" : ObjectId("58d22d87bceb6b112a171303"),
    "first_name" : "Albert",
    "last_name" : "Einstein",
    "age" : 70
}
{
    "_id" : ObjectId("58d22deebceb6b112a171305"),
    "first_name" : "Galileo",
    "last_name" : "Galilei",
    "gender" : "male"
}
{
    "_id" : ObjectId("58d22e10bceb6b112a171306"),
    "first_name" : "Isaac",
    "last_name" : "Newton",
    "age" : 84
}
```

使用`update`函数更新数据。`update`函数的会根据传入第一参数的特征去数据库中寻找满足条件的数据，然后用第二个参数的内容更新找到的数据。这里有两个需要注意的地方 ：

第一是，`update`的第一个参数可以是任意合法的 键-值 对，如果数据库中存在多个符合条件的 键-值 对，则只会修改最先找到的那个。因此像下面的例子那样使用 ‘first_name’ ( 用户自己定义的键 )来确定需要修改的数据并不稳妥，正确的方法是使用系统提供的 id 唯一的确定需要修改的数据；

第二点，更新的过程中，`update`的第二个参数完全取代原有数据，因此必须保证传入第二个参数的数据是完整的，而不是用户仅想更新的那部分数据。注意更新后'牛顿'的年龄信息消失了。

```shell
> db.physicist.update({first_name : "Isaac"}, {first_name : "Isaac",last_name : "Newton", gender : "male"});
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```

这是更新后的结果

```shell
> db.physicist.find().pretty();
{
    "_id" : ObjectId("58d22d87bceb6b112a171303"),
    "first_name" : "Albert",
    "last_name" : "Einstein",
    "age" : 70
}
{
    "_id" : ObjectId("58d22deebceb6b112a171305"),
    "first_name" : "Galileo",
    "last_name" : "Galilei",
    "gender" : "male"
}
{
    "_id" : ObjectId("58d22e10bceb6b112a171306"),
    "first_name" : "Isaac",
    "last_name" : "Newton",
    "gender" : "male"
}
```

如果没有在数据库中找到符合条件的 键-值 对，则不进行任何更新操作。

```shell
> db.physicist.update({first_name : "Marie"}, {first_name : "Marie", last_name : "Curie", sex : "female"});
WriteResult({ "nMatched" : 0, "nUpserted" : 0, "nModified" : 0 })
```

但是可以设置如果没有在数据库中找到，就根据`insert`函数的第二个参数，新建一条数据( 注意这里传入的第三个参数 )。

```shell
> db.physicist.update({first_name : "Marie"}, {first_name : "Marie", last_name : "Curie", sex : "female"}, {upsert : true});
WriteResult({
    "nMatched" : 0,
    "nUpserted" : 1,
    "nModified" : 0,
    "_id" : ObjectId("58d2307449c059955b7d2a75")
})
```

在保留原有数据的情况下插入单条 键-值 对，需要使用到 `$set` 操作符，这里需要特别注意 `$set` 的使用方法。

```shell
> db.physicist.update({first_name : "Albert"}, {$set : {gender : "male"}});
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```

`$inc`操作符用来增加( 或者减少，用负数 )某一数值型变量( 之前这条数据中我故意将爱因斯坦的年龄信息设置错误，实际是76岁，通过加 6 的操作修改原有数据 )

```shell
> db.physicist.update({first_name:"Albert"}, {$inc:{age:6}});
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```

使用`$unset`操作符删除某条数据的特定 键-值 对

```shell
> db.physicist.update({first_name : "Albert"}, {$unset:{age : ""}});
```

使用`$rename`操作符修改某个 键 的名称

```shell
> db.physicist.update({first_name:"Marie"}, {$rename:{"sex":"gender"}});
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```

想要了解更详细的关于操作符的使用情况，可以访问[Mongo DB Documentation](https://docs.mongodb.com/manual/reference/operator/)

---

## 删除数据

使用`remove`函数删除数据

```shell
> db.physicist.remove({first_name:"Galileo"});
WriteResult({ "nRemoved" : 1 })
```

与使用`update`函数的情况相似，如果使用用户自己定义的 键-值 对的话，会一次性删除所有满足符合条件的数据。除了像下面这样规定只能删除一条数据以外，更稳妥的方法同样是使用 id 筛选需要删除的数据，

```shell
> db.physicist.remove({first_name:"Galileo"},{justOne:true});
```

---

## 查找与筛选

新建一个 stars 的数据集，将下面的JSON数据导入数据集中。

```json
{
    "first_name" : "Leonardo",
    "last_name" : "DiCaprio",
    "age" : 42,
    "gender" : "male",
    "occupation" : ["Actor", "Film producer"],
    "address" : {
        "street" : "Battery Park City",
        "city" : "New York",
        "country" : "U.S."
    }
},
{
    "first_name" : "Will",
    "last_name" : "Smith",
    "age" : 48,
    "gender" : "male",
    "occupation" : ["Actor", "producer", "rapper", "songwriter"],
    "address" : {
        "city" : "Los Angeles",
        "state" : "California",
        "country" : "U.S."
    }
},
{
    "first_name" : "Scarlett",
    "last_name" : "Johansson",
    "age" : 32,
    "gender" : "female",
    "occupation" : ["Actress", "singer", "model"],
    "address" : {
        "city" : "New York",
        "country" : "U.S."
    }
},
{
    "first_name" : "Adele",
    "age" : 28,
    "gender" : "female",
    "occupation" : ["singer", "songwriter"],
    "address" : {
        "city" : "London",
        "country" : "England"
    }
}
```

通过向`find`函数传递参数，可以查找和筛选特定满足特定条件的数据，比如在下面的例子中指定了要找的人名为 Scarlett 。

```shell
> db.stars.find({first_name:"Scarlett"}).pretty();
{
    "_id" : ObjectId("58d235193b9e78d5baa5031e"),
    "first_name" : "Scarlett",
    "last_name" : "Johansson",
    "age" : 32,
    "gender" : "female",
    "occupation" : [
        "Actress",
        "singer",
        "model"
    ],
    "address" : {
        "city" : "New York",
        "country" : "U.S."
    }
}
```

同样可以使用`$or`操作符，注意需要将多个筛选条件放进一个数组中。

```shell
> db.stars.find({$or:[{first_name:"Scarlett"}, {first_name:"Leonardo"}]}).pretty();
{
    "_id" : ObjectId("58d234943b9e78d5baa5031c"),
    "first_name" : "Leonardo",
    "last_name" : "DiCaprio",
    "age" : 42,
    "gender" : "male",
    "occupation" : [
        "Actor",
        "Film producer"
    ],
    "address" : {
        "street" : "Battery Park City",
        "city" : "New York",
        "country" : "U.S."
    }
}
{
    "_id" : ObjectId("58d235193b9e78d5baa5031e"),
    "first_name" : "Scarlett",
    "last_name" : "Johansson",
    "age" : 32,
    "gender" : "female",
    "occupation" : [
        "Actress",
        "singer",
        "model"
    ],
    "address" : {
        "city" : "New York",
        "country" : "U.S."
    }
}
```

通过性别查找数据

```shell
> db.stars.find({gender:"female"}).pretty();
{
    "_id" : ObjectId("58d235193b9e78d5baa5031e"),
    "first_name" : "Scarlett",
    "last_name" : "Johansson",
    "age" : 32,
    "gender" : "female",
    "occupation" : [
        "Actress",
        "singer",
        "model"
    ],
    "address" : {
        "city" : "New York",
        "country" : "U.S."
    }
}
{
    "_id" : ObjectId("58d235193b9e78d5baa5031f"),
    "first_name" : "Adele",
    "age" : 28,
    "gender" : "female",
    "occupation" : [
        "singer",
        "songwriter"
    ],
    "address" : {
        "city" : "London",
        "country" : "England"
    }
}
```

查找年龄小于 40 岁的人，这里 `$lt` 是 less than， 小于的意思。其他比较的操作符包括：

- `$eq` ( equal to ) ，等于
- `$gt` ( greater than ) ，大于
- `$gte` ( greater than or equal ) ，大于等于
- `$lte` ( less than or equal ) ， 小于等于
- `$ne` ( not equal ) ，不等于

```shell
> db.stars.find({age:{$lt:40}}).pretty();
{
    "_id" : ObjectId("58d235193b9e78d5baa5031e"),
    "first_name" : "Scarlett",
    "last_name" : "Johansson",
    "age" : 32,
    "gender" : "female",
    "occupation" : [
        "Actress",
        "singer",
        "model"
    ],
    "address" : {
        "city" : "New York",
        "country" : "U.S."
    }
}
{
    "_id" : ObjectId("58d235193b9e78d5baa5031f"),
    "first_name" : "Adele",
    "age" : 28,
    "gender" : "female",
    "occupation" : [
        "singer",
        "songwriter"
    ],
    "address" : {
        "city" : "London",
        "country" : "England"
    }
}
```

这里查找的是数据库中没有住在美国的名人

```shell
> db.stars.find({"address.country":{$ne:"U.S."}}).pretty();
{
    "_id" : ObjectId("58d235193b9e78d5baa5031f"),
    "first_name" : "Adele",
    "age" : 28,
    "gender" : "female",
    "occupation" : [
        "singer",
        "songwriter"
    ],
    "address" : {
        "city" : "London",
        "country" : "England"
    }
}
```

这里演示的是如何查找数组中的数据

```shell
> db.stars.find({"occupation":"singer"}).pretty();
{
    "_id" : ObjectId("58d235193b9e78d5baa5031e"),
    "first_name" : "Scarlett",
    "last_name" : "Johansson",
    "age" : 32,
    "gender" : "female",
    "occupation" : [
        "Actress",
        "singer",
        "model"
    ],
    "address" : {
        "city" : "New York",
        "country" : "U.S."
    }
}
{
    "_id" : ObjectId("58d235193b9e78d5baa5031f"),
    "first_name" : "Adele",
    "age" : 28,
    "gender" : "female",
    "occupation" : [
        "singer",
        "songwriter"
    ],
    "address" : {
        "city" : "London",
        "country" : "England"
    }
}
```

---

## 排序

使用`sort`函数排序，注意这里的 1 表示正向排序， -1 则表示反向排序。

```shell
> db.stars.find().sort({age:1}).pretty();
{
    "_id" : ObjectId("58d235193b9e78d5baa5031f"),
    "first_name" : "Adele",
    "age" : 28,
    "gender" : "female",
    "occupation" : [
        "singer",
        "songwriter"
    ],
    "address" : {
        "city" : "London",
        "country" : "England"
    }
}
{
    "_id" : ObjectId("58d235193b9e78d5baa5031e"),
    "first_name" : "Scarlett",
    "last_name" : "Johansson",
    "age" : 32,
    "gender" : "female",
    "occupation" : [
        "Actress",
        "singer",
        "model"
    ],
    "address" : {
        "city" : "New York",
        "country" : "U.S."
    }
}
{
    "_id" : ObjectId("58d234943b9e78d5baa5031c"),
    "first_name" : "Leonardo",
    "last_name" : "DiCaprio",
    "age" : 42,
    "gender" : "male",
    "occupation" : [
        "Actor",
        "Film producer"
    ],
    "address" : {
        "street" : "Battery Park City",
        "city" : "New York",
        "country" : "U.S."
    }
}
{
    "_id" : ObjectId("58d234c83b9e78d5baa5031d"),
    "first_name" : "Will",
    "last_name" : "Smith",
    "age" : 48,
    "gender" : "male",
    "occupation" : [
        "Actor",
        "producer",
        "rapper",
        "songwriter"
    ],
    "address" : {
        "city" : "Los Angeles",
        "state" : "California",
        "country" : "U.S."
    }
}
```

这里是一个反向排序的例子

```shell
> db.stars.find({"occupation":"Actor"}).sort({first_name:-1}).pretty();
{
    "_id" : ObjectId("58d234c83b9e78d5baa5031d"),
    "first_name" : "Will",
    "last_name" : "Smith",
    "age" : 48,
    "gender" : "male",
    "occupation" : [
        "Actor",
        "producer",
        "rapper",
        "songwriter"
    ],
    "address" : {
        "city" : "Los Angeles",
        "state" : "California",
        "country" : "U.S."
    }
}
{
    "_id" : ObjectId("58d234943b9e78d5baa5031c"),
    "first_name" : "Leonardo",
    "last_name" : "DiCaprio",
    "age" : 42,
    "gender" : "male",
    "occupation" : [
        "Actor",
        "Film producer"
    ],
    "address" : {
        "street" : "Battery Park City",
        "city" : "New York",
        "country" : "U.S."
    }
}
```

使用`count`函数统计数据

```shell
> db.stars.find().count();
4
```

使用`limit`函数限制数据显示的条数

```shell
> db.stars.find().sort({first_name:1}).limit(2).pretty();
{
    "_id" : ObjectId("58d235193b9e78d5baa5031f"),
    "first_name" : "Adele",
    "age" : 28,
    "gender" : "female",
    "occupation" : [
        "singer",
        "songwriter"
    ],
    "address" : {
        "city" : "London",
        "country" : "England"
    }
}
{
    "_id" : ObjectId("58d234943b9e78d5baa5031c"),
    "first_name" : "Leonardo",
    "last_name" : "DiCaprio",
    "age" : 42,
    "gender" : "male",
    "occupation" : [
        "Actor",
        "Film producer"
    ],
    "address" : {
        "street" : "Battery Park City",
        "city" : "New York",
        "country" : "U.S."
    }
}
```

之前演示过在 MongoDB 中使用 JavaScript 函数，在这个例子中，可以向 `forEach` 传递一个自己定义的函数，实现特殊的输出格式。

```shell
> db.stars.find().forEach( function(doc){ print("Actor/Actress name : " + doc.first_name)} );
Actor/Actress name : Leonardo
Actor/Actress name : Will
Actor/Actress name : Scarlett
Actor/Actress name : Adele

```

---
上面讲的都只是常用的部分，要了解更多，请参见 [MongoDB 官方文档](https://docs.mongodb.com/)
