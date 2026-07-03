# LLM × Knowledge Graph レクチャー

一般ITエンジニア(RDBMS・プログラミング・生成AI経験者)向けに、Ontology / Knowledge Graph / Graph DB / RDF を理解し、LLMと組み合わせて効果が出るソリューションを実装できるようになるための授業スライド。

## 構成

| ファイル | 内容 |
|---|---|
| `00_コース目次と内容説明.md` | コース全体の目次・各章の学習目標と内容 |
| `01`〜`11_*.md` | 各章のスライド(Marp形式、全11章) |

各スライドの直後に HTMLコメント `<!-- ... -->` で講師ノート(話すポイントの箇条書き)を記載。Marp ではプレゼンターノートとして扱われる。

## スライドのビルド(Marp)

```bash
# Marp CLI(要 Node.js)
npm install -g @marp-team/marp-cli

# HTMLに変換
marp 01_なぜLLMにはKnowledge\ Graphが必要か.md -o build/01.html

# PDF / PPTX に変換
marp --pdf --allow-local-files 01_*.md
marp --pptx 01_*.md

# 全章一括
for f in 0[1-9]*.md 1[01]*.md; do marp --pdf "$f" -o "build/${f%.md}.pdf"; done

# プレビュー(VS Code の場合は Marp for VS Code 拡張を推奨)
marp -s .
```

## 講師ノートの見方

- Marp for VS Code:プレビューで自動認識
- PPTX出力:スピーカーノート欄に入る
- PDF出力:`--pdf-notes` オプションで注釈として埋め込み可能
