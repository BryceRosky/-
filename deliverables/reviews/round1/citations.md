# 第一轮独立引用审计（Reviewer：参考文献与引文）

核验对象：`/workspace/output/罗思琪_论文开题报告_内容源稿.md`（41 条参考文献）
核验日期：2026-10-09
本轮只审查，不编辑正文。

## 总体结论

- 41 个编号连续，正文实际使用了 `[1]`—`[41]` 全部编号，无漏号、越界号或“只列不引”。
- 没有发现完全捏造的论文身份。更准确的表述应是：**40 条具有公开论文集、arXiv 或作者官方项目记录；[32] 为用户提供的未公开作者稿，稿件本身可核验，但 TPAMI 在审状态只能由所附稿件/答辩材料支持。** 因而不宜笼统宣称 41 条都能由公开出版元数据确认。
- 发现 **1 处确定的作者表错误**（[37] 漏列第二作者且未用 `et al.`），以及 **3 处需修改或降级的证据表达**（[24] 的图像基线归属、[32] 的在审状态进入书目、[33] 的奖励微调表述）。
- 当前稿已正确避免三类常见传播错误：GenEval 的错误页码、LLaDA 的无依据 ICML 2025 归属，以及推理时扩展论文的预印本旧题名与 CVPR 正式题名混用。这三处不要在后续迭代中改回旧版本。

## 必须修改

### 1. [37] HPSv3 漏作者

当前：

> Ma Y, Wu X, Sun K, Li H. HPSv3: Towards Wide-Spectrum Human Preference Score ...

作者官方仓库 `MizzenAI/HPSv3` 列出的作者为 Yuhang Ma、Yunhao Shui、Xiaoshi Wu、Keqiang Sun、Hongsheng Li。当前条目漏掉第二作者 Yunhao Shui，又没有使用 `et al.`，属于确定的书目信息错误。

建议改为：

> [37] Ma Y, Shui Y, Wu X, Sun K, Li H. HPSv3: Towards Wide-Spectrum Human Preference Score. Proceedings of the IEEE/CVF International Conference on Computer Vision, 2025. arXiv:2508.03789.

### 2. [33] 正文把 Di[M]O 的可核验结论写得过细

正文 1.7 写道：

> Di[M]O 通过匹配教师与学生的 token 级条件分布……并允许蒸馏后再用奖励优化[33]。

作者官方仓库 `yuanzhi-zhu/DiMO` 明确支持的是“matching the output distribution of teacher and student model”，公开代码也能看到离散 token 概率上的 FKL/RKL/Jeffreys 分布匹配；但官方 README 和公开代码未直接给出“蒸馏后再用奖励优化”的依据。该说法来自答辩 PPT 的二手概括，不能只用 [33] 无限定地背书。

建议正文改为：

> Di[M]O 通过在离散 token 概率空间中匹配多步教师与一步学生的输出分布，将掩码扩散模型蒸馏为一步生成器[33]。

“允许蒸馏后再用奖励优化”若不能提供论文具体页码/章节，应删除；若保留，必须写成“答辩材料将其概括为可在蒸馏后进行奖励微调”，不得表述为已由 [33] 直接核实的事实。

### 3. [24] 与图像生成基线的对应关系需说清

正文 1.4 在图像生成语境中写“MaskGRPO 和 TraceRL 分别代表这两类思路[23-24]”。[24] 的公开论文/官方仓库主要提出 diffusion language model 的 TraceRL；《Sketch Then Paint》[25] 才明确说明其图像实验“following the implementation of TraceRL”构造了 `TraceGRPO`。现句容易让读者误以为 [24] 本身已经直接提出图像生成版 TraceGRPO。

建议改为：

> MaskGRPO 代表随机重掩码路线[23]；《Sketch Then Paint》依据 TraceRL 的轨迹记录思想构造了图像生成对照方法 TraceGRPO[24-25]。

### 4. [32] 将稿件身份与投稿状态分开

[32] 的 PDF 作者、题名和内容可由上传的 `TPAMI_TTS.pdf` 核验，但“submitted to IEEE Transactions on Pattern Analysis and Machine Intelligence”不是公开出版事实，只由作者稿版式和答辩材料支持。为避免书目让读者误读为数据库已收录的 TPAMI 论文，建议改为：

> [32] Luo S, Xin Y, Cheng Y, et al. dMLLM-TTS v2: Enhanced Self-Verified and Efficient Test-Time Scaling for Diffusion Multi-Modal Large Language Models. Unpublished manuscript, 2026.

正文工作积累处可继续写“答辩材料标注为 TPAMI 在审”。

## 建议修改

### 1. 补足几类没有原始文献支撑的综述判断

这些句子不是虚假引用，但作为博士开题的“国内外研究进展”证据密度不足：

- 1.7 写“连续扩散中已形成一致性模型、轨迹蒸馏和分布匹配等路线”，却没有引用 progressive distillation、consistency models、distribution matching/distillation 的代表工作。
- 1.8 与 3.6 将 POPE、MMBench、SEED、MMMU 设为关键理解基准，却未列原始论文或固定版本。
- 1.11 关于 reward hacking、校准和评价器非平稳性的判断缺少代表性来源。
- 1.12 对教师状态/学生状态分布偏移的论述未引 DAgger、on-policy distillation 或相关生成蒸馏来源。

建议在第二轮只补充确实会用于方法论论证或实验协议的原始论文；新增条目仍需逐条核验题名、作者、年份、载体和正文对应关系，避免为了数量堆引文。

### 2. 统一会议书目完整度

[2]、[5]、[27]、[29]、[30]、[37] 等正式会议论文没有页码，而 [1]、[3]、[31]、[33]、[36]、[39]、[40] 有页码。此问题不构成虚假引用，但建议按学校规定的 GB/T 7714 或统一英文格式处理：正式论文集优先补卷/页码/DOI；arXiv 只保留 arXiv 编号；不要在同一条中混合尚未公开的会议状态与正式出版格式。

### 3. 降低“结果证明机制”的措辞

例如 1.4 的“这些结果支持‘早期结构决策应获得更多训练信用’”最好改为“这些结果与该假设一致”。[25] 的性能和消融支持方法有效，但不能单独识别“早期 token 的因果贡献”；报告后文已经设计反事实实验，前后措辞应一致。

## 可保留并已重点复核的条目

| 编号 | 结论 | 核验要点 |
|---|---|---|
| [1]—[8] | 可保留 | 经典原始论文；题名、作者、年份和会议/卷号匹配。 |
| [9] | 可保留 | 当前 `NeurIPS 36: 52132–52152, 2023` 为修正版本；所附《Sketch Then Paint》参考文献中的 `76341–76366` 是错误传播源，不要恢复。 |
| [10] | 可保留 | ImageReward 题名、作者、NeurIPS 2023 卷页匹配。 |
| [11]—[13] | 可保留 | Emu3、Show-o、Janus-Pro 题名/arXiv 身份匹配；Show-o 的 ICLR 2025 状态可由作者官方仓库确认。 |
| [14] | 可保留 | MMaDA 官方仓库明确标注 NeurIPS 2025，arXiv:2505.15809 与作者表匹配。 |
| [15] | 可保留 | Muddit 官方仓库题名、arXiv:2505.23606 及仓库描述中的 ICLR 2026 状态一致。 |
| [16] | 可保留 | 上传 PDF 与 `Alpha-VLLM/Lumina-DiMOO` 官方仓库均对应 arXiv:2510.06308；正文架构、Self-GRPO 和实验数值与 PDF 相符。 |
| [17]—[18] | 可保留 | arXiv 标识和题名匹配；[18] 用唯一 arXiv 编号锁定题名版本是稳妥做法。 |
| [19] | 可保留 | 只写 arXiv 是正确的。独立检索 ICML 2025 官方 PMLR v267 未找到该论文；所附 Lumina-DiMOO 文末把它写成 ICML 2025 属二手书目错误，不应传播。 |
| [20]—[23] | 可保留 | ICLR/arXiv 身份匹配；MaskGRPO 官方仓库明确标注 ICLR 2026。 |
| [24] | 条目可保留、正文需改 | TraceRL 官方仓库及 OpenReview 链接确认 ICLR 2026；问题在图像基线的归属表达，而非论文真实性。 |
| [25] | 可保留、状态保守 | arXiv:2605.16842、题名和作者可由 `Alpha-VLLM/Lumina-DiMOO/ht_grpo` 官方 README 核验；公开稿仍以 arXiv/匿名投稿版为准。“NeurIPS 2026 已录用”继续限定为答辩材料标注。 |
| [26] | 可保留 | UniRL 题名、作者和 arXiv:2505.23380 与所附 Lumina-DiMOO 文末元数据一致。 |
| [27] | 可保留 | CVPR 2025 正式题名确为 *Scaling Inference Time Compute for Diffusion Models*；所附 dMLLM-TTS 使用的 *Inference-Time Scaling for Diffusion Models Beyond Scaling Denoising Steps* 是预印本旧题名。当前稿处理正确。 |
| [28]—[30] | 可保留 | arXiv 身份匹配；Reflect-DiT 官方仓库明确 ICCV 2025，Reflection to Perfection 也在 ICCV 2025 论文清单中。 |
| [31] | 可保留 | 上传的是 CVF Open Access 正式版；作者、题名、CVPR 2026 及页码 35726–35735 匹配。 |
| [32] | 稿件存在，书目需降级 | 上传作者稿可证实稿件存在；TPAMI 状态不等于正式出版，应按上文拆开。 |
| [33] | 条目可保留、正文需改 | Di[M]O 官方仓库、ICCV 2025 清单、arXiv:2503.15457、作者和页码匹配；奖励微调主张需另证。 |
| [34] | 可保留 | `Kaiwen-Zhu/MIGM-Shortcut` 官方仓库的题名、作者、arXiv:2602.23996 和 2026 年一致；所附匿名稿参考文献误写为 2025，当前稿已改正。 |
| [35]—[36] | 可保留 | ELLA/DPG-Bench 来源关系成立；DeQA 对应的 CVPR 2025 题名、作者和页码匹配。 |
| [37] | 必须改作者 | 论文和 ICCV 2025 状态真实，仅作者表漏人。 |
| [38] | 可保留 | Unified Reward Model 的题名、作者和 arXiv:2503.05236 匹配。 |
| [39] | 可保留 | ICML 2026 官方 PMLR v306 元数据逐项匹配：题名、作者、页码 83169–83197、卷 306；不是虚构条目。 |
| [40] | 可保留 | 上传 CVF Open Access 正式版；作者、题名、ICCV 2025 和页码 4360–4369 匹配。正文已明确它是 ViT 分类/适配证据，不是生成模型直接证据。 |
| [41] | 可保留 | SDXL 的作者、题名、ICLR 2024 与 arXiv:2307.01952 匹配。 |

## 正文数值与主张抽查

- Lumina-DiMOO：GenEval `0.88→0.91`，Colors `0.89→0.95`，Attribute `0.76→0.82` 与上传 PDF 表 3 一致；不能据此推断理解指标改善，正文已有正确限制。
- Lumina-DiMOO ML-Cache：上传 PDF 图示为约 `45 s→22 s`，显存 `38.9 GB→45.9 GB`；官方仓库后续示例使用另一组 `58.2 s→32.2 s` 设置。正文明确“在其设置下”，可保留，后续应写清硬件/分辨率/版本。
- HT-GRPO：MMaDA `0.56→0.83`、Lumina-DiMOO `0.83→0.92` 及 MaskGRPO 对照与上传匿名稿表 1 一致。
- dMLLM-TTS：`O(NT)` 与近似 `O(N+T)`、最高约 6× 效率表述与 CVPR 正式版一致；“效率”应继续限定为搜索/前向计算口径。
- dMLLM-TTS v2：`T=32, N=32, K=8` 下 LTS `0.91`、HTS `0.93` 与上传稿表 1 一致；正文已经避免把 NFE 五分之一直接等同于端到端五倍加速。
- Di[M]O：`48 步 0.54 / 一步 0.43` 目前只能追溯到答辩材料概括，正文已有来源限定；定稿前应补论文表号或删除精确数字。

## Reviewer 判定

本轮判定为 **小修后通过引用真实性审查**。不存在“完全虚构的 41 条参考文献”，但在修复 [37]、降级 [32]、澄清 [24]/[33] 前，不建议在核验说明中使用“逐条均完全正确”这种绝对表述。第二轮应再次自动检查编号覆盖，并人工复核所有新增条目及被修改的事实句。
