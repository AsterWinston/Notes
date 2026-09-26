
# 1. 聚合框架概述

聚合框架（Aggregation Pipeline）是MongoDB中最强大的数据分析工具。它基于"管道（Pipeline）"概念：文档流经一系列管道阶段（Stage），每个阶段对文档进行转换、过滤、分组、计算等操作。

核心概念：
- **Pipeline（管道）**：一组有序的阶段（Stage）数组
- **Stage（阶段）**：一个数据处理单元，如$match、$group、$sort等
- **Expression（表达式）**：在阶段内部使用的操作符，如$sum、$avg等

```
输入文档 → [$match] → [$group] → [$sort] → [$project] → 输出结果
  doc1        过滤      分组聚合     排序       字段重塑       最终文档
  doc2
  doc3
  ...
```

# 2. 聚合基本语法

```javascript
db.collection.aggregate([
    { $match: { status: "active" } },
    { $group: { _id: "$category", total: { $sum: "$price" } } },
    { $sort: { total: -1 } }
])
```

# 3. 聚合管道阶段（Stages）

## 3.1 $match（过滤）

```javascript
// 过滤文档，类似find()的条件
// 应尽量放在管道开头，利用索引减少后续处理量
{ $match: { status: "completed", amount: { $gt: 100 } } }
```

## 3.2 $group（分组）

```javascript
// 按字段分组，进行聚合计算
{
    $group: {
        _id: "$category",                    // 分组字段（$字段名表示引用文档字段）
        totalAmount: { $sum: "$amount" },    // 求和
        avgAmount: { $avg: "$amount" },      // 平均值
        maxAmount: { $max: "$amount" },      // 最大值
        minAmount: { $min: "$amount" },      // 最小值
        count: { $sum: 1 },                  // 计数
        first: { $first: "$name" },          // 第一个
        last: { $last: "$name" },            // 最后一个
        items: { $push: "$name" },           // 拼成数组（注意内存限制）
        uniqueItems: { $addToSet: "$name" }, // 去重后拼成数组
        stdDev: { $stdDevPop: "$score" },    // 标准差
        merged: { $mergeObjects: "$obj" }    // 合并对象
    }
}

// 分组后多字段
{ $group: {
    _id: { category: "$category", year: { $year: "$date" } },
    count: { $sum: 1 }
}}
```

## 3.3 $sort（排序）

```javascript
// 按字段排序，1升序，-1降序
{ $sort: { totalAmount: -1, category: 1 } }
```

## 3.4 $project（字段重塑）

```javascript
// 选择、排除、重命名、计算字段
{
    $project: {
        _id: 0,                              // 排除_id
        fullName: { $concat: ["$firstName", " ", "$lastName"] },  // 拼接
        age: 1,                              // 保留字段
        birthYear: { $subtract: [2024, "$age"] },  // 计算字段
        isAdult: { $gte: ["$age", 18] },     // 布尔判断
        combined: {                          // 复杂表达式
            $cond: {
                if: { $gte: ["$score", 60] },
                then: "及格",
                else: "不及格"
            }
        }
    }
}
```

## 3.5 $limit / $skip（分页）

```javascript
// $limit: 限制返回文档数
{ $limit: 20 }

// $skip: 跳过N个文档
{ $skip: 40 }

// 分页：skip + limit（注意：大offset性能差）
db.collection.aggregate([
    { $sort: { _id: -1 } },
    { $skip: 1000 },
    { $limit: 20 }
])

// 推荐：游标分页（优于skip+limit）
db.collection.aggregate([
    { $match: { _id: { $gt: lastId } } },
    { $sort: { _id: 1 } },
    { $limit: 20 }
])
```

## 3.6 $unwind（展开数组）

```javascript
// 将数组字段展开，每个元素生成一个文档
db.orders.aggregate([
    { $unwind: "$items" }    // 每个订单item展开为独立文档
])

// 选项
{ $unwind: {
    path: "$items",                // 数组字段路径
    includeArrayIndex: "itemIndex", // 添加元素索引
    preserveNullAndEmptyArrays: false // 是否保留null/空数组文档
}}
```

## 3.7 $lookup（左外连接）

```javascript
// 类似SQL的LEFT OUTER JOIN
// 基本语法
{
    $lookup: {
        from: "users",             // 目标集合名
        localField: "userId",      // 当前集合的关联字段
        foreignField: "_id",       // 目标集合的关联字段
        as: "user"                 // 结果存入的数组字段名
    }
}

// 复杂join（pipeline语法，3.6+）
{
    $lookup: {
        from: "orders",
        let: { userId: "$_id" },
        pipeline: [
            { $match: {
                $expr: {
                    $and: [
                        { $eq: ["$userId", "$$userId"] },
                        { $gte: ["$total", 100] }
                    ]
                }
            }},
            { $sort: { createTime: -1 } },
            { $limit: 5 }
        ],
        as: "recentOrders"
    }
}
```

## 3.8 $addFields / $set（添加字段）

```javascript
// 添加新字段（$addFields和$set等价）
{
    $addFields: {
        fullName: { $concat: ["$firstName", " ", "$lastName"] },
        totalPrice: { $multiply: ["$price", "$quantity"] }
    }
}
```

## 3.9 $replaceRoot（替换根文档）

```javascript
// 用子文档替换当前文档
{ $replaceRoot: { newRoot: "$user" } }

// $mergeObjects：合并对象到根级别
{ $replaceRoot: { 
    newRoot: { $mergeObjects: ["$user", { orderCount: "$count" }] } 
}}
```

## 3.10 $bucket / $bucketAuto（分桶）

```javascript
// 手动分桶
{
    $bucket: {
        groupBy: "$age",
        boundaries: [0, 18, 30, 50, 100],
        default: "其他",
        output: {
            count: { $sum: 1 },
            names: { $push: "$name" }
        }
    }
}

// 自动分桶（指定桶数，自动计算边界）
{
    $bucketAuto: {
        groupBy: "$score",
        buckets: 5,
        output: {
            count: { $sum: 1 },
            avg: { $avg: "$score" }
        }
    }
}
```

## 3.11 $facet（多维度分析）

```javascript
// 在同一组文档上并行执行多个聚合管道
db.products.aggregate([
    {
        $facet: {
            "byCategory": [
                { $group: { _id: "$category", count: { $sum: 1 } } }
            ],
            "priceStats": [
                { $group: { _id: null, avg: { $avg: "$price" }, min: { $min: "$price" }, max: { $max: "$price" } } }
            ],
            "top5Expensive": [
                { $sort: { price: -1 } },
                { $limit: 5 },
                { $project: { name: 1, price: 1 } }
            ]
        }
    }
])
```

## 3.12 $out / $merge（输出结果）

```javascript
// $out: 将聚合结果写入新集合（覆盖已有集合）
db.orders.aggregate([
    { $group: { _id: "$customerId", total: { $sum: "$amount" } } },
    { $out: "customerTotals" }
])

// $merge: 将结果合并到现有集合（4.2+，更灵活）
db.orders.aggregate([
    { $group: { _id: "$customerId", total: { $sum: "$amount" } } },
    {
        $merge: {
            into: "customerTotals",
            on: "_id",
            whenMatched: "merge",       // replace | keepExisting | merge | fail | [...]
            whenNotMatched: "insert"    // insert | discard | fail
        }
    }
])
```

## 3.13 $sample（随机采样）

```javascript
// 随机返回N个文档
{ $sample: { size: 10 } }
```

## 3.14 $sortByCount

```javascript
// 等价于 $group + $sort 的组合
// { $group: { _id: "$field", count: { $sum: 1 } } }, { $sort: { count: -1 } }
{ $sortByCount: "$category" }
```

# 4. 聚合表达式运算符

## 4.1 算术运算符

```javascript
{ $add: [ "$a", "$b", 10 ] }               // a + b + 10
{ $subtract: [ "$a", "$b" ] }              // a - b
{ $multiply: [ "$a", "$b" ] }              // a * b
{ $divide: [ "$a", "$b" ] }                // a / b
{ $mod: [ "$a", "$b" ] }                   // a % b
{ $abs: "$value" }                          // 绝对值
{ $ceil: 12.3 }                             // 向上取整 → 13
{ $floor: 12.7 }                            // 向下取整 → 12
{ $round: [12.345, 2] }                     // 四舍五入 → 12.35
{ $trunc: [12.345, 1] }                     // 截断 → 12.3
{ $sqrt: 25 }                               // 平方根 → 5
{ $pow: [2, 3] }                            // 2^3 → 8
{ $log: [100, 10] }                         // log10(100) → 2
```

## 4.2 字符串运算符

```javascript
{ $concat: ["Hello", " ", "World"] }                    // "Hello World"
{ $toUpper: "$name" }                                    // 转大写
{ $toLower: "$name" }                                    // 转小写
{ $substrCP: ["$name", 0, 3] }                           // 截取
{ $strLenCP: "$name" }                                   // 字符串长度
{ $split: ["$tags", ","] }                               // 分割为数组
{ $trim: { input: "$name", chars: " " } }                // 去除两端空格
{ $replaceOne: { input: "$text", find: "a", replacement: "A" } }
{ $indexOfCP: ["$name", "张"] }                          // 子串位置
{ $regexFind: { input: "$text", regex: /pattern/ } }     // 正则查找
```

## 4.3 日期运算符

```javascript
{ $year: "$date" }          // 年份
{ $month: "$date" }         // 月份(1-12)
{ $dayOfMonth: "$date" }    // 日(1-31)
{ $dayOfWeek: "$date" }     // 星期几(1-7,周日=1)
{ $hour: "$date" }          // 小时(0-23)
{ $minute: "$date" }        // 分钟(0-59)
{ $second: "$date" }        // 秒(0-59)
{ $week: "$date" }          // 一年中第几周(0-53)
{ $isoWeek: "$date" }       // ISO周数
{ $dayOfYear: "$date" }     // 一年中第几天
{ $isoDayOfWeek: "$date" }  // ISO星期(1-7,周一=1)

// 日期运算
{ $dateAdd: { startDate: "$date", unit: "day", amount: 7 } }      // 加7天
{ $dateSubtract: { startDate: "$date", unit: "month", amount: 1 } } // 减1月
{ $dateDiff: { startDate: "$start", endDate: "$end", unit: "day" } } // 相差天数

// 日期转字符串
{ $dateToString: { format: "%Y-%m-%d", date: "$date" } }
{ $dateFromString: { dateString: "2024-02-15", format: "%Y-%m-%d" } }
```

`$dateToString`格式符：

| 格式符 | 含义 | 示例 |
|--------|------|------|
| %Y | 4位年 | 2024 |
| %m | 2位月 | 02 |
| %d | 2位日 | 15 |
| %H | 2位时(24h) | 14 |
| %M | 2位分 | 30 |
| %S | 2位秒 | 45 |
| %L | 3位毫秒 | 123 |
| %j | 年中第几天 | 046 |
| %w | 星期几(数字) | 1 |
| %u | ISO星期几 | 1 |

## 4.4 条件运算符

```javascript
// $cond (if-then-else)
{ $cond: {
    if: { $gte: ["$score", 60] },
    then: "及格",
    else: "不及格"
}}

// $ifNull (类似COALESCE)
{ $ifNull: ["$nickname", "$name"] }         // 取第一个非null值

// $switch
{ $switch: {
    branches: [
        { case: { $gte: ["$score", 90] }, then: "优秀" },
        { case: { $gte: ["$score", 80] }, then: "良好" },
        { case: { $gte: ["$score", 60] }, then: "及格" }
    ],
    default: "不及格"
}}
```

## 4.5 数组运算符

```javascript
{ $size: "$tags" }                          // 数组长度
{ $slice: ["$tags", 3] }                    // 取前3个元素
{ $slice: ["$tags", -3] }                   // 取后3个元素
{ $slice: ["$tags", 1, 3] }                // 跳过1，取3个

{ $arrayElemAt: ["$tags", 0] }             // 取指定索引
{ $first: "$tags" }                         // 数组第一个元素
{ $last: "$tags" }                          // 数组最后一个元素

{ $filter: {
    input: "$items",
    as: "item",
    cond: { $gte: ["$$item.price", 50] }
}}                                           // 过滤数组

{ $map: {
    input: "$items",
    as: "item",
    in: { $multiply: ["$$item.price", 1.2] }
}}                                           // 映射转换

{ $reduce: {
    input: "$items",
    initialValue: 0,
    in: { $add: ["$$value", "$$this.price"] }
}}                                           // 归约

{ $in: ["$color", ["red", "blue"]] }        // 元素是否在数组中
{ $reverseArray: "$tags" }                   // 反转数组
{ $zip: { inputs: [ "$a", "$b" ] } }         // 拉链式合并
{ $setUnion: [ "$a", "$b" ] }               // 并集
{ $setIntersection: [ "$a", "$b" ] }        // 交集
{ $setDifference: [ "$a", "$b" ] }          // 差集(a-b)
{ $setEquals: [ "$a", "$b" ] }              // 是否相等
{ $setIsSubset: [ "$a", "$b" ] }            // a是否为b子集

// $unwind替换方案（某些场景）
{ $first: { $sortArray: { input: "$scores", sortBy: { score: -1 } } } }
```

## 4.6 类型转换运算符

```javascript
{ $toString: "$age" }
{ $toInt: "$ageStr" }
{ $toLong: "$value" }
{ $toDouble: "$value" }
{ $toDecimal: "$value" }
{ $toBool: "$value" }
{ $toDate: "$dateStr" }
{ $toObjectId: "$idStr" }

// 通用转换
{ $convert: { input: "$ageStr", to: "int", onError: 0, onNull: 0 } }
{ $type: "$field" }  // 获取字段类型字符串
```

# 5. 聚合管道的优化

## 5.1 管道顺序优化原则

```
1. $match → 尽早过滤，减少数据量
2. $project → 尽早裁剪字段，减少内存
3. $sort → 合理利用索引
4. $limit + $skip → 在$sort之后
5. $lookup → 在过滤后、分组前
6. $group → 聚合计算
7. $unwind → 在$match之后减少展开量
```

## 5.2 聚合管道与索引

```javascript
// 聚合管道的$match和$sort阶段会利用索引
// 前提：$match和$sort在管道开头，且未经过$unwind/$group等变换

// 有效使用索引：
db.orders.aggregate([
    { $match: { customerId: "C001" } },    // 第一个阶段，使用索引
    { $sort: { orderDate: -1 } },          // 第二个阶段，利用索引排序
    { $limit: 10 }
])
```

## 5.3 内存限制

```javascript
// 聚合管道默认每个阶段限制100MB内存
// 对于大数据量排序/分组，使用allowDiskUse允许使用磁盘
db.collection.aggregate([...], { allowDiskUse: true })
```

## 5.4 聚合管道调试

```javascript
// 逐个阶段调试（一次加一个阶段）
db.collection.aggregate([{ $match: {...} }])
db.collection.aggregate([{ $match: {...} }, { $group: {...} }])

// 使用$limit限制调试数据量
db.collection.aggregate([
    { $limit: 100 },
    { $match: {...} },
    // ...
])
```

# 6. 聚合实战示例

## 6.1 销售统计报表

```javascript
db.orders.aggregate([
    // 1. 过滤时间范围
    { $match: {
        orderDate: {
            $gte: ISODate("2024-01-01"),
            $lt: ISODate("2025-01-01")
        },
        status: { $in: ["completed", "shipped"] }
    }},
    // 2. 关联用户信息
    { $lookup: {
        from: "users",
        localField: "userId",
        foreignField: "_id",
        as: "user"
    }},
    { $unwind: "$user" },
    // 3. 按月份和分类分组
    { $group: {
        _id: {
            year: { $year: "$orderDate" },
            month: { $month: "$orderDate" },
            category: "$category"
        },
        totalRevenue: { $sum: "$total" },
        orderCount: { $sum: 1 },
        avgOrderValue: { $avg: "$total" },
        customerCount: { $addToSet: "$userId" }
    }},
    // 4. 计算额外指标
    { $addFields: {
        customerCount: { $size: "$customerCount" }
    }},
    // 5. 排序
    { $sort: { "_id.year": -1, "_id.month": -1, totalRevenue: -1 } }
])
```

## 6.2 用户行为漏斗分析

```javascript
db.events.aggregate([
    // 1. 按用户和事件类型分桶
    { $group: {
        _id: "$userId",
        events: { $push: { type: "$eventType", time: "$timestamp" } }
    }},
    // 2. 判断用户所处漏斗阶段
    { $addFields: {
        hasRegistered: { $in: ["注册", "$events.type"] },
        hasLoggedIn: { $in: ["登录", "$events.type"] },
        hasPurchased: { $in: ["购买", "$events.type"] }
    }},
    // 3. 按阶段分组
    { $facet: {
        registered: [
            { $match: { hasRegistered: true } },
            { $count: "count" }
        ],
        loggedIn: [
            { $match: { hasLoggedIn: true } },
            { $count: "count" }
        ],
        purchased: [
            { $match: { hasPurchased: true } },
            { $count: "count" }
        ]
    }}
])
```
