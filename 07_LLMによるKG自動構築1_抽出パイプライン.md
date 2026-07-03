---
marp: true
theme: default
paginate: true
header: "LLM × Knowledge Graph レクチャー"
footer: "第7章 LLMによるKG自動構築(1):抽出パイプライン"
style: |
  section {
    font-size: 24px;
    justify-content: flex-start;
    padding-top: 70px;
    color: #0E2841;
    background: #FFFFFF;
    font-family: "Noto Sans JP", "Yu Gothic", "Meiryo", "Hiragino Kaku Gothic Pro", Arial, sans-serif;
  }
  section.title {
    justify-content: center;
    text-align: center;
    position: relative;
    background: linear-gradient(135deg, #FFFFFF 55%, #E6F7F7 100%);
  }
  section.title::before {
    content: "";
    position: absolute;
    top: 36px;
    left: 40px;
    width: 26px;
    height: 26px;
    background: #0BA6AA;
  }
  section.title::after {
    content: "";
    position: absolute;
    left: 0;
    right: 0;
    bottom: 0;
    height: 14px;
    background: linear-gradient(90deg, #0E2841 0%, #0BA6AA 100%);
  }
  h1 { font-size: 40px; color: #0E2841; }
  h2 {
    font-size: 32px;
    color: #0E2841;
    border-bottom: 3px solid #0BA6AA;
    padding: 0 0 4px 16px;
    position: relative;
  }
  h2::before {
    content: "";
    position: absolute;
    left: 0;
    top: 8px;
    width: 10px;
    height: 10px;
    background: #0BA6AA;
  }
  table { font-size: 20px; border-collapse: collapse; }
  table th { background: #0E2841; color: #FFFFFF; }
  table th, table td { border: 1px solid #92AAB7; }
  code { font-size: 90%; }
  pre { font-size: 18px; line-height: 1.35; background: #F4F7F8; }
  blockquote { background: #E6F7F7; border-left: 6px solid #0BA6AA; padding: 8px 16px; color: #0E2841; }
  .columns { display: grid; grid-template-columns: 1fr 1fr; gap: 24px; }
  .small { font-size: 18px; }
  footer, header { color: #0BA6AA; }
---

<!-- _class: title -->
<!-- _paginate: false -->

# LLM × Knowledge Graph
## 効果が出るソリューションを作る

### 第7章 LLMによるKG自動構築(1):抽出パイプライン

<!--
- ここからコースの「構築」パート。1章で見せたGraphRAGデモのKGは、まさにこの章の技術で作られていたと種明かしする
- 「KG構築は人手で高コスト」が長年の普及の壁だった。LLMがそれを壊した — 本コースの存在理由に関わる章
- 2〜6章の道具が全部使われる:2章のモデル、4章のCypher、5章のオントロジー、6章の外部知識
- 今日は「動く抽出パイプライン」まで。品質とコストの作り込みは次章(8章)と分担
-->

---

## 本章のゴール

- KG構築パイプラインの**全体像**と各工程の役割を理解する
- LLMトリプル抽出の**プロンプト+structured output**を設計できる
- **オントロジー誘導型抽出**でハルシネーションを抑える方法を知る
- チャンク戦略と抽出漏れ・重複への対処を設計できる
- 既製ツール(LangChain / Neo4j GraphRAG)を**中身を理解した上で**使える
- 冪等・再実行可能なパイプラインとして実装できる

> 本章のスコープ:テキスト → トリプル → Neo4j 投入まで
> 名寄せ・品質評価・コストは**次章(8章)**

<!--
- ゴールが多めだが、串は1本:「LLMに“決められた形”で事実を吐き出させ、機械的にグラフへ流し込む」
- 「structured output」「JSONスキーマ」は生成AI経験者なら使ったことがあるはず — 挙手で確認。あるなら話が早い
- 5章のオントロジーが「LLMの制御装置」としてここで実戦投入される、という接続を最初に言う
- 8章との分担を明確に:今日の成果物は「動くが品質未保証のパイプライン」。それでよい、と期待値を設定
-->

---

## KG構築パイプラインの全体像

```
┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐
│ スキーマ設計 │ → │   抽出     │ → │ 正規化・解決 │ → │   投入     │ → │  品質管理   │
│  (5章)     │   │ 【本章】    │   │  (8章)     │   │ 【本章】    │   │  (8章)     │
└────────────┘   └────────────┘   └────────────┘   └────────────┘   └────────────┘
 オントロジー      テキスト→        名寄せ・         MERGE で        評価・検証・
 =抽出の制約      トリプル         外部KGリンク      Neo4jへ         継続更新
```

- **本章**:抽出と投入 — パイプラインの「背骨」を通す
- **8章**:正規化・解決と品質管理 — 「動くデモ」を「使えるシステム」にする
- スキーマ設計(5章)が最上流:**ここが曖昧だと下流が全部ブレる**

<!--
- この図が7〜8章の地図。迷子になったらここに戻る、と宣言
- 「抽出→投入」を先に通すのは意図的:早く動くものを作り、品質改善(8章)は計測しながら回すため。ウォーターフォールで完璧を目指すと頓挫する(11章の失敗パターン先出し)
- 正規化・解決を飛ばすと何が起きるか予告:「トヨタ」と「トヨタ自動車」が別ノードになったKGが今日できあがる。それを8章で直す
- 受講者への問いかけ:「ETLパイプラインの経験者は?」— KG構築はETLの一種(Extract相当がLLM)と捉えると設計勘が働く
-->

---

## なぜLLMで構築するのか:従来手法との比較

| 観点 | 従来(NER+関係抽出モデル) | LLM抽出 |
|---|---|---|
| 精度 | ドメイン学習済みなら高い | 汎用で高い。ゼロショットでも実用域 |
| **開発コスト** | 学習データ作成に数週間〜数ヶ月 | **プロンプト設計のみ。数日** |
| 新しい関係の追加 | 再アノテーション+再学習 | **プロンプト修正のみ** |
| ドメイン適応 | ドメイン毎にモデル構築 | few-shot例の差し替えで対応 |
| 実行コスト | 安い(自前GPU/CPU) | API課金(トークン量に比例) |
| 弱点 | 未知の関係型に弱い | **ハルシネーション**、再現性 |

**使い分け**:少種類・大量・定型 → 従来手法も検討 / 多様・変化する関係 → LLM

<!--
- 「LLMが常に正解」ではない、と最初に釘を刺す。大量・定型(例:毎日100万件の定型文書から日付と金額だけ)なら従来手法や正規表現が速くて安い
- LLMの本質的な優位は「開発コストと変更容易性」:関係の種類を1つ足すのに、再学習ではなくプロンプトの1行追加で済む
- 実行コストの逆転に注意:文書量が桁で増えるとAPI課金が効いてくる。8章のコスト設計で扱う
- 右下の「ハルシネーション」がこの章の主敵。以降のスライド(structured output、オントロジー誘導)はすべてこれとの戦い
-->

---

## LLMトリプル抽出:プロンプト設計の実例

```text
あなたは知識グラフ構築のためのトリプル抽出器です。
以下のテキストから、エンティティと関係を抽出してください。

# 制約
- エンティティ型は Person / Company / Product のみ
- 関係型は WORKS_AT / CEO_OF / DEVELOPS / PARTNER_OF のみ
- テキストに明示的に書かれた事実のみ抽出する。推測・一般知識で補わない
- 該当がなければ空配列を返す

# テキスト
{text}
```

プロンプトの4要素:**役割**・**型の制約**・**根拠の制約**・**出力形式**(次スライド)

<!--
- 生成AI経験者向けに「何が普通のプロンプトと違うか」に絞って解説する
- 効き所その1「型の制約」:自由に抽出させると型が発散する(「CEO」「代表取締役」「社長」が別関係になる)。許可リスト方式が基本
- 効き所その2「テキストに明示された事実のみ」:LLMは一般知識で穴埋めしたがる(例:文中にない「トヨタの本社は愛知」を勝手に足す)。この1行の有無で結果が目に見えて変わる — 演習で実験させる
- 「空配列を返してよい」も地味に重要:逃げ道がないとLLMは無理やり何か抽出する
- このプロンプトの制約部分、どこかで見た形では? → 5章のオントロジーそのもの。次スライドで機械可読にする
-->

---

## structured output:JSONスキーマ=5章のオントロジー

```json
{
  "type": "object",
  "properties": {
    "triples": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "subject":      { "type": "string" },
          "subject_type": { "enum": ["Person", "Company", "Product"] },
          "predicate":    { "enum": ["WORKS_AT", "CEO_OF", "DEVELOPS", "PARTNER_OF"] },
          "object":       { "type": "string" },
          "object_type":  { "enum": ["Person", "Company", "Product"] },
          "source_text":  { "type": "string" }
        },
        "required": ["subject", "subject_type", "predicate", "object", "object_type"]
      }
    }
  }
}
```

- `enum` = 5章で設計した**クラスと関係型そのもの**
- `source_text`(根拠の原文)を必ず持たせる → 8章のプロビナンスに繋がる

<!--
- 5章の伏線回収の瞬間:「オントロジーは文書ではなくコード。JSONスキーマとしてLLM APIに直接渡る」
- structured output(各社APIのスキーマ強制機能)を使えば、パース不能なJSONや形式逸脱は原理的に排除できる。「お願い」ではなく「強制」
- ただしスキーマが守られても「中身の正しさ」は保証されない:形式のガードレールと事実のガードレールは別、と明確に区別する
- source_textは今は使わないが必ず入れておく:8章で「この事実の根拠は原文のどこか」を示すプロビナンスとして効く。後から足すのは大変
- few-shot例を2〜3個添えると精度が安定する:良い例+「抽出すべきでない例(推測になるケース)」のペアが効く
-->

---

## オントロジー誘導型抽出:型制約でハルシネーションを抑える

**制約なし(自由抽出)で起きること**

```
入力:「田中氏はアクメ社のCTOとして新製品Xを発表した」
出力:(田中, 役職, CTO)(アクメ社, 発表, 製品X)(田中, 有名人である, true)…
      → 型が発散・勝手な関係・一般知識の混入
```

**オントロジー誘導(型を enum で制約)**

```
出力:(田中:Person, WORKS_AT, アクメ社:Company)
      (アクメ社:Company, DEVELOPS, 製品X:Product)
      → 語彙が収束。スキーマ外の「創作」が構造的に不可能
```

- 制約は**表現力とのトレードオフ**:狭すぎると本当にある事実を取りこぼす
- 運用:取りこぼしを定期レビュー → 必要ならオントロジーに型を追加(5章の開世界設計)

<!--
- この章の核心スライド。「ハルシネーション対策の第一防衛線はプロンプトの工夫ではなくスキーマ制約」
- 上下の対比を実演すると効果的:同じテキストを制約あり/なしで抽出して差を見せる(演習でも体験させる)
- 「CTOという役職情報が消えたのでは?」という鋭い質問が来たら:その通り。WORKS_ATにroleプロパティを持たせる設計もある(2章のプロパティ設計)。「何を捨てるか」はコンピテンシークエスチョン次第(5章)
- トレードオフの落とし所:最初は狭く始めて、取りこぼしログを見ながら型を足す。閉じたスキーマを「育てる」運用が現実解
- 「LLMの創造性を殺すのが正しい」というこの用途の特殊性を強調。生成タスクとは真逆の設計思想
-->

---

## チャンク戦略(1):分割と文脈保持

**問題:長文をそのまま入れられない/入れたくない**
(コンテキスト上限、長文での抽出精度低下、コスト)

| 戦略 | 内容 | 注意点 |
|---|---|---|
| 固定長+オーバーラップ | N トークンごと、10-20%重複 | 文境界を跨ぐと精度低下 |
| **構造単位**(推奨) | 段落・セクション・章で分割 | 文書形式ごとの前処理が必要 |
| 文脈ヘッダ付与 | 各チャンクに文書名・章題を前置 | わずかなトークン増で大幅改善 |

**代名詞・照応の問題**:「同社は…」→ どの会社?チャンク先頭では解決不能
→ 対策:オーバーラップ、文脈ヘッダ、または前段でLLMに**照応解決**(言い換え)させる

<!--
- ベクトルRAGのチャンク分割経験者は多いはず。「検索用チャンク」と「抽出用チャンク」で最適が違う点が今日の新情報
- 抽出では「1つの事実が分割線で泣き別れる」のが最悪:「田中氏がアクメ社の」/「CTOに就任した」— どちらのチャンクからも抽出できない
- 文脈ヘッダは費用対効果が最高のテク:「文書:2025年度組織改編通知/セクション:営業本部」を頭に付けるだけで「同社」「当部門」の解決率が上がる
- 照応解決の前処理はコストが倍かかる。まずヘッダ+オーバーラップで測ってから検討、の順番を推奨
- 「チャンクサイズはいくつがいいですか?」には:文書と型による、8章で測り方を学んでから自分のデータでA/Bする、と答える
-->

---

## チャンク戦略(2):抽出漏れと重複への対処

**チャンク分割の宿命:同じ事実が複数チャンクに、ある事実がどこにも**

| 問題 | 原因 | 対処 |
|---|---|---|
| **重複抽出** | オーバーラップ・繰り返し記述 | 投入時の `MERGE` で自然に吸収(後述) |
| **表記ゆれ重複** | 「田中太郎」「田中氏」 | 8章のエンティティ解決で対処 |
| **抽出漏れ** | 分割で文脈切断、LLMの見落とし | 複数回抽出の和集合、gleaning |

- **gleaning**(Microsoft GraphRAGの手法):同じチャンクに「見落としはないか?」と**追加の抽出パスを回す** — 再現率が上がるがコスト増
- 設計指針:**漏れと重複なら、重複に倒す**(重複はMERGEと8章で消せる。漏れは検出が難しい)

<!--
- 「漏れと重複、どちらがマシか?」と問いかけてから指針を出すと納得感が出る。重複は後から機械的に潰せるが、漏れは「ないことに気づけない」
- gleaningは「もう一度見て。見落としある?」と聞くだけの単純な手法だが、LLMの見落としは1パス目で意外と多く、実測で効果が確認されている。ただしパス数×コストなので8章のコスト設計とセット
- 重複がMERGEで消える仕組みは投入スライドで実物を見せる。ここでは「Neo4j側に受け皿がある」とだけ
- 表記ゆれは本章では対処しない(スコープ外)と明言。今日のKGには田中さんが2人いる状態になる — 8章への引き
-->

---

## 既製ツール(1):LangChain LLMGraphTransformer

```python
from langchain_experimental.graph_transformers import LLMGraphTransformer
from langchain_anthropic import ChatAnthropic
from langchain_core.documents import Document
from langchain_neo4j import Neo4jGraph

llm = ChatAnthropic(model="claude-sonnet-4-5", temperature=0)
transformer = LLMGraphTransformer(
    llm=llm,
    allowed_nodes=["Person", "Company", "Product"],      # ← オントロジー誘導
    allowed_relationships=["WORKS_AT", "CEO_OF", "DEVELOPS"],
)
docs = [Document(page_content=chunk) for chunk in chunks]
graph_docs = transformer.convert_to_graph_documents(docs)

graph = Neo4jGraph()
graph.add_graph_documents(graph_docs, include_source=True)  # 原文ノードも保存
```

- ここまで学んだ内容(プロンプト+スキーマ制約+投入)が**この十数行に凝縮**されている

<!--
- 「今まで自前で設計してきたものが数行で動く」ことにガッカリさせない:中身を知らずに使うと、精度が出ないときに手も足も出ない。allowed_nodes/relationshipsがオントロジー誘導そのものだと対応づける
- include_source=True で原文チャンクが(:Document)ノードとして残り、MENTIONSエッジで繋がる — プロビナンス(8章)と、9章のハイブリッドGraphRAGの土台になる重要オプション
- temperature=0は抽出タスクの定石(再現性重視)
- experimentalパッケージである点に注意:APIが変わりやすい。バージョン固定を推奨
- 内部プロンプトは英語ベース。日本語文書で精度が出ないときは自前プロンプト(前スライドの設計)に切り替える判断ができるのが「中身を理解した上で使う」ということ
-->

---

## 既製ツール(2):Neo4j GraphRAG SimpleKGPipeline

```python
from neo4j_graphrag.experimental.pipeline.kg_builder import SimpleKGPipeline
from neo4j_graphrag.llm import AnthropicLLM
from neo4j_graphrag.embeddings import OpenAIEmbeddings

pipeline = SimpleKGPipeline(
    llm=AnthropicLLM(model_name="claude-sonnet-4-5"),
    driver=neo4j_driver,
    embedder=OpenAIEmbeddings(),       # チャンクのembeddingも同時格納
    entities=["Person", "Company", "Product"],
    relations=["WORKS_AT", "CEO_OF", "DEVELOPS"],
    from_pdf=False,
)
await pipeline.run_async(text=text)
```

| | LLMGraphTransformer | SimpleKGPipeline |
|---|---|---|
| 提供元 | LangChain(experimental) | Neo4j公式 |
| 範囲 | 抽出+投入 | 分割〜抽出〜embedding〜投入まで一式 |
| 向き | 既存LangChain構成に組み込む | Neo4j中心でまとめて構築 |

<!--
- Neo4j公式パッケージの強み:チャンク分割・抽出・embedding格納・投入までパイプライン一式が入っている。9章のGraphRAG(同じパッケージ)へ地続き
- embedderを渡すとチャンクノードにベクトルが付く=4章のベクトルインデックス+9章のハイブリッド検索の準備が構築時に済む、という設計
- どちらを選ぶか:正解はない。両方とも「素早く背骨を通す」用途では優秀。精度を詰める段階でカスタムプロンプト・カスタム分割に差し替えることになる
- 「experimental」が両方に付いている点は正直に:この領域はAPIの動きが速い。抽象(パイプラインの各工程)を理解していればツールの乗り換えは怖くない
-->

---

## LLM不要な部分にLLMを使わない

**構造化データはすでに「エンティティと関係」になっている**

| データ源 | 手段 | LLMの出番 |
|---|---|---|
| RDBのマスタ・トランザクション | SQL→CSV→`LOAD CSV` / ドライバで直接変換(4章) | **なし** |
| CSV / Excel台帳 | 列→プロパティ、外部キー相当→リレーション | なし(列の意味の解釈程度) |
| 外部KG(Wikidata等) | SPARQL→MERGE(6章) | なし |
| **非構造化テキスト** | **本章のLLM抽出** | **ここだけ** |

- 実務のKGは**構造化データが骨格、LLM抽出が肉付け**が定石
- 例:社員・組織・顧客マスタはRDBから確実に投入 → 文書からは「誰が何を担当」等の**マスタにない関係**だけLLMで抽出

<!--
- コストと品質の両面で最重要の設計判断。「全文書をLLMに食わせる」のは高くて不正確な遠回り
- RDB由来のデータはLLM抽出よりも確実(precision 100%)。骨格が確かだと、LLM抽出結果の名寄せ先(8章)としても機能する
- 「マスタにあるものはマスタから、文書にしかないものだけLLMで」— この線引きを要件定義の段階でやる
- 受講者への問いかけ:「みなさんの社内で、KGに入れたい情報のうち構造化済みの割合は?」— 意外と骨格の大半はRDBにある、という気づきを促す
- 3章の併用アーキテクチャ(RDB→Graph同期)がここで再登場していることも指摘
-->

---

## Neo4jへの投入:MERGEによる冪等な書き込み

```python
records = [t.model_dump() for t in extraction.triples]  # 抽出結果(JSON)
```

```cypher
UNWIND $records AS t
CALL apoc.merge.node([t.subject_type], {name: t.subject}) YIELD node AS s
CALL apoc.merge.node([t.object_type],  {name: t.object})  YIELD node AS o
CALL apoc.merge.relationship(s, t.predicate, {}, {source: t.source_text}, o)
  YIELD rel
RETURN count(rel)
```

- `MERGE` = **なければ作る、あればマッチ**(4章)→ 重複抽出が自然に吸収される
- ラベル・関係型を動的に指定するため APOC を利用(固定なら素の `MERGE` でよい)
- 事前に一意制約を作成:`CREATE CONSTRAINT FOR (p:Person) REQUIRE p.name IS UNIQUE`

<!--
- 4章のMERGEがパイプラインの要になる回収ポイント:「同じトリプルが3回来ても、グラフには1つ」— チャンク重複対策の受け皿
- APOCが必要な理由を明確に:CypherのMERGEはラベル・関係型をパラメータ化できない。抽出結果の型でノードを作り分けるにはapoc.merge.*を使う(型の種類が少なければWHEN分岐やクエリ生成でも可)
- 制約はMERGEの性能と整合性の両方に効く。「制約なしMERGEは遅い」は4章の復習
- ここでのMERGEキーがnameであることの危うさに触れる:「田中太郎」が同姓同名なら誤マージ、「田中氏」なら重複 — 名寄せ(8章)がなぜ独立した工程なのかの伏線
- source_textをエッジのプロパティに載せている:根拠を持つKGの最小実装
-->

---

## パイプライン設計:冪等性・再実行・差分更新

| 設計原則 | 内容 | 実装 |
|---|---|---|
| **冪等性** | 同じ入力を何度流しても結果が同じ | `MERGE` + 一意制約(`CREATE`禁止) |
| **再実行** | 途中失敗から安全にやり直せる | 文書単位で処理状態を記録(pending/done/error) |
| **差分更新** | 新規・変更文書だけ処理 | 文書のハッシュ・更新日時で判定 |
| **追跡可能性** | どの文書からどのエッジができたか | `(:Document)-[:MENTIONS]->` +エッジにsource |

```
処理台帳(RDBでもNeo4jでもよい):
  doc_id | content_hash | status | processed_at | model | prompt_version
```

- **prompt_version を記録する**:プロンプト改善後、どの文書を再処理すべきか判る

<!--
- 「LLMを呼ぶ処理は失敗する」前提で設計する:レートリミット、タイムアウト、不正な出力。1000文書の800件目で落ちたとき、最初からやり直しは費用も時間も無駄
- ETL経験者には馴染みの設計ばかり。「LLMが入っても本質はETL」で安心させる。唯一の新要素がprompt_version
- prompt_versionの実務価値を具体例で:「関係型を1つ追加した→過去文書を全部再処理?」— バージョン記録があれば旧バージョンで処理した文書だけ再処理できる
- 文書を更新・削除したときのグラフ側の掃除(古いエッジの削除)は差分更新の難所。MENTIONSエッジで文書由来のサブグラフを特定できる設計にしておくと消せる
- 8章の増分処理・鮮度管理はこの台帳設計の延長にある
-->

---

## パイプライン全体像(本章のまとめ図)

```
文書群 ──→ ①前処理・チャンク分割(構造単位+文脈ヘッダ)
              │
              ▼
          ②LLM抽出(プロンプト+JSONスキーマ=5章オントロジー)
              │      ↺ gleaning(任意)
              ▼
          ③検証(スキーマ適合・enum外の型を弾く)
              │
              ▼
          ④Neo4j投入(UNWIND+MERGE、原文リンク付き)

  並走:RDB/CSV/外部KG ──(LLMなし)──→ ④へ直接投入
  台帳:文書ごとの status / hash / prompt_version を記録
```

<!--
- 1枚で本章を総復習する図。演習の前に「今から作るもの」として提示する
- ③の検証を軽視しない:structured outputでも、enumにない型が来る・主語と目的語が同一・空文字などは起きる。投入前の機械チェック(Pydantic等)は数行で書けて事故を大きく減らす
- 「並走」の線が入っていることを確認:LLMパイプラインと構造化データ投入は合流する設計(骨格+肉付け)
- この図の②③に品質評価、②〜④の間に名寄せが挟まるのが8章。図がどう成長するか予告する
-->

---

## 【演習】テキストからKGを構築する

**課題:ニュース記事風テキスト10件(配布)からKGを構築・可視化する**

1. 抽出の実装(20分)
   - 配布のJSONスキーマ(Person / Company / Product+関係4種)でLLM抽出
   - 1件だけ**型制約を外して**抽出し、結果の発散を観察する
2. Neo4j投入(10分)
   - `UNWIND` + `MERGE` テンプレートで投入(一意制約も作成)
   - 同じ文書を**2回流して**ノード数が増えないこと(冪等性)を確認
3. 可視化と観察(10分)
   - Neo4j Browserで全体を表示し、**気になる点をメモ**
   - 例:同一人物が別ノード? 関係の向きは正しい? 根拠は辿れる?

⏱ 40分(個人ワーク+最後に観察結果の共有5分)

<!--
- 配布物:テキスト10件、抽出スクリプト雛形、投入Cypherテンプレート、docker環境(付録A)。写経ではなく穴埋め形式
- 1の「制約を外す実験」が学習効果最大:オントロジー誘導の価値は、外した結果の惨状を見るのが一番早い
- 2の「2回流す」も体験必須:冪等性は言葉で聞くより、ノード数が増えない画面を見る方が腹落ちする
- 3の観察メモが8章の教材になる:「田中太郎と田中氏が別ノード」「A社とエー社」等の発見をそのまま次章の名寄せ課題として持ち越す — 回収を約束する
- 時間が余った人向け:gleaning(2パス目)を実装して抽出数の変化を見る/自分で書いたテキストを食わせてみる
-->

---

## 第7章まとめ

- KG構築は**5工程のパイプライン**:スキーマ設計→抽出→正規化・解決→投入→品質管理(本章は抽出と投入)
- LLM抽出の強みは**開発コストと変更容易性**。ただし定型・大量なら従来手法も検討
- 制御の要は **structured output+オントロジー誘導**(5章のスキーマがそのまま制約になる)
- チャンクは**構造単位+文脈ヘッダ**、漏れと重複なら**重複に倒す**(MERGEで吸収)
- 既製ツール(LLMGraphTransformer / SimpleKGPipeline)は**中身を理解して**使う
- **LLM不要な部分にLLMを使わない**:構造化データが骨格、LLM抽出は肉付け
- パイプラインは**冪等・再実行可能・差分更新**が前提。prompt_versionも記録

**次章**:KG自動構築(2)— エンティティ解決(名寄せ)・品質評価・コスト設計。「動くデモ」を「使えるシステム」へ

<!--
- 演習で作ったKGの「気持ち悪さ」(重複ノード、怪しいエッジ)を全員が抱えた状態で終わるのが理想。それが8章の学習動機になる
- 「今日のKGはまだ使い物にならない。でもそれで正しい。品質は測って直すもの」— 完璧主義で止まるより背骨を先に通す、を再確認
- 次章予告:「トヨタ/トヨタ自動車/Toyota Motorを1つにする技術、抽出精度の測り方、そしてAPIコストの見積もり。6章のWikidata IDがここで再登場します」
- 演習環境は8章でそのまま使うので消さないよう案内して終了
-->
