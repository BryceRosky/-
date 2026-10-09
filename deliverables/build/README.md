# 构建说明

`build_proposal.py` 从学校原始模板、冻结 Markdown 源稿和统一价值框架图重建两份 DOCX：

```bash
python deliverables/build/build_proposal.py
```

依赖 Python 3、`python-docx` 和 `lxml`。脚本输出：

- `deliverables/thesis_proposal_luosiqi.docx`：学校字体版；
- `deliverables/thesis_proposal_luosiqi_render_compatible.docx`：使用 Noto/Liberation 字体的 Linux 渲染副本。

在安装 LibreOffice 和 Noto CJK 字体的环境中，可从渲染兼容版生成 PDF：

```bash
soffice --headless --convert-to pdf --outdir deliverables deliverables/thesis_proposal_luosiqi_render_compatible.docx
```

生成后的默认 PDF 文件名带 `_render_compatible`；验收通过后可重命名为 `thesis_proposal_luosiqi.pdf`。正式提交应使用学校字体版 DOCX 补齐封面与签名。
