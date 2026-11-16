写对 YAML 只是第一步，真正把它读进程序、做校验、做转换，靠的是解析器和命令行工具。不同语言的库默认行为差别很大，尤其是"安全加载"这一项——用错方法，一个不可信的 YAML 文件就能实例化任意对象。本篇横向对比 SnakeYAML、Jackson、PyYAML、js-yaml，再讲 yq、格式化和 schema 校验工具怎么用。

# 1. 解析器生态总览

| 语言 | 常用库 | 特点 |
|------|--------|------|
| Java | SnakeYAML | 底层解析，Spring Boot 默认依赖 |
| Java | Jackson + YAMLFactory | 与 JSON 统一的数据绑定 |
| Python | PyYAML | 最常用，`safe_load` 与 `load` 区别关键 |
| Python | ruamel.yaml | 保留注释与格式，支持 1.2 |
| JavaScript | js-yaml | Node 生态主流，v4 默认安全 |
| Go | gopkg.in/yaml.v3 | 结构体标签绑定 |
| Rust | serde_yaml | 与 serde 集成（已归档，社区有替代） |

★ **关键认知**：选库先看两点——**是否默认安全加载**、**是否保留注释和原始格式**。前者关乎安全，后者决定它能否用于"改配置文件"而不是"只读数据"。

# 2. Java

## 2.1 SnakeYAML 基础

```java
import org.yaml.snakeyaml.Yaml;
import java.util.Map;

public class Demo {
    public static void main(String[] args) {
        Yaml yaml = new Yaml();
        String text = "name: alice\nage: 18\n";
        Map<String, Object> data = yaml.load(text);
        System.out.println(data.get("name")); // alice
    }
}
```

多文档用 `loadAll`：

```java
Iterable<Object> docs = yaml.loadAll(text);
for (Object doc : docs) {
    System.out.println(doc); // 逐个文档处理
}
```

## 2.2 SnakeYAML 的安全加载

`new Yaml()` 使用默认构造器，在不可信输入上可能通过 `!!` 标签实例化任意类，存在反序列化风险。安全写法是显式指定 `SafeConstructor` 并限制资源：

```java
import org.yaml.snakeyaml.LoaderOptions;
import org.yaml.snakeyaml.Yaml;
import org.yaml.snakeyaml.constructor.SafeConstructor;

LoaderOptions options = new LoaderOptions();
options.setAllowDuplicateKeys(false);   // 重复键直接报错
options.setMaxAliasesForCollections(50); // 限制别名数量，防 YAML 炸弹

Yaml yaml = new Yaml(new SafeConstructor(options));
Object data = yaml.load(untrustedText);  // 只解析成 Map/List/标量，不实例化任意类
```

★ **关键认知**：安全加载的核心是**禁用自定义类型实例化**。`SafeConstructor` 只产出 `Map`、`List`、`String`、数字、布尔、null 这些基础类型，不会根据标签去 `Class.forName`。处理用户上传、外部拉取的 YAML 时，这是必须项。

## 2.3 SnakeYAML 绑定到对象

需要强类型时用 `Constructor` 加目标类：

```java
import org.yaml.snakeyaml.Yaml;
import org.yaml.snakeyaml.constructor.Constructor;

Yaml yaml = new Yaml(new Constructor(ServerConfig.class, new LoaderOptions()));
ServerConfig config = yaml.load(text);
```

```java
public class ServerConfig {
    private String host;
    private int port;
    // getter/setter 省略
}
```

⚠ **关键认知**：绑定到自定义类时，`Constructor` 会按 YAML 的键反射设置字段。只对**你信任的、结构固定**的配置这样做；对不可信输入，先 `SafeConstructor` 读成 Map，再手动映射，避免意外实例化。

## 2.4 Jackson + YAML

Spring Boot 底层用 SnakeYAML，但业务代码常用 Jackson 统一处理 JSON/YAML：

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;

ObjectMapper mapper = new ObjectMapper(new YAMLFactory());
ServerConfig config = mapper.readValue(text, ServerConfig.class);
```

Jackson 的 YAML 模块同样存在标签与多态反序列化风险，处理不可信输入时要关闭默认类型信息（`activateDefaultTyping` 一律不要开）。

# 3. Python

## 3.1 PyYAML 基础

```python
import yaml

with open("config.yaml", encoding="utf-8") as f:
    data = yaml.safe_load(f)

print(data["name"])
```

多文档：

```python
with open("all.yaml", encoding="utf-8") as f:
    docs = list(yaml.safe_load_all(f))
```

## 3.2 `load` 与 `safe_load`

```python
# 危险：可构造任意 Python 对象
yaml.load(text, Loader=yaml.Loader)

# 安全：只产出基础类型
yaml.safe_load(text)
```

PyYAML 5.1 之后，`yaml.load()` 不再有默认 Loader，必须显式传 `Loader`。`yaml.FullLoader` 比 `Loader` 安全一些，但仍不如 `SafeLoader`。

★ **关键认知**：永远用 `safe_load` / `safe_load_all`。除非你 100% 确定输入来源，否则 `Loader`、`UnsafeLoader`、`FullLoader` 都不该出现在业务代码里。

## 3.3 保留注释用 ruamel.yaml

PyYAML 读写会丢注释、打乱格式。需要"改配置文件但保留原样"时用 ruamel.yaml：

```python
from ruamel.yaml import YAML

yaml = YAML()
yaml.preserve_quotes = True

with open("config.yaml", encoding="utf-8") as f:
    data = yaml.load(f)

data["server"]["port"] = 9090

with open("config.yaml", "w", encoding="utf-8") as f:
    yaml.dump(data, f)
```

# 4. JavaScript

## 4.1 js-yaml

```javascript
const yaml = require("js-yaml");
const fs = require("fs");

const data = yaml.load(fs.readFileSync("config.yaml", "utf8"));
console.log(data.name);
```

多文档：

```javascript
const docs = yaml.loadAll(fs.readFileSync("all.yaml", "utf8"));
docs.forEach((doc) => console.log(doc));
```

js-yaml v4 起 `load` 默认不执行任意代码，安全模型比 v3 好。但仍建议：

```javascript
// 限制 schema，避免解析出意外类型
const data = yaml.load(text, { schema: yaml.CORE_SCHEMA });
```

⚠ **关键认知**：js-yaml v3 的 `load` 曾支持 `!!js/function` 等危险类型，v4 已移除。升级到 v4 不只是版本号变化，而是安全边界的收紧；老代码里依赖这些标签的逻辑会直接失效。

# 5. 命令行工具

## 5.1 yq

`yq`（mikefarah 版）是处理 YAML 的瑞士军刀：

```bash
# 格式化输出
yq '.' config.yaml

# 读取字段
yq '.spec.replicas' deploy.yaml

# 修改并写回
yq -i '.spec.replicas = 5' deploy.yaml

# YAML 转 JSON
yq -o=json config.yaml

# JSON 转 YAML
yq -P config.json

# 多文档逐个处理
yq eval-all '.' all.yaml
```

★ **关键认知**：`yq -i` 会**原地修改文件**，且会丢失注释和原有格式（取决于版本）。改生产配置前先 `git diff`，或先输出到新文件再对比。

## 5.2 格式化与 lint

| 工具 | 用途 | 命令 |
|------|------|------|
| `yamllint` | 缩进、重复键、行宽 | `yamllint .` |
| Prettier | 统一格式 | `prettier --write "*.yaml"` |
| `yq` | 重排/规范化 | `yq -i '.' file.yaml` |
| `pre-commit` | 提交前自动检查 | 配置 `.pre-commit-config.yaml` |

yamllint 示例配置：

```yaml
extends: default
rules:
  line-length:
    max: 120
  truthy:
    allowed-values: ["true", "false"]
    check-keys: true
```

⚠ **关键认知**：格式化工具会改变缩进和换行，可能让 `git diff` 出现大段噪声。团队应统一一个格式化器，并在 CI 里加 `--check`，避免有人本地格式化、有人不格式化。

## 5.3 schema 校验

```bash
# Python：按 JSON Schema 校验 YAML
check-jsonschema --schemafile schema.json config.yaml

# K8s 离线校验
kubeconform -strict -summary deploy.yaml

# JS：ajv
npx ajv-cli validate -s schema.json -d config.yaml
```

编辑器侧，VS Code 的 YAML 插件支持文件头声明：

```yaml
# yaml-language-server: $schema=./schema.json
```

# 6. 安全与最佳实践

YAML 的常见攻击面：

| 风险 | 原理 | 防御 |
|------|------|------|
| 任意对象实例化 | `!!` 标签触发构造函数 | 用 SafeLoader / SafeConstructor |
| YAML 炸弹 | 锚点别名指数展开 | 限制别名数量、限制文件大小 |
| 重复键覆盖 | 后值覆盖前值绕过校验 | 开启重复键报错 |
| 深度嵌套 | 递归耗尽栈 | 限制嵌套深度 |
| 超长输入 | 内存耗尽 | 限制输入大小 |

用 ASCII 图看 YAML 炸弹的展开：

```text
a: &a ["x","x","x","x","x","x","x","x","x"]
b: &b [*a,*a,*a,*a,*a,*a,*a,*a,*a]
c: &c [*b,*b,*b,*b,*b,*b,*b,*b,*b]
d: &d [*c,*c,*c,*c,*c,*c,*c,*c,*c]
...
```

每层引用上一层九次，文档体积很小，展开后指数级增长。

★ **关键认知**：安全加载要同时做三件事——**用安全构造器**、**限制别名和嵌套**、**限制输入大小**。只做其中一件都不够：SafeConstructor 挡不住炸弹，大小限制挡不住对象实例化。

# 7. 动手练习

1. 用 PyYAML 的 `safe_load` 读取一份多文档 YAML，统计每个文档的顶层键；再换成 `load` 并构造一个 `!!python/object` 输入，观察 `safe_load` 的报错。

2. 用 SnakeYAML 的 `SafeConstructor` + `LoaderOptions` 读取一份含大量别名的 YAML，把 `setMaxAliasesForCollections` 调小，验证会抛异常。

3. 用 `yq` 把一份 K8s Deployment 的 `replicas` 改成 2 并转成 JSON，然后用 `check-jsonschema` 或 `kubeconform` 校验结果。

# 8. 小结

| 语言/工具 | 推荐做法 | 关键点 |
|-----------|----------|--------|
| Java SnakeYAML | `SafeConstructor` + `LoaderOptions` | 禁用任意类实例化 |
| Java Jackson | `YAMLFactory` | 不开启默认类型信息 |
| Python PyYAML | `safe_load` / `safe_load_all` | 不用 `Loader` |
| Python ruamel | `YAML()` | 保留注释用 |
| JS js-yaml | v4 + `CORE_SCHEMA` | v4 已移除危险标签 |
| yq | `-o=json`、`-P`、`-i` | `-i` 会改写原文件 |
| 校验 | yamllint + JSON Schema | 风格与结构分层 |

一句话原则：**读取不可信 YAML，安全加载是底线；修改配置文件，选保留注释的库**。下一篇做最终对比与总结，把 YAML 与 JSON/XML/TOML 放在一起选型，并给出团队级规范和清单。
