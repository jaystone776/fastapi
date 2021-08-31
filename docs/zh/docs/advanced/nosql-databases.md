# NoSQL（分布式/大数据）数据库

**FastAPI** 还支持与 <abbr title="分布式数据库（大数据），也称为 'Not Only SQL'">NoSQL</abbr> 数据库集成。

本章示例使用 **<a href="https://www.couchbase.com/" class="external-link" target="_blank">Couchbase</a>**，它是一种基于<abbr title="Document here refers to a JSON object (a dict), with keys and values, and those values can also be other JSON objects, arrays (lists), numbers, strings, booleans, etc.">文档</abbr>的 NoSQL 数据库。

除了 **Couchbase** 外，还可以使用以下 NoSQL 数据库：

* **MongoDB**
* **Cassandra**
* **CouchDB**
* **ArangoDB**
* **ElasticSearch** 等

!!! tip "提示"

    **FastAPI** 官方提供了使用 **CouchBase** 的项目生成器，所有内容都包含在 **Docker** 容器中，包括前端等工具：<a href="https://github.com/tiangolo/full-stack-fastapi-couchbase" class="external-link" target="_blank">https://github.com/tiangolo/full-stack-fastapi-couchbase</a>

## 导入 Couchbase 组件

现在，先不考虑其它内容，只关注导入：

```Python hl_lines="3-5"
{!../../../docs_src/nosql_databases/tutorial001.py!}
```

## 定义**文档类型**常量

稍后使用这个常量，在文档中作为固定字段 `type`。

CouchBase 不需要这段代码，但这种方式稍后会有帮助。

```Python hl_lines="9"
{!../../../docs_src/nosql_databases/tutorial001.py!}
```

## 添加获取 `Bucket` 的函数

**Couchbase** 中，Bucket 是一组不同类型的文档。

它们通常都关联到同一个应用。

与关系型数据库对应的概念是**数据库**（专指数据库，不是数据库服务器）。

与 **MongoDB** 对应的是**集合**。

在代码中，`Bucket` 表示与数据库通信的主入口点。

这个工具函数将：

* 关联 **Couchbase** 集群（可能只是一台机器）
    * Set defaults for timeouts.
    * 设置超时的默认值
* 在集群中验证身份
* 获取 `Bucket` 实例
    * 设置超时的默认值
* 返回值

```Python hl_lines="12-21"
{!../../../docs_src/nosql_databases/tutorial001.py!}
```

## 创建 Pydantic 模型

**Couchbase** 文档实际上只是 **JSON 对象**，因此可以用 Pydantic 建模。

### `User` 模型

首先，创建 `User` 模型：

```Python hl_lines="24-28"
{!../../../docs_src/nosql_databases/tutorial001.py!}
```

*路径操作函数*中使用的是这个模型，因此，不要把 `hashed_password` 包含在这个模型里。

### `UserInDB` 模型

接下来，创建 `UserInDB` 模型。

使用这个模型把数据保存到数据库里。

`UserInDB` 模型不是 Pydantic 中 `BaseModel` 的子类，而是 `User` 的子类，因为除了 `User` 所有属性之外，它只增加两个新属性：

```Python hl_lines="31-33"
{!../../../docs_src/nosql_databases/tutorial001.py!}
```

!!! note "笔记"

    注意，`hashed_password` 与 `type` 字段会保存到数据库里。
    
    但它不是通用 `User` 模型的组件，（而是在*路径操作*中返回）。 

## 提取用户

创建实现以下功能的函数：

* 提取用户名
* 根据用户名生成文档 ID
* 使用 ID 提取文档
* 把文档内容放入  `UserInDB` 模型

创建独立于*路径操作函数*的，使用 `username` （或任何其它参数）提取用户的专属函数，可以更轻易地在多个位置复用该函数，并为它添加<abbr title="Automated test, written in code, that checks if another piece of code is working correctly.">单元测试</abbr>。

```Python hl_lines="36-42"
{!../../../docs_src/nosql_databases/tutorial001.py!}
```

### f-字符串

`f"userprofile::{username}"` 是 Python 的 **<a href="https://docs.python.org/3/glossary.html#term-f-string" class="external-link" target="_blank">f-string</a>**。

f-string 把`{}` 里的变量展开并注入到字符串里。

### `dict` 解包

如果您不清楚什么是 `UserInDB(**result.value)`， <a href="https://docs.python.org/3/glossary.html#term-argument" class="external-link" target="_blank">它使用的是 `dict` ”解包“</a> 技术。

它从 `result.value` 中提取字典，并提取每个键和值，然后把键值对以关键字参数的形式传递给 `UserInDB`。

如果**字典**包含以下内容：

```Python
{
    "username": "johndoe",
    "hashed_password": "some_hash",
}
```

它会以如下方式传递给 `UserInDB`：

```Python
UserInDB(username="johndoe", hashed_password="some_hash")
```

## 创建 **FastAPI** 代码

### 创建 `FastAPI` 应用

```Python hl_lines="46"
{!../../../docs_src/nosql_databases/tutorial001.py!}
```

### 创建*路径操作函数*

因为本例中的代码调用了 Couchbase，并且没有使用<a href="https://docs.couchbase.com/python-sdk/2.5/async-programming.html#asyncio-python-3-5" class="external-link" target="_blank">试验性的 Python <code>await</code> 支持</a>，因此，要把函数声明为普通函数（`def`），而不是异步函数（`async`）。

Couchbase 还建议不要在多<abbr title="A sequence of code being executed by the program, while at the same time, or at intervals, there can be others being executed too.">线程</abbr>中使用单个 `Bucket` 对象，因此，可以直接获取 Bucket，并把它传递给工具函数：

```Python hl_lines="49-53"
{!../../../docs_src/nosql_databases/tutorial001.py!}
```

## 小结

只使用标准包，就可以集成第三方 NoSQL 数据库。

同样，这种方式也适用于任何外部工具、系统或 API。

