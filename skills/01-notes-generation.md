# Skill 1 — Notes Generation

## English

**Purpose**: turn a single module's slice of the exam guide (a set of AWS services and sub-topics) into a complete, structured set of study notes — the kind that could stand alone as a reference, not just a summary.

**Input**: the module's scope as defined by the daily/monthly plan (e.g. "Module 6: Databases — RDS, Aurora, ElastiCache, DynamoDB"), plus the relevant slice of the official exam guide.

**Process**: 
- For each module, generate notes covering the core concepts, the comparisons that matter for the exam (e.g. Multi-AZ vs. Read Replica, DAX vs. ElastiCache), concrete numbers and limits worth memorizing, common exam traps, and a scenario-to-answer quick-reference table at the end of major sections. 
- After generation, search AWS official documents and validate relevant content, especially numbers in notes (such as memory size, cost, number of allowed instances)

**Output**: the per-module notes that became this repo's source content in Notion, later reorganized into the bilingual `zh/`/`en/` files here.

**Design notes**: notes are written densely — tables and bullet comparisons over prose. Priority markers (⭐/⭐⭐/⭐⭐⭐) were used throughout to flag how often a topic tends to show up on the actual exam, based on patterns across practice questions and community-reported exam experiences.

---

## 中文

**用途**：把一个模块在考纲里对应的范围（一组 AWS 服务和子主题），转化成一份完整、成体系的学习笔记——目标是能独立当参考资料用，而不只是一份摘要。

**输入**：由每日/月度计划定义的模块范围（例如"Module 6：数据库——RDS、Aurora、ElastiCache、DynamoDB"），加上官方考纲里对应的部分。

**处理过程**：
- 针对每个模块，生成覆盖核心概念的笔记，包括对考试真正有用的对比（如 Multi-AZ vs Read Replica、DAX vs ElastiCache）、值得背下来的具体数字和限制、常见的考试陷阱，以及在主要小节末尾给出"场景 → 答案"速查表。
- 在生成结束后，搜索aws对应的官方文档并查证相关内容，注意需要着重考察内容里的数字（如 memory size, cost, number of allowed instances）

**输出**：这份仓库在 Notion 里的原始来源内容，就是这样逐模块生成的，后来被整理进这份仓库里的双语 `zh/`/`en/` 文件。

**设计说明**：笔记写得比较密——多用表格和对比式要点，少用长段落。依据练习题里的规律以及社区里反馈的考试经验，全文用优先级标记（⭐/⭐⭐/⭐⭐⭐）标注一个知识点在真实考试里出现的频率。