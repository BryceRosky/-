# 开题报告交付说明

生成日期：2026-10-09。当前文件为两轮评阅后的待补信息评阅稿。

## 主要文件

| 文件 | 用途 |
|---|---|
| `thesis_proposal_luosiqi.docx` | 学校字体版可编辑 Word；建议在装有宋体、黑体、仿宋和 Times New Roman 的 Microsoft Word 中打开并用于最终填表 |
| `thesis_proposal_luosiqi.pdf` | 已验收的 55 页 A4 阅读版 PDF |
| `thesis_proposal_luosiqi_render_compatible.docx` | Linux/LibreOffice 渲染兼容副本，用于复现当前 PDF，不替代学校字体版 Word |
| `thesis_proposal_luosiqi_bundle.zip` | 主要文档、评审记录、引用审计、源稿和校验文件的完整下载包 |
| `thesis_proposal_content.md` | 冻结内容源稿 |
| `reference_audit.md` | 最终编号版引用真实性、元数据和正文支持关系审计 |
| `review_round1_summary.md` | 第一轮问题与修订对照 |
| `review_round2_summary.md` | 第二轮问题关闭情况与最终判定 |
| `revision_log_two_rounds.md` | 两轮迭代合并记录 |
| `final_logic_qc.md` / `final_format_qc.md` | 最终逻辑与格式验收记录 |
| `SHA256SUMS` | 交付文件完整性校验值 |

`reviews/round1/` 与 `reviews/round2/` 保存格式、逻辑、文献综述、引用、方法统计和模拟答辩六类原始评审意见；`iterations/` 保存两轮内容快照；`assets/` 与 `build/` 保存框架图和构建脚本。

## 最终核验结果

- 第一轮判定：大修后复审。
- 第二轮判定：定向小修完成后，内容与方法评阅通过；正式提交前补封面事实与资源/个人贡献证据。
- 参考文献：49 条连续唯一，正文 49/49 全部引用，首次出现严格按 1—49 排列；48 条有公开记录，1 条 `dMLLM-TTS v2` 仅由附件作者稿支持并标为 `Unpublished manuscript, 2026`。
- PDF：55 页、A4、无空白页；全文可提取且无替换字符；33 个字体子集全部嵌入并带 Unicode 映射；三处公式、图 1 和三张表均存在。
- 封面保持 5 个 `【待填写】`：学号、学院、专业、开题日期、开题地点。学术型博士生、全日制、自拟课题为当前暂定选择，须由本人确认。

## 正式提交前必须补充

1. 补齐封面五项事实，确认姓名、导师、论文题目及三个选择项与学籍系统一致，并在实际签署时填写签名和日期。
2. 提供实际 GPU 型号、数量、GPU-hours、存储及任务时限等资源证明；资源不足时按正文的分级门槛收缩结论。
3. 用代码提交、实验日志、稿件段落或本人/导师/合作者核签确认个人贡献比例。
4. 在提交日复核 NeurIPS 录用、TPAMI 在审、IJCV/Pattern Recognition 审稿状态；没有正式外部记录时继续按“答辩材料标注”表述。
