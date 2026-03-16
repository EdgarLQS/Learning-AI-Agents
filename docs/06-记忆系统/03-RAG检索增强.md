# 03-RAG检索增强

RAG（Retrieval-Augmented Generation，检索增强生成）让 Agent 能够从知识库中获取相关信息。本文档详细介绍 RAG 的核心概念、Chunk 策略、检索流程和重排序技术。

## RAG 概述

### 什么是 RAG

RAG 通过检索外部知识库来增强 LLM 的回答质量，解决模型知识过时或不足的问题。

```
┌─────────────────────────────────────────────────────────────────┐
│                         RAG 流程                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  用户问题 ────────────────────────────┐                         │
│  "如何在 Python 中处理异步编程？"       │                         │
│                                       │                         │
│  ┌────────────────────────────────────┴───────────────────┐    │
│  │                      RAG 流程                           │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │  1. 检索 (Retrieval)                                      │    │
│  │     ↓                                                     │    │
│  │     查询编码 → 向量数据库 → Top-K 相关文档                │    │
│  │                                                         │    │
│  │  2. 增强 (Augmentation)                                   │    │
│  │     ↓                                                     │    │
│  │     上下文构建 → System Prompt + 检索内容 + 用户问题     │    │
│  │                                                         │    │
│  │  3. 生成 (Generation)                                     │    │
│  │     ↓                                                     │    │
│  │     LLM 生成答案                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                       ↓                         │
│  最终回答：基于检索到的文档内容生成答案...                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### RAG vs 微调对比

| 特性 | RAG | 微调 (Fine-tuning) |
|------|-----|-------------------|
| 知识更新 | 实时更新知识库 | 需要重新训练 |
| 成本 | 推理时额外检索 | 训练成本高 |
| 幻觉 | 减少 | 减少 |
| 可解释性 | 可追溯来源 | 黑盒 |
| 适用场景 | 动态知识、专业领域 | 稳定知识、风格学习 |

## 文档处理流程

### 1. 文档加载

```python
from pathlib import Path
from typing import List
import os

class DocumentLoader:
    """文档加载器"""

    SUPPORTED_TYPES = {
        ".txt": "text",
        ".md": "text",
        ".py": "code",
        ".js": "code",
        ".json": "json",
        ".pdf": "pdf",
        ".docx": "docx",
    }

    def load(self, file_path: str) -> str:
        """加载单个文件"""
        path = Path(file_path)
        suffix = path.suffix.lower()

        if suffix not in self.SUPPORTED_TYPES:
            raise ValueError(f"不支持的文件类型: {suffix}")

        if suffix == ".pdf":
            return self._load_pdf(path)
        elif suffix == ".docx":
            return self._load_docx(path)
        else:
            return self._load_text(path)

    def _load_text(self, path: Path) -> str:
        """加载文本文件"""
        with open(path, 'r', encoding='utf-8') as f:
            return f.read()

    def _load_pdf(self, path: Path) -> str:
        """加载 PDF"""
        from pypdf import PdfReader
        reader = PdfReader(path)
        text = ""
        for page in reader.pages:
            text += page.extract_text() + "\n"
        return text

    def _load_docx(self, path: Path) -> str:
        """加载 Word 文档"""
        from docx import Document
        doc = Document(path)
        return "\n".join(p.text for p in doc.paragraphs)

    def load_directory(self, dir_path: str) -> List[dict]:
        """加载目录下的所有文档"""
        documents = []
        for root, dirs, files in os.walk(dir_path):
            for file in files:
                try:
                    path = os.path.join(root, file)
                    content = self.load(path)
                    documents.append({
                        "content": content,
                        "path": path,
                        "filename": file,
                    })
                except Exception as e:
                    print(f"加载失败 {file}: {e}")
        return documents
```

### 2. 文本分块（Chunking）

```python
from typing import List, Callable

class TextChunker:
    """文本分块器"""

    def __init__(self,
                 chunk_size: int = 500,
                 chunk_overlap: int = 50,
                 split_strategy: str = "recursive"):
        self.chunk_size = chunk_size
        self.chunk_overlap = chunk_overlap
        self.split_strategy = split_strategy

    def chunk(self, text: str, metadata: dict = None) -> List[dict]:
        """将文本分块"""

        if self.split_strategy == "recursive":
            chunks = self._recursive_split(text)
        elif self.split_strategy == "fixed":
            chunks = self._fixed_split(text)
        elif self.split_strategy == "semantic":
            chunks = self._semantic_split(text)
        else:
            chunks = self._recursive_split(text)

        # 添加元数据
        return [
            {
                "content": chunk,
                "metadata": metadata or {},
                "chunk_index": i,
            }
            for i, chunk in enumerate(chunks)
        ]

    def _recursive_split(self, text: str) -> List[str]:
        """递归分块 - 保持语义完整"""
        # 先按段落分割
        paragraphs = text.split("\n\n")

        chunks = []
        current_chunk = ""

        for para in paragraphs:
            para = para.strip()
            if not para:
                continue

            # 如果单个段落就超过 chunk_size，进一步分割
            if len(para) > self.chunk_size:
                if current_chunk:
                    chunks.append(current_chunk.strip())
                    current_chunk = ""

                # 按句子分割大段落
                sentences = self._split_sentences(para)
                for sent in sentences:
                    if len(current_chunk) + len(sent) > self.chunk_size:
                        if current_chunk:
                            chunks.append(current_chunk.strip())
                        current_chunk = sent
                    else:
                        current_chunk += " " + sent
            else:
                if len(current_chunk) + len(para) > self.chunk_size:
                    chunks.append(current_chunk.strip())
                    current_chunk = para
                else:
                    current_chunk += "\n\n" + para

        if current_chunk:
            chunks.append(current_chunk.strip())

        # 处理重叠
        return self._add_overlap(chunks)

    def _fixed_split(self, text: str) -> List[str]:
        """固定长度分块"""
        chunks = []
        start = 0

        while start < len(text):
            end = start + self.chunk_size
            chunk = text[start:end]
            chunks.append(chunk)
            start = end - self.chunk_overlap

        return chunks

    def _split_sentences(self, text: str) -> List[str]:
        """按句子分割"""
        import re
        # 简单实现
        sentences = re.split(r'(?<=[。！？])\s+', text)
        return [s for s in sentences if s.strip()]

    def _add_overlap(self, chunks: List[str]) -> List[str]:
        """添加重叠"""
        if not chunks or self.chunk_overlap == 0:
            return chunks

        overlapped = [chunks[0]]
        for i in range(1, len(chunks)):
            prev = chunks[i-1]
            curr = chunks[i]

            # 取前一个 chunk 的最后部分
            overlap_text = prev[-self.chunk_overlap:] if len(prev) > self.chunk_overlap else prev
            overlapped.append(overlap_text + "\n" + curr)

        return overlapped
```

### 3. 代码专用分块

```python
class CodeChunker:
    """代码专用分块器"""

    def __init__(self, chunk_size: int = 1000):
        self.chunk_size = chunk_size

    def chunk(self, code: str, language: str = "python") -> List[dict]:
        """按函数/类分块"""

        if language == "python":
            return self._chunk_python(code)
        elif language == "javascript":
            return self._chunk_javascript(code)
        else:
            return self._chunk_by_structure(code)

    def _chunk_python(self, code: str) -> List[dict]:
        """Python 代码分块"""
        import re

        # 匹配函数和类定义
        pattern = r'^(\s*(?:async\s+)?def\s+\w+|class\s+\w+)'

        matches = []
        for i, line in enumerate(code.split('\n')):
            if re.match(pattern, line):
                matches.append(i)

        # 边界处理
        boundaries = matches + [len(code.split('\n'))]

        chunks = []
        for i in range(len(boundaries) - 1):
            start = boundaries[i]
            end = boundaries[i + 1]

            # 获取代码片段
            lines = code.split('\n')[start:end]
            chunk_code = '\n'.join(lines)

            if len(chunk_code) > 100:  # 忽略太小的片段
                chunks.append({
                    "content": chunk_code,
                    "chunk_type": "function" if "def " in lines[0] else "class",
                    "line_start": start + 1,
                    "line_end": end,
                })

        return chunks

    def _chunk_javascript(self, code: str) -> List[dict]:
        """JavaScript 代码分块"""
        import re

        pattern = r'^\s*(?:function\s+\w+|const\s+\w+\s*=|async\s+function)'
        matches = []

        for i, line in enumerate(code.split('\n')):
            if re.match(pattern, line):
                matches.append(i)

        # 类似处理...
        return []
```

## 向量化与存储

### Embedding 模型选择

```python
class EmbeddingModelSelector:
    """Embedding 模型选择"""

    MODELS = {
        "openai": {
            "text-embedding-3-small": {"dim": 1536, "cost": "low"},
            "text-embedding-3-large": {"dim": 3072, "cost": "medium"},
            "text-embedding-ada-002": {"dim": 1536, "cost": "medium"},
        },
        "open-source": {
            "sentence-transformers/all-MiniLM-L6-v2": {"dim": 384, "cost": "free"},
            "BAAI/bge-large-zh-v1.5": {"dim": 1024, "cost": "free"},
            "BAAI/bge-base-zh-v1.5": {"dim": 768, "cost": "free"},
        }
    }

    @staticmethod
    def select(task: str, language: str = "en") -> str:
        """选择合适的模型"""

        if task == "code":
            # 代码检索
            return "microsoft/codebert-base"

        if language == "zh":
            # 中文
            return "BAAI/bge-base-zh-v1.5"

        # 通用英文
        return "text-embedding-3-small"
```

### 向量化处理

```python
from sentence_transformers import SentenceTransformer
import numpy as np

class VectorStore:
    """向量存储"""

    def __init__(self, model_name: str, index_type: str = "chroma"):
        self.model = SentenceTransformer(model_name)

        if index_type == "chroma":
            import chromadb
            self.client = chromadb.Client()
            self.collection = self.client.create_collection("documents")
        elif index_type == "faiss":
            import faiss
            self.index = None
            self.documents = []
            self.dimension = None

    def add_documents(self, chunks: List[dict]):
        """添加文档到向量库"""

        texts = [chunk["content"] for chunk in chunks]
        ids = [f"doc_{i}" for i in range(len(chunks))]

        # 生成 embedding
        embeddings = self.model.encode(texts).tolist()

        # 存储
        if hasattr(self, 'collection'):
            self.collection.add(
                embeddings=embeddings,
                documents=texts,
                ids=ids,
                metadatas=[c.get("metadata", {}) for c in chunks]
            )
        else:
            # FAISS
            if self.index is None:
                self.dimension = len(embeddings[0])
                import faiss
                self.index = faiss.IndexFlatL2(self.dimension)

            self.index.add(np.array(embeddings).astype('float32'))
            self.documents.extend(chunks)

    def search(self, query: str, top_k: int = 5) -> List[dict]:
        """相似度搜索"""

        # 查询向量化
        query_embedding = self.model.encode([query]).tolist()

        if hasattr(self, 'collection'):
            results = self.collection.query(
                query_embeddings=query_embedding,
                n_results=top_k
            )
            return self._format_chroma_results(results)
        else:
            # FAISS
            distances, indices = self.index.search(
                np.array(query_embedding).astype('float32'), top_k
            )
            return self._format_faiss_results(indices, distances)

    def _format_chroma_results(self, results) -> List[dict]:
        """格式化 Chroma 结果"""
        formatted = []
        for i in range(len(results["documents"][0])):
            formatted.append({
                "content": results["documents"][0][i],
                "metadata": results["metadatas"][0][i],
                "distance": results["distances"][0][i],
            })
        return formatted

    def _format_faiss_results(self, indices, distances) -> List[dict]:
        """格式化 FAISS 结果"""
        results = []
        for i, idx in enumerate(indices[0]):
            if idx >= 0:
                results.append({
                    "content": self.documents[idx]["content"],
                    "metadata": self.documents[idx].get("metadata", {}),
                    "distance": distances[0][i],
                })
        return results
```

## 检索增强流程

### 完整 RAG 流程

```python
class RAGPipeline:
    """完整的 RAG 流程"""

    def __init__(self,
                 embedding_model: str,
                 vector_store: VectorStore,
                 llm_client,
                 chunk_size: int = 500):
        self.vector_store = vector_store
        self.llm = llm_client
        self.chunker = TextChunker(chunk_size=chunk_size)

    def index_documents(self, documents: List[dict]):
        """索引文档"""

        all_chunks = []
        for doc in documents:
            # 分块
            chunks = self.chunker.chunk(
                doc["content"],
                metadata={"source": doc.get("path", "unknown")}
            )
            all_chunks.extend(chunks)

        # 存储到向量库
        self.vector_store.add_documents(all_chunks)
        print(f"已索引 {len(all_chunks)} 个文本块")

    def retrieve(self, query: str, top_k: int = 5) -> List[dict]:
        """检索相关文档"""
        return self.vector_store.search(query, top_k)

    def generate(self, query: str, retrieved_docs: List[dict],
                 system_prompt: str = None) -> str:
        """生成回答"""

        # 构建上下文
        context = self._build_context(retrieved_docs)

        # 构建 prompt
        prompt = f"""基于以下参考资料回答问题。如果资料不足以回答，请说明无法确定。

参考资料:
{context}

问题: {query}

回答:"""

        if system_prompt:
            system_prompt += "\n" + "你是一个专业的 AI 助手，基于给定的参考资料回答问题。"

        # 调用 LLM
        response = self.llm.chat.completions.create(
            model="gpt-4",
            messages=[
                {"role": "system", "content": system_prompt or "你是一个专业的 AI 助手。"},
                {"role": "user", "content": prompt}
            ],
            temperature=0.7
        )

        return {
            "answer": response.choices[0].message.content,
            "sources": [doc["metadata"].get("source", "unknown")
                       for doc in retrieved_docs]
        }

    def _build_context(self, docs: List[dict]) -> str:
        """构建检索上下文"""
        context_parts = []
        for i, doc in enumerate(docs, 1):
            source = doc["metadata"].get("source", "unknown")
            content = doc["content"]
            context_parts.append(f"[{i}] 来源: {source}\n{content}")
        return "\n\n".join(context_parts)

    def query(self, question: str, top_k: int = 5) -> dict:
        """完整查询流程"""
        # 1. 检索
        retrieved = self.retrieve(question, top_k)

        # 2. 生成
        result = self.generate(question, retrieved)

        return result
```

## 重排序技术

### 重排序的作用

初步检索返回的结果可能包含不够相关的内容，重排序（Reranking）可以进一步提升结果质量。

```python
class Reranker:
    """文档重排序"""

    def __init__(self, model_name: str = "cross-encoder/ms-marco-MiniLM-L-6-v2"):
        from sentence_transformers import CrossEncoder
        self.model = CrossEncoder(model_name)

    def rerank(self, query: str, documents: List[dict],
               top_k: int = 3) -> List[dict]:
        """对文档重排序"""

        # 准备文档对
        doc_pairs = [
            (query, doc["content"]) for doc in documents
        ]

        # 计算相关性分数
        scores = self.model.predict(doc_pairs)

        # 按分数排序
        scored_docs = [
            {**doc, "rerank_score": float(score)}
            for doc, score in zip(documents, scores)
        ]
        scored_docs.sort(key=lambda x: x["rerank_score"], reverse=True)

        return scored_docs[:top_k]
```

## 高级 RAG 技术

### 1. 混合检索

```python
class HybridRAG:
    """混合检索 - 向量 + 关键词"""

    def __init__(self, vector_store, keyword_index):
        self.vector_store = vector_store
        self.keyword_index = keyword_index

    def search(self, query: str, alpha: float = 0.5, top_k: int = 5):
        """混合搜索"""

        # 向量搜索
        vector_results = self.vector_store.search(query, top_k * 2)

        # 关键词搜索
        keyword_results = self.keyword_index.search(query, top_k * 2)

        # RRF 融合
        results = self._reciprocal_rank_fusion(
            vector_results, keyword_results, alpha, top_k
        )

        return results

    def _reciprocal_rank_fusion(self, vec_res, kw_res, alpha, top_k):
        """RRF 融合"""
        scores = {}

        for rank, doc in enumerate(vec_res):
            doc_id = doc.get("id", rank)
            scores[doc_id] = scores.get(doc_id, 0) + (1 - alpha) / (rank + 1)

        for rank, doc in enumerate(kw_res):
            doc_id = doc.get("id", rank)
            scores[doc_id] = scores.get(doc_id, 0) + alpha / (rank + 1)

        sorted_scores = sorted(scores.items(), key=lambda x: x[1], reverse=True)
        return [item[0] for item in sorted_scores[:top_k]]
```

### 2. 查询扩展

```python
class QueryExpander:
    """查询扩展"""

    def __init__(self, llm_client):
        self.llm = llm_client

    async def expand(self, query: str) -> List[str]:
        """扩展查询"""

        prompt = f"""为以下查询生成3个不同的表达方式，要求:
1. 保持原意
2. 使用不同的关键词
3. 可以包含同义词

原始查询: {query}

扩展后:
"""

        response = await self.llm.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": prompt}]
        )

        # 解析结果
        expansions = response.choices[0].message.content.strip().split("\n")
        expansions = [q.strip().lstrip("123.- ") for q in expansions if q.strip()]

        return [query] + expansions
```

## 下一步

- [02-向量数据库](./02-向量数据库.md) - 了解向量存储技术
- [04-记忆架构设计](./04-记忆架构设计.md) - 设计完整记忆系统