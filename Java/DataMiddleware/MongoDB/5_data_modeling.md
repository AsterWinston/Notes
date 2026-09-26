
# 1. MongoDB数据建模的核心哲学

MongoDB的数据建模与关系型数据库有本质不同。RDBMS强调"如何存储"（范式化、消除冗余），而MongoDB强调"如何使用"（根据查询模式优化数据结构）。

核心设计原则：
- **数据一起访问，就一起存储**（嵌入）
- **数据独立访问，就分开存储**（引用）
- **根据应用的读写模式设计模型，而非根据数据的内在结构**

# 2. 嵌入式文档 vs 引用式文档

## 2.1 嵌入式文档（Embedded / Denormalized）

```javascript
// 嵌入式建模示例：订单与订单项
db.orders.insertOne({
    _id: "ORD001",
    userId: "U001",
    orderDate: new Date(),
    status: "processing",
    items: [
        { productId: "P001", name: "机械键盘", price: 299, quantity: 1 },
        { productId: "P002", name: "鼠标", price: 99, quantity: 2 }
    ],
    shippingAddress: {
        name: "张三",
        city: "北京",
        street: "中关村大街",
        phone: "13800138000"
    },
    totalAmount: 497
})
```

**优点**：
- 一次查询获取所有数据（无JOIN，高性能）
- 单文档原子操作（更新订单状态+所有订单项）
- 数据局部性好（连续存储，减少磁盘IO）

**缺点**：
- 文档膨胀（最大16MB限制）
- 冗余数据可能导致不一致
- 难以跨文档查询嵌入的子文档

**适用场景**：
- 一对一或一对少关系
- 子数据离开父数据没有独立意义（如订单项、收货地址）
- 数据总是被一起读取
- 子数据很少独立更新

## 2.2 引用式文档（Reference / Normalized）

```javascript
// 引用式建模示例：用户与订单（一对多）
// 用户集合
db.users.insertOne({
    _id: "U001",
    name: "张三",
    email: "zhangsan@example.com"
})

// 订单集合
db.orders.insertOne({
    _id: "ORD001",
    userId: "U001",             // 引用用户ID
    orderDate: new Date(),
    totalAmount: 497,
    items: [
        { productId: "P001", name: "机械键盘", price: 299, quantity: 1 }
    ]
})

// 查询时通过$lookup关联
db.orders.aggregate([
    { $match: { userId: "U001" } },
    {
        $lookup: {
            from: "users",
            localField: "userId",
            foreignField: "_id",
            as: "user"
        }
    },
    { $unwind: "$user" }
])
```

**优点**：
- 避免数据重复，保持一致性
- 文档大小可控
- 独立数据可以独立查询

**缺点**：
- 需要多次查询或$lookup
- 无法在单文档操作中保证跨集合原子性（除非用事务）

**适用场景**：
- 多对多关系
- 子数据需要独立频繁更新
- 数据被多个父文档引用
- 子数据数量不可预测或可能无限增长

## 2.3 两种模式选择决策树

```
数据是否总是一起访问？
├── 是 → 嵌入
└── 否 → 引用
数据关系是包含（contain）还是关联（associate）？
├── 包含 → 嵌入
└── 关联 → 引用
子数据是否有独立于父数据的生命周期？
├── 是 → 引用
└── 否 → 嵌入
子文档数量是否无限增长？
├── 是 → 引用（或桶模式）
└── 否 → 嵌入
```

## 2.4 混合模式

实践中常将两种模式混用：

```javascript
// 商品集合：商品基本信息 + 嵌入的规格（总是需要的）
db.products.insertOne({
    _id: "P001",
    name: "机械键盘",
    brand: "Cherry",
    price: 299,
    specs: {                    // 嵌入：基本规格信息
        switch: "红轴",
        layout: "87键",
        backlight: "RGB"
    }
})

// 商品详情集合：大数据量详情页面（不总需要）
db.productDetails.insertOne({
    productId: "P001",          // 引用
    description: "这是一款...",  // 长文本
    images: [...],              // 大量图片URL
    reviews: [...]              // 大量评论
})
```

# 3. 常见设计模式

## 3.1 多态模式（Polymorphic Pattern）

当集合中的文档结构相似但不完全相同时：

```javascript
// 多态集合：不同类型的支付记录
db.payments.insertMany([
    {
        type: "credit_card",
        amount: 299,
        cardNumber: "****1234",
        cardType: "Visa"
    },
    {
        type: "wechat",
        amount: 150,
        openid: "oABC123",
        transactionId: "TXN001"
    },
    {
        type: "bank_transfer",
        amount: 5000,
        bankName: "工商银行",
        accountNo: "****5678"
    }
])

// 通过type字段实现多态
db.payments.find({ type: "credit_card" })
db.payments.find({ type: { $in: ["wechat", "alipay"] } })
```

适用场景：单一视图展示不同类型数据，如CMS文章系统、支付记录、事件日志。

## 3.2 桶模式（Bucket Pattern）

适用于时序数据或需要聚合展示的数据：

```javascript
// 桶模式：每小时聚合成一个文档
{
    _id: ObjectId(),
    sensorId: "S001",
    startTime: ISODate("2024-01-15T10:00:00Z"),
    endTime: ISODate("2024-01-15T10:59:59Z"),
    recordCount: 60,
    avgTemperature: 23.5,
    maxTemperature: 25.2,
    minTemperature: 21.8,
    measurements: [
        { time: "10:01", temp: 23.0 },
        { time: "10:02", temp: 23.1 },
        // ... 60条数据
    ]
}
```

桶模式的优势：
- 减少文档数量（60条→1条）
- 减少索引大小
- 预聚合数据提升查询效率
- 方便归档和过期处理

## 3.3 子集模式（Subset Pattern）

大数据集按照访问频率拆分冷热数据：

```javascript
// 热数据（经常访问的摘要信息）
db.productSummaries.insertOne({
    _id: "P001",
    name: "机械键盘",
    price: 299,
    rating: 4.5,
    reviewCount: 230,
    mainImage: "kb.jpg"
})

// 冷数据（不常访问的详情信息）
db.productDetails.insertOne({
    _id: "P001",
    description: "详细描述...",
    specs: {...},
    allImages: [...],
    reviews: [...],       // 所有评论
    relatedProducts: [...]
})
```

## 3.4 属性模式（Attribute Pattern）

处理大量不同字段但需要支持搜索的场景：

```javascript
// 不推荐：字段爆炸
{
    name: "商品A",
    attr_color: "红色",
    attr_size: "XL",
    attr_weight: "500g",
    attr_material: "纯棉",
    // ... 可能有上百个属性
}

// 推荐：属性模式
{
    name: "商品A",
    attributes: [
        { key: "color", value: "红色" },
        { key: "size", value: "XL" },
        { key: "weight", value: "500g" },
        { key: "material", value: "纯棉" }
    ]
}

// 查询：获取所有红色商品
db.products.find({ 
    attributes: { $elemMatch: { key: "color", value: "红色" } } 
})

// 索引
db.products.createIndex({ "attributes.key": 1, "attributes.value": 1 })
```

## 3.5 溢出桶模式（Outlier Pattern / Extended Reference）

用于防止少数"异常"文档导致数组无限增长：

```javascript
// 主文档：限制最多嵌入100条评论
{
    _id: "P001",
    name: "热门商品",
    reviewCount: 1520,
    latestReviews: [
        // 仅嵌入最近100条评论
        { userId: "U100", rating: 5, comment: "好评", time: ... },
        // ...
    ],
    hasExtra: true       // 标记是否有更多溢出数据
}

// 溢出文档
{
    productId: "P001",
    reviews: [
        // 其余所有评论
    ]
}
```

## 3.6 计算模式（Computed Pattern）

在写入时计算，避免读取时重复计算：

```javascript
// 不要在读取时计算聚合值
// 而是在写入时维护聚合值
{
    _id: "P001",
    name: "商品",
    reviewCount: 230,        // 写评论时原子自增
    avgRating: 4.5,          // 写评论时重新计算
    totalSales: 15000        // 下单时原子递增
}

// 写入评论时的原子更新
db.products.updateOne(
    { _id: "P001" },
    {
        $inc: { reviewCount: 1 },
        $set: { avgRating: newAvg }
    }
)
```

## 3.7 树形结构模式

MongoDB中存储树结构的几种方式：

**① 父引用模式（Parent Reference）**：

```javascript
// 每个节点存储父节点ID
{ _id: "北京", parent: null }
{ _id: "海淀", parent: "北京" }
{ _id: "朝阳", parent: "北京" }
{ _id: "中关村", parent: "海淀" }

// 查询子节点：简单
db.categories.find({ parent: "北京" })

// 查询祖先：需要逐级查询
```

**② 子引用模式（Child Reference）**：

```javascript
// 每个节点存储所有子节点ID
{ _id: "北京", children: ["海淀", "朝阳", "丰台"] }

// 查询子节点：直接获取
// 查询祖先：困难
```

**③ 祖先数组模式（Ancestors Array）**：

```javascript
// 每个节点存储所有祖先路径
{ _id: "中关村", ancestors: ["北京", "海淀"], parent: "海淀" }

// 查询所有后代：db.categories.find({ ancestors: "北京" })
// 查询所有祖先：直接从ancestors获取
// 更新困难：移动节点需要更新所有后代
```

**④ 物化路径模式（Materialized Path）**：

```javascript
// 使用路径字符串
{ _id: "中关村", path: "北京,海淀,中关村" }

// 查询后代：使用正则
db.categories.find({ path: /^北京/ })

// 查询直接子节点：使用正则
db.categories.find({ path: /^北京,[^,]+$/ })
```

# 4. 一对多关系的建模选择

| 关系规模 | 推荐模式 | 说明 |
|---------|---------|------|
| 一对极少（<几十） | 嵌入 | 如用户→收货地址 |
| 一对多（几百） | 子引用（父持有子ID数组） | 如分类→商品 |
| 一对很多（几千+） | 父引用（子持有父ID） | 如商品→评论 |
| 一对无限 | 子引用+分页+溢出桶 | 如传感器数据 |

# 5. Schema Validation（模式校验）

## 5.1 基础校验

```javascript
db.createCollection("users", {
    validator: {
        $jsonSchema: {
            bsonType: "object",
            required: ["name", "email", "age"],
            properties: {
                name: {
                    bsonType: "string",
                    description: "必须是字符串且必填",
                    minLength: 2,
                    maxLength: 50
                },
                email: {
                    bsonType: "string",
                    pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
                },
                age: {
                    bsonType: "int",
                    minimum: 0,
                    maximum: 150
                },
                status: {
                    enum: ["active", "inactive", "suspended"]
                },
                address: {
                    bsonType: "object",
                    properties: {
                        city: { bsonType: "string" },
                        street: { bsonType: "string" }
                    }
                },
                tags: {
                    bsonType: "array",
                    items: { bsonType: "string" },
                    uniqueItems: true
                }
            }
        }
    },
    validationLevel: "strict",       // strict | moderate | off
    validationAction: "error"        // error | warn
})
```

## 5.2 校验级别

| 级别 | 说明 |
|------|------|
| strict | 所有插入和更新都进行校验（默认） |
| moderate | 仅对已存在且满足校验条件的文档进行校验更新 |
| off | 关闭校验 |

## 5.3 动态调整校验规则

```javascript
// 修改校验器
db.runCommand({
    collMod: "users",
    validator: { /* 新的校验规则 */ },
    validationLevel: "moderate",
    validationAction: "warn"
})
```

# 6. 文档设计实践准则

## 6.1 设计流程

1. **明确查询模式**：列出应用会用到的所有查询、读频率、写频率
2. **识别实体和关系**：确定实体、关系类型（1:1、1:N、M:N）
3. **应用设计模式**：针对每种关系选择合适的模式
4. **评估性能**：使用explain()分析查询计划
5. **迭代优化**：根据实际负载调整设计

## 6.2 文档大小与性能

| 文档大小 | 建议 |
|---------|------|
| < 100KB | 最佳范围，性能最优 |
| 100KB ~ 1MB | 可接受，注意网络传输 |
| 1MB ~ 16MB | 需慎重，考虑拆分 |
| > 16MB | 不支持，必须用GridFS |

## 6.3 字段命名建议

- 使用驼峰命名法（camelCase）或下划线命名法（snake_case），团队保持一致
- 字段名尽量简短（每个字段名都占用BSON空间）
- 避免使用`.`和`$`（MongoDB内部使用这些字符）
- 避免以数字开头的字段名
