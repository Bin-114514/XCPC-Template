# HUST Three Boars XCPC Template

哈斯特三野猪队伍的补充模板库。

本仓库不追求收集所有算法，只维护另一份已打印模板中缺失、存在坑，或不完全符合本队编码习惯的内容。Markdown 是唯一编辑源；PDF 由 GitHub Actions 自动生成，供赛前打印或离线阅读。

## 快速使用

```text
git pull
编辑 docs/对应分区.md
git add docs/对应分区.md
git commit -m "docs: update <section>"
git push
```

队友直接提交自己的分支即可。由仓库维护者负责 PR、检查和合并。

## 内容索引

- [补充模板手册](docs/supplement.md)
- [贡献说明](CONTRIBUTING.md)

## PDF

合并到 `main` 后，`release-pdf.yml` 会生成带目录和页码的 `hust-three-boars-template.pdf`。在 GitHub Actions 的运行记录中下载 artifact；需要固定赛前版本时给提交打 tag 并从 Release 下载。
