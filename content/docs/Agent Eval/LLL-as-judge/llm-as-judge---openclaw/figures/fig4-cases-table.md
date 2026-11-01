# 图4：主流工业方案对比

| 方案 | 维护方 | 评估范式 | 规模 | 公开性 | 特色 |
|------|--------|----------|------|--------|------|
| **Chatbot Arena** | LMSYS (UC Berkeley) | Pairwise + ELO | 百万级投票 | ✅ 开源排行榜 | 众包 + LLM 混合，影响力最大的公开基准 |
| **Constitutional AI** | Anthropic | Self-Critique + Pointwise | 内部规模 | ⚠️ 论文公开 | Judge 即训练数据生成器，闭环优化 |
| **Gemini Eval** | Google | Multi-dimension Pointwise | 大规模内部 | ❌ 不公开 | 工业级管线，与人工高度对齐 |
| **OpenCompass** | OpenCompass 社区 | 全范式 | 多基准集成 | ✅ 开源 | 中文场景覆盖全，支持多模态 |
| **HELM** | Stanford | 多维度标准化 | 大规模 | ✅ 开源 | 透明化评测方法论标杆 |
| **RAGAS** | RAGAS 社区 | Reference-free Pointwise | 轻量级 | ✅ 开源 | 专注 RAG 场景的质量评估 |
| **AlpacaEval** | Tatsu Lab | Pairwise (自动) | 轻量级 | ✅ 开源 | 低成本，与 Arena 相关性 ~0.97 |
| **Prometheus** | KAIST | Pointwise (专用模型) | 50K 训练样本 | ✅ 开源模型 | 开源可微调的 Judge 模型 |
