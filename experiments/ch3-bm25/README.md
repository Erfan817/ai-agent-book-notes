# 实验 3-5 复现材料：BM25 在"数据结构与算法"语料上的表现

配套笔记：[第 3 章](../../notes/chapter3.md)。语料和标注是我自己写的，用来验证书中"稀疏检索精确匹配强、读不懂同义词"的结论在技术问答场景下是否成立。

## 文件

| 文件 | 说明 |
| --- | --- |
| `corpus.json` | 10 篇数据结构/算法文档（英文，与书配套引擎的英文分词器一致） |
| `labels.json` | 9 条查询标注：5 条精确术语、3 条纯同义改写、1 条复杂度记号 |
| `eval_bm25.json` | `--eval` 的完整输出（每条查询的召回排序与指标） |

## 复跑

需要原书仓库的配套代码 `chapter3/sparse-embedding/`（`cli.py` 只用标准库，无需模型与 API Key）：

```bash
cd chapter3/sparse-embedding
python cli.py --eval \
  -c  <本目录>/corpus.json \
  --labels <本目录>/labels.json \
  -k 5 -o <本目录>/eval_bm25.json
```

查看逐词贡献（复现笔记里的两个"翻车现场"）：

```bash
python cli.py -c <本目录>/corpus.json -q "the fastest way to travel between two cities on a road map" --explain -k 3
python cli.py -c <本目录>/corpus.json -q "O(n log n)" --explain -k 5
python cli.py -c <本目录>/corpus.json -q "fastest travel cities road map"      # 对照组：零命中
```

## 实测结果（k1=1.5, b=0.75, avgdl=22.90, k=5）

| 查询 | 类型 | recall@5 | MRR |
| --- | --- | :--: | :--: |
| red-black tree | 精确术语 | 1.00 | 1.00 |
| Dijkstra priority queue | 精确术语 | 1.00 | 1.00 |
| negative cycles | 精确术语 | 1.00 | 1.00 |
| amortized append | 精确术语 | 1.00 | 1.00 |
| topological sort directed acyclic graph | 精确术语 | 1.00 | 1.00 |
| keeping a phone book so names are found almost instantly | 同义改写 | 0.00 | 0.00 |
| a strategy that costs the same no matter how bad the input looks | 同义改写 | 0.00 | 0.00 |
| the fastest way to travel between two cities on a road map | 同义改写 | 1.00 | 0.50 |
| O(n log n) | 复杂度记号 | 1.00 | 0.50 |
| **宏平均** | | **0.778** | **0.667** |

两条值得记录的观察：

1. 第三条"同义改写"查询的 recall=1.00 是假象：真正支撑排序的是虚词 `two`（IDF=1.8458）和 `to`（IDF=0.3677），意图词 `fastest`/`road`/`map` 一个都没命中；去掉虚词后召回为 0。
2. `O(n log n)` 被分词为 `o` / `n` / `log` / `n`，单字母在语料中出现频率过高，IDF 被压到下限 0（`max(raw_idf, 1e-6)`），排序实际只由 `log` 决定，于是 `O(log n)` 的红黑树排在了快排、归并前面。
