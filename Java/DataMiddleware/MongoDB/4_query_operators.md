
# 1. 操作符分类总览

MongoDB提供了丰富的查询操作符，分为以下大类：

| 类别 | 用途 | 代表操作符 |
|------|------|-----------|
| 比较操作符 | 值比较 | $eq, $gt, $gte, $lt, $lte, $ne, $in, $nin |
| 逻辑操作符 | 逻辑运算 | $and, $or, $nor, $not |
| 元素操作符 | 字段存在性和类型 | $exists, $type |
| 评估操作符 | 表达式/正则/JSON Schema | $expr, $regex, $jsonSchema, $mod |
| 数组操作符 | 数组查询 | $all, $elemMatch, $size |
| 地理空间 | 地理位置查询 | $geoWithin, $geoIntersects, $near, $nearSphere |
| 位操作符 | 按位运算 | $bitsAllSet, $bitsAnySet, $bitsAllClear, $bitsAnyClear |
| 投影操作符 | 结果投影 | $, $elemMatch, $slice, $meta |
| 杂项 | 注释等 | $comment, $rand |

# 2. 比较操作符

```javascript
// 准备测试数据
db.products.insertMany([
    { name: "商品A", price: 50, stock: 100, category: "电子" },
    { name: "商品B", price: 120, stock: 50, category: "家居" },
    { name: "商品C", price: 80, stock: 200, category: "电子" },
    { name: "商品D", price: 150, stock: 0, category: "户外" },
    { name: "商品E", price: 200, stock: 30, category: "电子" }
])
```

## 2.1 $eq / $ne（等于 / 不等于）

```javascript
// $eq：等于（可省略，默认就是等于）
db.products.find({ price: { $eq: 50 } })
db.products.find({ price: 50 })  // 等价写法

// $ne：不等于
db.products.find({ price: { $ne: 50 } })
db.products.find({ category: { $ne: "电子" } })

// 不等于null时也排除不存在的字段
db.products.find({ category: { $ne: null } })
```

## 2.2 $gt / $gte / $lt / $lte（大于 / 大于等于 / 小于 / 小于等于）

```javascript
// 大于
db.products.find({ price: { $gt: 100 } })

// 大于等于
db.products.find({ price: { $gte: 100 } })

// 小于
db.products.find({ price: { $lt: 100 } })

// 小于等于
db.products.find({ price: { $lte: 100 } })

// 范围查询：100 <= price <= 200
db.products.find({ price: { $gte: 100, $lte: 200 } })

// 日期范围查询
db.orders.find({ 
    createTime: { $gte: new Date("2024-01-01"), $lt: new Date("2024-02-01") } 
})
```

## 2.3 $in / $nin（包含 / 不包含）

```javascript
// $in：匹配数组中任意值的文档
db.products.find({ category: { $in: ["电子", "家居"] } })
db.products.find({ stock: { $in: [50, 100, 200] } })

// $nin：不匹配数组中任意值的文档
db.products.find({ category: { $nin: ["电子", "户外"] } })

// $in对数组字段的行为
db.users.find({ tags: { $in: ["Java"] } })  // tags数组包含"Java"
db.users.find({ tags: { $in: ["Java", "Python"] } })  // tags包含Java或Python
```

# 3. 逻辑操作符

## 3.1 $and

```javascript
// 显式$and（当同一字段有多个条件时自动使用）
db.products.find({
    $and: [
        { category: "电子" },
        { price: { $gt: 100 } }
    ]
})

// 隐式$and（逗号连接的多个条件）
db.products.find({ category: "电子", price: { $gt: 100 } })

// 同一字段多条件必须显式使用$and
db.products.find({
    $and: [
        { price: { $gt: 50 } },
        { price: { $lt: 150 } }
    ]
})
// 不能写成：{ price: { $gt: 50, $lt: 150 } } ← 但这种写法对于同一字段是合法的！
// 需要注意的是，对于不同字段，逗号连接等同于$and
```

## 3.2 $or

```javascript
// $or：满足任一条件
db.products.find({
    $or: [
        { category: "电子" },
        { price: { $lt: 60 } }
    ]
})

// $or可以匹配不同字段
db.products.find({
    $or: [
        { stock: { $lt: 10 } },       // 库存不足
        { status: "discontinued" }     // 已停产
    ]
})
```

`$or`性能注意：与`$in`不同，`$or`会对每个子句分别执行查询然后合并结果，可能无法充分利用索引。在可能的情况下，优先使用`$in`替代`$or`。

## 3.3 $nor

```javascript
// $nor：所有条件都不满足
db.products.find({
    $nor: [
        { category: "电子" },
        { stock: 0 }
    ]
})
// 等价于：category != "电子" AND stock != 0
```

## 3.4 $not

```javascript
// $not：对单个条件取反
db.products.find({ price: { $not: { $gt: 100 } } })
// 等价于 price <= 100

// $not与正则结合
db.users.find({ name: { $not: /^A/ } })  // 名字不以A开头
```

# 4. 元素操作符

## 4.1 $exists

```javascript
// 字段存在
db.users.find({ email: { $exists: true } })

// 字段不存在
db.users.find({ phone: { $exists: false } })

// 注意：null值的字段$exists: true也返回true
db.users.insertOne({ name: "test", email: null })
db.users.find({ email: { $exists: true } })  // 也会匹配
```

## 4.2 $type

```javascript
// 按类型查询（使用类型名称）
db.collection.find({ field: { $type: "string" } })
db.collection.find({ field: { $type: ["int", "long"] } })

// 按类型数字ID查询
db.collection.find({ field: { $type: 2 } })  // 2=string

// 常用类型别名
// "double", "string", "object", "array", "binData", "objectId",
// "bool", "date", "null", "regex", "int", "long", "decimal",
// "timestamp", "minKey", "maxKey"
```

# 5. 评估操作符

## 5.1 $expr

`$expr`允许在查询中使用聚合表达式，实现文档字段间的比较：

```javascript
// 查询支出大于预算的部门
db.departments.find({
    $expr: { $gt: ["$expense", "$budget"] }
})

// 查询折扣价低于原价50%的商品
db.products.find({
    $expr: { $lt: ["$discountPrice", { $multiply: ["$originalPrice", 0.5] }] }
})

// $expr + 聚合运算符实现复杂条件
db.orders.find({
    $expr: {
        $and: [
            { $gte: [{ $subtract: ["$total", "$discount"] }, 100] },
            { $eq: ["$status", "completed"] }
        ]
    }
})
```

## 5.2 $regex

```javascript
// 正则表达式查询
db.users.find({ name: { $regex: /^张/ } })
db.users.find({ name: { $regex: "张", $options: "i" } })

// $options选项：
// "i" — 不区分大小写
// "m" — 多行模式（^和$匹配行首尾）
// "x" — 忽略空白字符和#注释
// "s" — dotAll模式（.匹配换行符）

// 结合$text使用全文搜索比$regex效率高得多
```

## 5.3 $mod

```javascript
// 取模运算：查询age % 5 == 0的文档
db.users.find({ age: { $mod: [5, 0] } })

// 查询是偶数的stock
db.products.find({ stock: { $mod: [2, 0] } })
```

## 5.4 $jsonSchema

文档校验查询，筛选符合特定JSON Schema的文档：

```javascript
db.collection.find({
    $jsonSchema: {
        bsonType: "object",
        required: ["name", "age"],
        properties: {
            name: { bsonType: "string" },
            age: { bsonType: "int", minimum: 0, maximum: 150 }
        }
    }
})
```

# 6. 数组操作符

## 6.1 $all

匹配数组包含所有指定元素的文档（不关心顺序和其他元素）：

```javascript
// tags包含"Java"和"MongoDB"（可以同时有其他元素）
db.users.find({ tags: { $all: ["Java", "MongoDB"] } })

// 与$in的区别：
// $in: tags中任一元素匹配即可（tags包含Java或MongoDB）
// $all: tags必须包含列表中所有元素
```

## 6.2 $elemMatch

匹配数组中至少有一个元素满足所有条件的文档：

```javascript
// 准备数据
db.students.insertMany([
    {
        name: "张三",
        scores: [
            { subject: "语文", score: 85 },
            { subject: "数学", score: 92 },
            { subject: "英语", score: 78 }
        ]
    },
    {
        name: "李四",
        scores: [
            { subject: "语文", score: 95 },
            { subject: "数学", score: 65 },
            { subject: "英语", score: 88 }
        ]
    }
])

// 查询至少有一个成绩在80~90之间的科目
db.students.find({
    scores: {
        $elemMatch: {
            score: { $gte: 80, $lte: 90 }
        }
    }
})

// 不使用$elemMatch会导致"条件分离"问题！
// 这种写法可能匹配到不同数组元素分别满足条件的情况
db.students.find({
    "scores.score": { $gte: 80 },
    "scores.subject": "语文"  // 可能匹配到语文85分的元素，但score条件可能来自另一个元素
})
// 使用$elemMatch确保所有条件由同一个数组元素满足
```

## 6.3 $size

```javascript
// 查询tags数组长度为3的文档
db.users.find({ tags: { $size: 3 } })

// 注意：$size不支持范围查询，不能写成{ $gt: 3 }
// 需要通过聚合或新增计数字段来实现
// 变通方案：
db.users.find({ $expr: { $gt: [{ $size: "$tags" }, 3] } })
```

# 7. 地理空间操作符

## 7.1 $geoWithin

查询完全包含在指定几何区域内的文档：

```javascript
// 在矩形区域内
db.locations.find({
    location: {
        $geoWithin: {
            $geometry: {
                type: "Polygon",
                coordinates: [[
                    [116.0, 39.5], [117.0, 39.5],
                    [117.0, 40.5], [116.0, 40.5],
                    [116.0, 39.5]
                ]]
            }
        }
    }
})

// 在圆形区域内
db.locations.find({
    location: {
        $geoWithin: {
            $centerSphere: [[116.397, 39.908], 10 / 6371]  // 半径10km
        }
    }
})
```

## 7.2 $near / $nearSphere

按距离排序的查询（必须配合2dsphere或2d索引）：

```javascript
// 查找最近的5个位置
db.locations.find({
    location: {
        $nearSphere: {
            $geometry: { type: "Point", coordinates: [116.397, 39.908] },
            $maxDistance: 5000,   // 最大距离5000米
            $minDistance: 100     // 最小距离100米
        }
    }
}).limit(5)
```

# 8. 位操作符

```javascript
// $bitsAllSet: 指定位全部为1
db.collection.find({ permissions: { $bitsAllSet: [1, 3] } })

// $bitsAnySet: 指定位任意一个为1
db.collection.find({ permissions: { $bitsAnySet: [1, 3] } })

// $bitsAllClear: 指定位全部为0
db.collection.find({ permissions: { $bitsAllClear: [1, 3] } })

// $bitsAnyClear: 指定位任意一个为0
db.collection.find({ permissions: { $bitsAnyClear: [1, 3] } })
```

# 9. 查询性能优化提示

## 9.1 利用索引的操作符

以下操作符能有效利用索引：
- `$eq`, `$gt`, `$gte`, `$lt`, `$lte` — 范围查询，B树索引友好
- `$in` — 索引能有效优化
- `$exists` — 配合稀疏索引

## 9.2 需要谨慎使用的操作符

- `$regex`（非前缀匹配，如`/pattern/i`）：无法利用索引，需要全表扫描
- `$regex`（前缀匹配，如`/^前缀/`）：可以利用索引
- `$where`：完全无法利用索引，需要在服务端执行JavaScript
- `$expr`：字段间比较，通常无法利用索引
- `$mod`：取模运算无法利用普通索引
- `$nor`：通常导致全集合扫描

## 9.3 $or的索引优化

```javascript
// $or中的每个子句都应独立使用索引
db.collection.find({
    $or: [
        { field1: value1 },  // field1应有索引
        { field2: value2 }   // field2应有索引
    ]
})

// 用$in代替$or（当条件是同一字段时）
// 不如：db.collection.find({ $or: [{ field: "A" }, { field: "B" }] })
// 更好：db.collection.find({ field: { $in: ["A", "B"] } })
```
