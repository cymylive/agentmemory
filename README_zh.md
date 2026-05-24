# agentmemory

<div align="center">

[![Lint and Test](https://github.com/AutonomousResearchGroup/agentmemory/actions/workflows/test.yml/badge.svg)](https://github.com/AutonomousResearchGroup/agentmemory/actions/workflows/test.yml)
[![PyPI version](https://badge.fury.io/py/agentmemory.svg)](https://badge.fury.io/py/agentmemory)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://github.com/AutonomousResearchGroup/easycompletion/blob/main/LICENSE)

**AI Agent 的轻量级记忆管理库** — 语义搜索、知识图谱、事件记录、聚类分析

</div>

---

## 目录

- [简介](#简介)
- [安装](#安装)
- [快速开始](#快速开始)
- [部署配置](#部署配置)
  - [ChromaDB（本地模式）](#chromadb本地模式)
  - [PostgreSQL（生产部署）](#postgresql生产部署)
- [API 参考](#api-参考)
  - [记忆 CRUD](#1-记忆-crud)
  - [语义搜索](#2-语义搜索)
  - [事件系统](#3-事件系统)
  - [聚类分析](#4-聚类分析)
  - [数据持久化](#5-数据持久化)
  - [模型管理](#6-模型管理)
  - [辅助工具](#7-辅助工具)
- [常见问题](#常见问题)
- [贡献指南](#贡献指南)

---

## 简介

`agentmemory` 是为 AI Agent 设计的记忆管理库，提供以下核心能力：

- **向量化存储** — 自动将文本转为向量嵌入，支持语义搜索
- **双后端支持** — 本地 ChromaDB 与生产级 PostgreSQL + pgvector
- **事件系统** — 基于纪元（epoch）的顺序事件记录
- **聚类分析** — 内建 DBScan 密度聚类算法
- **数据持久化** — JSON 导入/导出，方便备份与迁移

---

## 安装

```bash
pip install agentmemory
```

---

## 快速开始

```python
from agentmemory import create_memory, search_memory

# 创建一条记忆
create_memory(
    "conversation",
    "I can't do that, Dave.",
    metadata={"speaker": "HAL", "some_other_key": "some value"}
)

# 语义搜索记忆
memories = search_memory("conversation", "Dave")

print(memories)
# 输出：
# [{
#     "id": 0,
#     "document": "I can't do that, Dave.",
#     "metadata": {"speaker": "HAL", ...},
#     "embedding": [0.001, 0.002, ...]  # 可选
# }]
```

---

## 部署配置

### ChromaDB（本地模式）

默认使用 ChromaDB 本地持久化存储，数据保存在 `./memory` 目录：

```python
from agentmemory import create_memory

# 开箱即用，无需额外配置
create_memory("test", "hello world")
```

通过环境变量自定义存储路径：

```bash
export STORAGE_PATH="/path/to/memory"
```

### PostgreSQL（生产部署）

适用于大规模生产环境，使用 pgvector 扩展进行向量搜索：

```bash
export CLIENT_TYPE=POSTGRES
export POSTGRES_CONNECTION_STRING="postgres://user:password@host:5432/dbname"
export POSTGRES_MODEL_NAME="all-MiniLM-L6-v2"  # 可选，默认值
export EMBEDDING_WIDTH=384                       # 可选，默认值
```

代码中使用：

```python
import os
os.environ["CLIENT_TYPE"] = "POSTGRES"
os.environ["POSTGRES_CONNECTION_STRING"] = "postgres://..."

from agentmemory import create_memory
create_memory("conversation", "Hello, world!")
```

> 推荐使用 [Supabase](https://supabase.com/) 部署 PostgreSQL + pgvector，
> 参考教程：[OpenAI Embeddings & Postgres Vector](https://supabase.com/blog/openai-embeddings-postgres-vector)

---

## API 参考

### 1. 记忆 CRUD

#### `create_memory(category, text, metadata={}, embedding=None, id=None)`

创建一条新记忆。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `category` | str | — | 分类/集合名称 |
| `text` | str | — | 记忆内容文本 |
| `metadata` | dict | `{}` | 元数据，自动添加 `created_at`/`updated_at` 时间戳 |
| `embedding` | list | `None` | 预计算的向量嵌入（不传则自动生成） |
| `id` | str/int | `None` | 自定义 ID（不传则自动生成） |

**返回：** `str` 或 `int` — 新记忆的 ID

```python
create_memory("books", "百年孤独", metadata={"author": "马尔克斯", "year": 1967})
# 返回：0
```

---

#### `create_unique_memory(category, content, metadata={}, similarity=0.95)`

仅在无高度相似内容时创建新记忆，用于去重。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `category` | str | — | 分类名称 |
| `content` | str | — | 记忆内容 |
| `metadata` | dict | `{}` | 元数据 |
| `similarity` | float | `0.95` | 相似度阈值（0~1），越高越严格 |

- 未找到相似内容：`metadata["novel"] = "True"`，创建新记忆
- 找到相似内容：`metadata["novel"] = "False"`，记录关联 ID

```python
create_unique_memory("books", "百年孤独", similarity=0.9)
```

---

#### `get_memory(category, id, include_embeddings=True)`

按 ID 获取单条记忆。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `category` | str | — | 分类名称 |
| `id` | str/int | — | 记忆 ID |
| `include_embeddings` | bool | `True` | 是否包含向量 |

**返回：** `dict` 或 `None`

```python
memory = get_memory("books", 0)
print(memory["document"])  # 百年孤独
```

---

#### `get_memories(category, sort_order="desc", contains_text=None, filter_metadata=None, n_results=20, include_embeddings=True, novel=False)`

获取分类下的多条记忆，支持过滤和排序。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `category` | str | — | 分类名称 |
| `sort_order` | str | `"desc"` | 排序方向：`"asc"` 或 `"desc"` |
| `contains_text` | str | `None` | 文档必须包含的文本 |
| `filter_metadata` | dict | `None` | 元数据精确匹配 |
| `n_results` | int | `20` | 返回条数 |
| `include_embeddings` | bool | `True` | 是否包含向量 |
| `novel` | bool | `False` | 是否只返回新记忆 |

**返回：** `list[dict]`

```python
memories = get_memories("books", sort_order="asc", n_results=5)
```

---

#### `update_memory(category, id, text=None, metadata=None, embedding=None)`

更新记忆内容或元数据。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `category` | str | — | 分类名称 |
| `id` | str/int | — | 要更新的记忆 ID |
| `text` | str | `None` | 新文本 |
| `metadata` | dict | `None` | 新元数据 |
| `embedding` | list | `None` | 新向量 |

```python
update_memory("books", 0, text="霍乱时期的爱情", metadata={"author": "马尔克斯"})
```

---

#### `delete_memory(category, id)`

按 ID 删除单条记忆。

```python
delete_memory("books", 0)
```

---

#### `delete_memories(category, document=None, metadata=None)`

按文档内容或元数据批量删除记忆。

```python
# 删除作者为马尔克斯的所有记忆
delete_memories("books", metadata={"author": "马尔克斯"})

# 删除包含"百年"的记忆
delete_memories("books", document="百年")
```

---

#### `delete_similar_memories(category, content, similarity_threshold=0.95)`

查找并删除与指定内容相似的记忆。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `category` | str | — | 分类名称 |
| `content` | str | — | 匹配内容 |
| `similarity_threshold` | float | `0.95` | 相似度阈值 |

**返回：** `bool` — 是否删除了任何记忆

```python
delete_similar_memories("books", "百年孤独", similarity_threshold=0.9)
```

---

#### `count_memories(category, novel=False)`

统计分类中的记忆数量。

```python
count = count_memories("books")
print(f"共有 {count} 本书")
```

---

#### `memory_exists(category, id, includes_metadata=None)`

检查指定记忆是否存在。

```python
if memory_exists("books", 0):
    print("记忆存在")
```

---

#### `wipe_category(category)`

删除整个分类及其所有记忆。

```python
wipe_category("books")
```

---

#### `wipe_all_memories()`

删除所有分类下的全部记忆。

```python
wipe_all_memories()
```

---

### 2. 语义搜索

#### `search_memory(category, search_text, n_results=5, filter_metadata=None, contains_text=None, include_embeddings=True, include_distances=True, max_distance=None, min_distance=None, novel=False)`

基于语义相似度的记忆搜索。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `category` | str | — | 分类名称 |
| `search_text` | str | — | 搜索关键词 |
| `n_results` | int | `5` | 返回结果数量 |
| `filter_metadata` | dict | `None` | 元数据精确匹配 |
| `contains_text` | str | `None` | 文档必须包含的文本 |
| `include_embeddings` | bool | `True` | 是否返回向量 |
| `include_distances` | bool | `True` | 是否返回距离分数 |
| `max_distance` | float | `None` | 最大距离过滤（0=完全相同，1=完全不同） |
| `min_distance` | float | `None` | 最小距离过滤 |
| `novel` | bool | `False` | 是否只返回新记忆 |

**返回：** `list[dict]` — 每项包含 `{id, document, metadata, embedding?, distance?}`

```python
# 搜索关于"人工智能"的记忆
results = search_memory("conversation", "人工智能", n_results=3)

# 带元数据过滤
results = search_memory(
    "conversation",
    "人工智能",
    filter_metadata={"speaker": "HAL"},
    max_distance=0.5
)
```

> 注意：距离过滤在查询后应用，可能会减少结果数量，这是 ChromaDB 的当前限制。

---

### 3. 事件系统

事件系统提供基于纪元（epoch）的顺序事件记录，适用于对话轮次、循环迭代等场景。

#### `reset_epoch()`

重置纪元为 1。

```python
reset_epoch()
```

---

#### `set_epoch(epoch)`

设置指定纪元值。

```python
set_epoch(5)
```

---

#### `increment_epoch()`

纪元值 +1。

**返回：** `int` — 新的纪元值

```python
current = increment_epoch()
print(f"当前纪元: {current}")
```

---

#### `get_epoch()`

获取当前纪元值。

**返回：** `int`

```python
epoch = get_epoch()
```

---

#### `create_event(text, metadata={}, embedding=None)`

创建一条纪元事件，自动注入当前 epoch 到元数据中。

```python
create_event("用户询问天气", metadata={"intent": "weather"})
```

---

#### `get_events(epoch=None, n_results=10, filter_metadata=None)`

获取事件列表，支持按纪元筛选。

```python
# 获取所有事件
events = get_events()

# 获取特定纪元的事件
events = get_events(epoch=1)
```

---

### 4. 聚类分析

#### `cluster(epsilon, min_samples, category, filter_metadata=None, novel=False)`

对指定分类的记忆执行 DBScan 密度聚类。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `epsilon` | float | — | 邻域半径，决定两个样本被视为相邻的最大距离 |
| `min_samples` | int | — | 核心点的最小邻域样本数 |
| `category` | str | — | 要聚类的分类 |
| `filter_metadata` | dict | `None` | 聚类前的元数据过滤 |
| `novel` | bool | `False` | 是否只处理新记忆 |

聚类结果直接写入每个记忆的元数据中的 `"cluster"` 字段：
- 噪声点标记为 `"noise"`
- 正常簇标记为簇 ID（整数）

```python
from agentmemory import cluster

cluster(
    epsilon=0.3,
    min_samples=3,
    category="conversation",
    filter_metadata={"speaker": "HAL"}
)
```

> DBScan 算法参考：[原始论文](https://www.aaai.org/Papers/KDD/1996/KDD96-037.pdf)

---

### 5. 数据持久化

#### `export_memory_to_json(include_embeddings=True)`

将所有记忆导出为字典。

**返回：** `dict` — `{collection_name: [memory_dict, ...], ...}`

```python
data = export_memory_to_json(include_embeddings=False)
```

---

#### `export_memory_to_file(path="./memory.json", include_embeddings=True)`

将所有记忆导出到 JSON 文件。

```python
export_memory_to_file(path="./backup.json")
```

---

#### `import_json_to_memory(data, replace=True)`

从字典导入记忆到数据库。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `data` | dict | — | `{category: [memory_dict, ...]}` 格式的数据 |
| `replace` | bool | `True` | 是否清空现有记忆后再导入 |

```python
import_json_to_memory(data, replace=False)
```

---

#### `import_file_to_memory(path="./memory.json", replace=True)`

从 JSON 文件导入记忆。

```python
import_file_to_memory(path="./backup.json", replace=True)
```

---

### 6. 模型管理

#### `check_model(model_name="all-MiniLM-L6-v2", model_path=None)`

检查并自动下载 ONNX 嵌入模型。

- 默认模型路径：`~/.cache/onnx_models`
- 下载来源：`https://chroma-onnx-models.s3.amazonaws.com/`

```python
from agentmemory import check_model
model_path = check_model()
```

---

#### `infer_embeddings(documents, model_path, batch_size=32)`

批量将文本转换为向量嵌入。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `documents` | list[str] | — | 文本列表 |
| `model_path` | str | — | `check_model()` 返回的模型路径 |
| `batch_size` | int | `32` | 批处理大小 |

```python
embeddings = infer_embeddings(["你好", "世界"], model_path)
```

---

### 7. 辅助工具

#### `get_client(client_type=None)`

获取后端客户端实例（单例模式）。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `client_type` | str | `None` | `"CHROMA"` 或 `"POSTGRES"`，默认从环境变量读取 |

```python
from agentmemory import get_client
client = get_client("CHROMA")
```

---

#### `chroma_collection_to_list(collection)`

将 ChromaDB 返回的字典格式转换为统一的对象列表。

```python
from agentmemory import chroma_collection_to_list
items = chroma_collection_to_list(chroma_result)
```

---

#### `list_to_chroma_collection(list)`

将对象列表转换回 ChromaDB 字典格式（`chroma_collection_to_list` 的逆操作）。

```python
from agentmemory import list_to_chroma_collection
chroma_dict = list_to_chroma_collection(items)
```

---

## 调试模式

有两种方式启用调试日志：

```python
# 方式一：传参
create_memory("test", "hello", debug=True)

# 方式二：环境变量
export DEBUG=True
```

调试模式下会输出详细的操作日志，向量值会被截断为 `[...]` 以便阅读。

---

## 常见问题

**Q: 如何切换存储后端？**

设置环境变量 `CLIENT_TYPE=POSTGRES` 并配置 `POSTGRES_CONNECTION_STRING`。

**Q: ChromaDB 数据存在哪里？**

默认存储在 `./memory` 目录，可通过 `STORAGE_PATH` 环境变量自定义。

**Q: 支持哪些元数据类型？**

ChromaDB 限制 metadata 值只能为字符串、数字、布尔。`dict`/`list`/`bool` 类型会自动转为字符串。

**Q: 如何备份记忆数据？**

使用 `export_memory_to_file()` 导出为 JSON，`import_file_to_memory()` 恢复。

---

## 贡献指南

欢迎贡献代码！本项目追求简洁和易用性，请保持代码风格一致。

- 提交 PR 前请确保测试通过
- 保持低依赖、高可读性
- 详细说明请参阅英文版 [README.md](README.md)

---

<div align="center">

**agentmemory** — 让 AI Agent 拥有持久记忆

</div>
