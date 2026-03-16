# 02-代码Embedding

代码 Embedding 将代码转换为语义向量，使 Agent 能够进行语义级别的代码搜索和理解。本文档详细介绍代码向量化的方法和工具。

## 代码 Embedding 概述

### 什么是代码 Embedding

代码 Embedding 将源代码文本转换为稠密的数学向量，使得语义相似的代码在向量空间中距离更近。

```
┌─────────────────────────────────────────────────────────────────┐
│                    代码 Embedding 流程                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  源代码                    向量化                    向量空间   │
│  ┌──────────┐              ┌──────────┐              ┌────────┐ │
│  │ def      │    ──────→   │ [0.1,    │    ──────→   │   *    │ │
│  │ add(a,b) │              │  -0.2,   │              │  *  *  │ │
│  │   return │              │  0.8,..] │              │ *   *  │ │
│  │   a+b    │              │          │              │   *  * │ │
│  └──────────┘              └──────────┘              └────────┘ │
│                                                              ↓    │
│  相似代码检索                                                 │    │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Query: "sum two numbers"                                    │  │
│  │     ↓                                                      │  │
│  │ 向量化                                                     │  │
│  │     ↓                                                      │  │
│  │ Result: "def add(a, b): return a + b" (similarity: 0.95)   │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 文本 Embedding vs 代码 Embedding

| 特性 | 文本 Embedding | 代码 Embedding |
|------|---------------|----------------|
| 训练数据 | 自然语言 | 代码语料库 |
| 理解能力 | 语义理解 | 语法+语义 |
| 特殊处理 | 无 | AST、Token、标识符 |
| 适用场景 | 文档、对话 | 代码搜索、代码理解 |

## 主流代码 Embedding 模型

### 模型对比

| 模型 | 维度 | 语言 | 特点 |
|------|------|------|------|
| **CodeBERT** | 768 | 多语言 | 微软发布，支持代码补全 |
| **GraphCodeBERT** | 768 | 多语言 | 考虑数据流 |
| **CodeT5** | 768 | 多语言 | T5 架构 |
| **unixcoder** | 768 | 多语言 | 统一表示 |
| **CodeLlama** | 4096 | 多语言 | Meta 大模型 |
| **StarCoder** | 4096 | 多语言 | BigCode 项目 |
| **BAAI/bge-code** | 1024 | 中英 | BAAI 开源 |

## 代码 Embedding 实现

### 使用 Sentence-Transformers

```python
from sentence_transformers import SentenceTransformer
import numpy as np

class CodeEmbedder:
    """代码向量化器"""

    def __init__(self, model_name: str = "microsoft/codebert-base"):
        self.model_name = model_name
        # 使用支持代码的模型
        self.model = SentenceTransformer('microsoft/codebert-base')

    def encode(self, code: str, normalize: bool = True) -> np.ndarray:
        """将代码编码为向量"""

        # CodeBERT 需要特殊处理
        embedding = self.model.encode(
            code,
            convert_to_numpy=True,
            normalize_embeddings=normalize,
            show_progress_bar=False
        )

        return embedding

    def encode_batch(self, codes: list, normalize: bool = True) -> np.ndarray:
        """批量编码"""

        embeddings = self.model.encode(
            codes,
            convert_to_numpy=True,
            normalize_embeddings=normalize,
            show_progress_bar=True,
            batch_size=32
        )

        return embeddings
```

### 使用专门的代码 Embedding 模型

```python
class SpecializedCodeEmbedder:
    """专门的代码向量化器"""

    MODELS = {
        "codebert": {
            "name": "microsoft/codebert-base",
            "dim": 768,
            "max_length": 512,
        },
        "unixcoder": {
            "name": "microsoft/unixcoder-base",
            "dim": 768,
            "max_length": 512,
        },
        "starcoder": {
            "name": "bigcode/starcoder",
            "dim": 4096,
            "max_length": 8192,
        },
        "bge-code": {
            "name": "BAAI/bge-code-zh-v1.5",
            "dim": 1024,
            "max_length": 512,
        }
    }

    def __init__(self, model_type: str = "codebert"):
        config = self.MODELS.get(model_type, self.MODELS["codebert"])
        self.model_name = config["name"]
        self.dim = config["dim"]
        self.max_length = config["max_length"]

        from transformers import AutoModel, AutoTokenizer
        self.tokenizer = AutoTokenizer.from_pretrained(self.model_name)
        self.model = AutoModel.from_pretrained(self.model_name)

    def encode(self, code: str) -> np.ndarray:
        """编码单段代码"""

        # 截断
        code = code[:self.max_length]

        # 编码
        inputs = self.tokenizer(code, return_tensors="pt",
                                truncation=True, max_length=self.max_length)

        with torch.no_grad():
            outputs = self.model(**inputs)

        # 使用 [CLS] token 的输出作为 embedding
        embedding = outputs.last_hidden_state[:, 0, :].numpy()

        # L2 归一化
        embedding = embedding / np.linalg.norm(embedding, axis=1, keepdims=True)

        return embedding[0]
```

### 使用 Code as Text

```python
class TextAsCodeEmbedder:
    """使用通用文本模型处理代码"""

    def __init__(self, model_name: str = "sentence-transformers/all-MiniLM-L6-v2"):
        from sentence_transformers import SentenceTransformer
        self.model = SentenceTransformer(model_name)

    def encode(self, code: str) -> np.ndarray:
        """将代码作为文本编码"""

        # 预处理：压缩空白
        code = self._preprocess(code)

        return self.model.encode(code)

    def _preprocess(self, code: str) -> str:
        """预处理代码"""

        # 1. 移除注释
        lines = []
        for line in code.split('\n'):
            # 移除单行注释
            if '//' in line:
                line = line[:line.index('//')]
            if '#' in line:
                line = line[:line.index('#')]
            lines.append(line)

        code = '\n'.join(lines)

        # 2. 规范化空白
        import re
        code = re.sub(r'\s+', ' ', code)

        return code.strip()
```

## 代码 Embedding 优化

### 1. 添加上下文信息

```python
class ContextualCodeEmbedder:
    """带上下文的代码向量化"""

    def __init__(self, base_embedder):
        self.base = base_embedder

    def encode_with_context(self, code: str, file_path: str = "",
                           language: str = "") -> np.ndarray:
        """添加上下文后编码"""

        # 构建增强文本
        enhanced = self._build_context(code, file_path, language)

        return self.base.encode(enhanced)

    def _build_context(self, code: str, file_path: str, language: str) -> str:
        """构建上下文增强的代码表示"""

        parts = []

        # 添加语言信息
        if language:
            parts.append(f"[{language}]")

        # 添加文件路径
        if file_path:
            path = Path(file_path)
            parts.append(f"File: {path.name}")
            if path.parent.name:
                parts.append(f"Module: {path.parent.name}")

        # 添加代码
        parts.append(code)

        return " ".join(parts)
```

### 2. 处理长代码

```python
class LongCodeEmbedder:
    """处理长代码的向量化器"""

    def __init__(self, embedder, max_chunk_length: int = 512):
        self.embedder = embedder
        self.max_chunk_length = max_chunk_length

    def encode(self, code: str, strategy: str = "sliding") -> np.ndarray:
        """编码长代码"""

        if len(code.split()) <= self.max_chunk_length:
            return self.embedder.encode(code)

        if strategy == "sliding":
            return self._sliding_window(code)
        elif strategy == "summary":
            return self._summary_based(code)
        else:
            return self._first_chunk(code)

    def _sliding_window(self, code: str) -> np.ndarray:
        """滑动窗口 + 聚合"""
        tokens = code.split()
        window_size = self.max_chunk_length
        step = window_size // 2  # 50% 重叠

        embeddings = []

        for i in range(0, len(tokens), step):
            chunk = ' '.join(tokens[i:i + window_size])
            emb = self.embedder.encode(chunk)
            embeddings.append(emb)

            if i + window_size >= len(tokens):
                break

        # 平均池化
        return np.mean(embeddings, axis=0)

    def _summary_based(self, code: str) -> np.ndarray:
        """摘要 + 全文组合"""
        # 这里可以接入 LLM 生成摘要
        # 简化：取开头和结尾
        lines = code.split('\n')
        head = '\n'.join(lines[:50])
        tail = '\n'.join(lines[-50:])

        emb_head = self.embedder.encode(head)
        emb_tail = self.embedder.encode(tail)

        return np.concatenate([emb_head, emb_tail])

    def _first_chunk(self, code: str) -> np.ndarray:
        """只取第一块（可能丢失信息）"""
        tokens = code.split()[:self.max_chunk_length]
        chunk = ' '.join(tokens)
        return self.embedder.encode(chunk)
```

### 3. 多模态 Embedding

```python
class MultiModalCodeEmbedder:
    """多模态代码嵌入 - 结合结构信息"""

    def __init__(self, embedder):
        self.embedder = embedder

    def encode(self, code: str, ast_info: dict = None) -> np.ndarray:
        """结合 AST 信息编码"""

        # 1. 纯文本 embedding
        text_emb = self.embedder.encode(code)

        # 2. 如果有 AST 信息，添加结构 embedding
        if ast_info:
            structure_emb = self._encode_structure(ast_info)
            # 拼接
            combined = np.concatenate([text_emb, structure_emb])
            # 归一化
            combined = combined / np.linalg.norm(combined)
            return combined

        return text_emb

    def _encode_structure(self, ast_info: dict) -> np.ndarray:
        """编码 AST 结构信息"""

        # 简化：根据函数/类数量、深度等特征
        features = []

        # 函数数量
        features.append(ast_info.get("function_count", 0) / 10)

        # 类数量
        features.append(ast_info.get("class_count", 0) / 5)

        # 最大深度
        features.append(ast_info.get("max_depth", 0) / 20)

        # 导入数量
        features.append(ast_info.get("import_count", 0) / 20)

        # 填充到固定维度
        while len(features) < 128:
            features.append(0)

        return np.array(features[:128])
```

## 代码语义搜索

### 向量检索实现

```python
class CodeSemanticSearch:
    """代码语义搜索"""

    def __init__(self, embedder, index_type: str = "faiss"):
        self.embedder = embedder

        if index_type == "faiss":
            import faiss
            self.index = None
            self.dimension = None
        elif index_type == "chroma":
            import chromadb
            self.client = chromadb.Client()
            self.collection = self.client.create_collection("code_search")

        self.code_chunks = []

    def add_chunks(self, chunks: List[dict]):
        """添加代码块"""

        for chunk in chunks:
            content = chunk["content"]
            embedding = self.embedder.encode(content)

            if hasattr(self, 'index'):
                # FAISS
                if self.index is None:
                    self.dimension = len(embedding)
                    self.index = faiss.IndexFlatL2(self.dimension)

                self.index.add(embedding.reshape(1, -1))
            else:
                # Chroma
                self.collection.add(
                    embeddings=[embedding.tolist()],
                    documents=[content],
                    ids=[chunk.get("id", str(len(self.code_chunks)))]
                )

            self.code_chunks.append(chunk)

    def search(self, query: str, top_k: int = 5) -> List[dict]:
        """语义搜索"""

        # 查询向量化
        query_emb = self.embedder.encode(query)

        if hasattr(self, 'index'):
            # FAISS 搜索
            distances, indices = self.index.search(
                query_emb.reshape(1, -1), top_k
            )

            return [
                {
                    **self.code_chunks[idx],
                    "distance": float(distances[0][i])
                }
                for i, idx in enumerate(indices[0])
                if idx >= 0
            ]
        else:
            # Chroma 搜索
            results = self.collection.query(
                query_embeddings=[query_emb.tolist()],
                n_results=top_k
            )

            return [
                {
                    "content": results["documents"][0][i],
                    "distance": results["distances"][0][i]
                }
                for i in range(len(results["documents"][0]))
            ]
```

## 实践建议

### 模型选择

| 场景 | 推荐模型 | 原因 |
|------|----------|------|
| 英文代码搜索 | CodeBERT | 成熟稳定 |
| 多语言代码 | unixcoder | 统一表示 |
| 中文代码 | bge-code-zh | 中文优化 |
| 超长代码 | StarCoder | 长上下文 |
| 综合性能 | bge-code-zh-v1.5 | 效果好 |

### 性能优化

```python
# 1. 使用 GPU 加速
import torch
if torch.cuda.is_available():
    self.model = self.model.to("cuda")

# 2. 批量处理
embeddings = model.encode(codes, batch_size=64)

# 3. 缓存
from functools import lru_cache

@lru_cache(maxsize=10000)
def encode_cached(code_hash: str, code: str):
    return embedder.encode(code)
```

## 下一步

- [01-代码切分策略](./01-代码切分策略.md) - 了解代码切分方法
- [03-AST代码分析](./03-AST代码分析.md) - 了解 AST 分析
- [04-Code-Graph构建](./04-Code-Graph构建.md) - 了解代码图谱