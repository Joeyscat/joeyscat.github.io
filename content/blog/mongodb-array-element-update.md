+++
title = "MongoDB更新数组元素"
description = "文档中内嵌了数组时，如何更新数组中的元素"
tags = ["mongodb"]
date = 2022-01-08
+++

原文档

```js
[
  {
    _id: ObjectId("61d96f26284728a5cbd2b5da"),
    title: '你好啊',
    raw_content: '你好！！！！！！！！！！！！！！！！！！！！！！！！！！！！！\r\n\r\n+1',
    author_id: ObjectId("61d96e0af67275682b21ec9d"),
    comments: [
      {
        content: '好啊👌',
        author_id: ObjectId("61d96e0af67275682b21ec9d")
      },
      {
        content: '好啥呀？？',
        author_id: ObjectId("61d70cdc4a138b2ed4f4b087"),
        reply_to: ObjectId("61d96e0af67275682b21ec9d")
      }
    ]
  }
]
```

`comments` 数组中的文档只记录 `author_id`，但是我们想加上一个字段 `author_name` 方便查询。

可以这样[^1]：

```js
db.article.updateMany(
  { comments: { $exists: true, $ne: [] } },
  {
    $set: {
      "comments.$[elem].author_name": "Joeyscat"
    }
  },
  {
    arrayFilters: [
      {
        "elem.author_id": ObjectId("61d70cdc4a138b2ed4f4b087")
      }
    ]
  }
)
```

`{ comments: { $exists: true, $ne: [] } },` 是为了过滤掉没有 `comments` 的文档。

更新后的文档：

```js
[
  {
    _id: ObjectId("61d96f26284728a5cbd2b5da"),
    title: '你好啊',
    raw_content: '你好！！！！！！！！！！！！！！！！！！！！！！！！！！！！！\r\n\r\n+1',
    author_id: ObjectId("61d96e0af67275682b21ec9d"),
    comments: [
      {
        content: '好啊👌',
        author_id: ObjectId("61d96e0af67275682b21ec9d")
        author_name: 'rustman'
      },
      {
        content: '好啥呀？？',
        author_id: ObjectId("61d70cdc4a138b2ed4f4b087"),
        reply_to: ObjectId("61d96e0af67275682b21ec9d"),
        author_name: 'Joeyscat'
      }
    ]
  }
]
```

[^1]: [Update All Documents That Match arrayFilters in an Array](https://docs.mongodb.com/manual/reference/operator/update/positional-filtered/#update-all-documents-that-match-arrayfilters-in-an-array)
