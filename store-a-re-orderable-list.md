# 问题

有一个有序列表，可以修改列表中元素的位置，可以向列表添加元素或删除列表中的元素。

如何持久存储？

## 一、双向链表

```js
{
    "id": "a",
    "pre": null,
    "next": "b",
}
{
    "id": "b",
    "pre": "a",
    "next": "c"
}
// {
    "id": "c",
    "pre": "b",
    "next": "d"
}
{
    "id": "d",
    "pre": "c",
    "next": null
}
```

## 二、每条数据都有一个编号来记录位置

### 1. 顺序递增的整数

每个元素增加一个 position 字段表示元素所在的位置.  
如果插入元素，需要修改插入位置之后所有元素的 position

```js
{
    "id": "a",
    "position": 1,
}
{
    "id": "b",
    "position": 2
}
{
    "id": "c",
    "position": 3
}
{
    "id": "d",
    "position": 4
}
```

### 2. 相邻的记录间设置一个差值

比如差值为1000

```js
{
    "id": "a",
    "position": 1000
}
{
    "id": "b",
    "position": 2000
}
{
    "id": "c",
    "position": 3000
}
```

如果需要在 a 和 b 之间插入记录 d, d 的 position 为 `(1000 + 2000) / 2 = 1500`,
如果一直往 a 的后面插入记录,大约 log(1000) 次, 会出现数字相邻的情况，

```js
{
    "id": "a",
    "position": 1000
}
{
    "id": "x",
    "position": 1001
}
{
    "id": "w",
    "position": 1002
}
....
{
    "id": "b",
    "position": 2000
}
{
    "id": "c",
    "position": 3000
}
```

这个时候如果在 a 的后面插入记录 y ，可以把 a 之后所有的记录加 2000:

```js
{
    "id": "a",
    "position": 1000
}
{
    "id": "y",
    "position": 2000 // (1000 + 3001) / 2
}
{
    "id": "x",
    "position": 3001 // 1001 + 2000
}
{
    "id": "x",
    "position": 3002 // 1002 + 2000
}
....
{
    "id": "b",
    "position": 4000 // 2000 + 2000
}
{
    "id": "c",
    "position": 5000
}
```

> position 可以改为浮点数，但是在多次操作之后会有精度的问题。一样需要批量修改 position 来增大间距。

### 3. 用字符串来表示 position, 根据字典序来排序

```js
/* a, b, c, d, e, f, g, h, i, j, k, l, m, n, o, p, q, r, s, t, u, v, w, x, y, z*/
{
    "id": "a",
    "position": "1"
}
```

在 a 后插入 b

```js
{
    "id": "a",
    "position": "1"
}
{
    "id": "b",
    "position": "2"
}
```

在 a 后插入 c

```js
{
    "id": "a",
    "position": "1"
}
{
    "id": "c",
    "position": "15" // (10 + 20) / 2 = 15
}
{
    "id": "b",
    "position": "2"
}
```

## 三、根据数组位置记录位置

 直接操作数组（或者其他数据结构优化)

```js
[ "a", "b", "c", "d" ]
```

把 c 移到 a b 中间 

```js
[ "a", "c", "b", "d" ]
```

更改位置的时候可以直接读写 mysql, 或者先操作 redis, 然后用 offliner 把数据同步到 mysql

## 四、记录修改日志, 根据操作记录在内存复原列表

> 适用于列表种类不多，列表元素数据多但是对实时性要求不高  

服务器增加 log 系统, 记录每一个记录的增删改, offliner 扫描 log 把数据同步到 redis 或者 mysql 

如下， a 1 表示把 a 插入第一个位置

```js
a 1  // 按操作时间递增排序
b 2
c 2
d 3
b 1
```

```js
// a 1
{"id": "a"}

// b 2
{"id": "a"}
{"id": "b"}

// c 2
{"id": "a"}
{"id": "c"}
{"id": "b"}

// d 3
{"id": "a"}
{"id": "c"}
{"id": "d"}
{"id": "b"}

// b 1
{"id": "b"}
{"id": "a"}
{"id": "c"}
{"id": "d"}
```

ref:
https://stackoverflow.com/questions/5651299/store-positioned-list-in-database-gap-approach
