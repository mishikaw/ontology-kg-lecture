---
marp: true
theme: default
paginate: true
header: "LLM × Knowledge Graph レクチャー"
footer: "第6章 RDF / SPARQL と外部知識の活用"
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

### 第6章 RDF / SPARQL と外部知識の活用

<!--
- 本コースはプロパティグラフ(Neo4j)主軸だが、この章だけRDFの世界に足を踏み入れる
- 冒頭で宣言する:「RDFを教養として学ぶ章ではありません。外部の公開ナレッジグラフからデータを取ってくるための実用スキルの章です」
- 4章(Neo4j)と並行受講可の位置づけ。Cypherを先に知っていればSPARQLとの対比で理解が速い
- 今日のゴール画像:Wikidataから取ったデータが自社のNeo4jに入っている状態
-->

---

## 本章のゴール

- RDF(トリプル・IRI・リテラル・Turtle)を**読み書きできる**
- SPARQLの基本を**SQL対比**で習得し、実クエリを書ける
- **Wikidata SPARQLエンドポイント**から実データを取得できる
- 外部KGを自社KGの強化に使う**活用パターン**を知る
- RDF → Neo4j(プロパティグラフ)への取り込み方法を知る

> RDFは「教養」ではなく「**外部知識の取り込み手段**」
> Wikidata・DBpedia・標準語彙の世界はRDFでできている

<!--
- 「なぜNeo4jのコースでRDF?」という疑問には次のスライドで正面から答える
- ゴールは「RDFの専門家になる」ではなく「必要なデータを外から取ってこられる」こと
- 8章のエンティティリンキング(Wikidata IDによる名寄せ)で本章の成果を使う、と先に予告
- SQLとCypherの経験がそのまま活きる章。身構えなくてよいと伝える
-->

---

## なぜプロパティグラフ主軸のコースでRDFを学ぶのか

| 世界 | モデル | 代表 |
|---|---|---|
| 自社KG(本コース主軸) | プロパティグラフ | Neo4j, Neptune |
| **公開ナレッジグラフ** | **RDF** | **Wikidata, DBpedia** |
| **標準語彙・オントロジー** | **RDF** | **schema.org, FIBO, SNOMED** |

- 世界最大級の構造化知識(Wikidata:1億超のエンティティ)は**RDFでしか取れない**
- 業界標準ID・標準語彙への接続もRDFの世界
- つまり:**外に出るときの共通言語がRDF**。中はプロパティグラフでよい

<!--
- 「社内はNeo4j、外の世界はRDF」という構図を最初に固定する。以降の学習動機がぶれない
- Wikidataの規模感を具体的に:人物・企業・地名・作品など1億超、多言語ラベル付き、無料・API制限緩め
- 「自社データだけでKGを作ると、企業名の表記ゆれや業界分類を全部自前で整備することになる。外部KGはそれを肩代わりしてくれる」
- 受講者への問いかけ:「マスタデータ整備に苦労した経験は?」— 外部KG活用の価値が刺さる層を確認
-->

---

## RDF最速理解:すべてはトリプル

**トリプル = 主語(Subject)– 述語(Predicate)– 目的語(Object)**

```
<http://example.org/toyota> <http://schema.org/founder> <http://example.org/kiichiro> .
<http://example.org/toyota> <http://schema.org/name> "トヨタ自動車"@ja .
```

| 構成要素 | 内容 | RDB/プロパティグラフとの対応 |
|---|---|---|
| **IRI** | エンティティ・関係のグローバル一意識別子(URL形式) | 主キー(ただし**世界で一意**) |
| **リテラル** | 文字列・数値・日付などの値。言語タグ `@ja`、型 `^^xsd:date` | カラム値 / プロパティ値 |
| **トリプル** | 主語–述語–目的語の3つ組。KGはトリプルの集合 | 1行1事実 / 1エッジ |

<!--
- 「RDFは主語-述語-目的語の3つ組がすべて。テーブルも、ノードのプロパティも、全部トリプルに分解される」
- IRIの本質は「世界で一意なID」。RDBの主キーはDB内で一意、IRIはWeb全体で一意 — だから他組織のデータと機械的に繋がる
- リテラルの言語タグはWikidata活用で必須知識(日本語ラベルを取るのに使う)
- プロパティグラフとの違いで混乱しやすい点を先回り:「RDFのエッジ(述語)にはプロパティを直接持てない」— 変換時の論点として後で再登場
-->

---

## Turtle記法:RDFの標準的な書き方

```turtle
@prefix ex:     <http://example.org/> .
@prefix schema: <http://schema.org/> .
@prefix xsd:    <http://www.w3.org/2001/XMLSchema#> .

ex:toyota a schema:Corporation ;                      # a = rdf:type
    schema:name "トヨタ自動車"@ja , "Toyota Motor"@en ;  # , で目的語を列挙
    schema:foundingDate "1937-08-28"^^xsd:date ;      # ; で述語を続ける
    schema:founder ex:kiichiro .

ex:kiichiro a schema:Person ;
    schema:name "豊田喜一郎"@ja .
```

- `@prefix`:長いIRIの短縮(SQLのスキーマ名エイリアスに近い)
- `;` 同じ主語を継続 / `,` 同じ述語で目的語を列挙 / `.` 文の終わり
- `a` は `rdf:type` の略記(「〜は〜である」)

<!--
- Turtleは「読むため」に最頻出の形式。GitHubで公開されている語彙・オントロジーはほぼTurtle
- 記号3つ(; , .)だけ覚えれば読める。声に出して読ませる:「toyotaはCorporationである。nameは日本語でトヨタ自動車…」
- prefix宣言を省略した資料が多いので「ex: や schema: を見たら冒頭のprefixを探す」習慣を伝える
- つまずきポイント:`.` の付け忘れ・`;` と `,` の混同。演習で必ず誰かが踏むので笑い話として予告しておく
-->

---

## JSON-LD:Web API連携で出会うRDF

```json
{
  "@context": "https://schema.org/",
  "@type": "Corporation",
  "@id": "http://example.org/toyota",
  "name": "トヨタ自動車",
  "founder": { "@type": "Person", "name": "豊田喜一郎" }
}
```

- **中身はRDFトリプルと等価**。JSONの皮を被っているだけ
- `@context` がキー名→IRIの対応表(`name` → `schema:name`)
- 出会う場所:**Webページの構造化データ**(SEO)、REST APIのレスポンス、schema.org実装
- 読めればOK。手書きする機会は少ない

<!--
- 「JSONならもう毎日書いてますよね」— JSON-LDはRDF陣営がWeb開発者に歩み寄った形式
- 実務での遭遇率が一番高いのは検索エンジン向け構造化データ(ECサイトの商品情報など)。Googleリッチリザルトの中身がこれ
- @contextの役割だけ押さえれば十分:「ただのJSONキーにグローバルな意味を与える対応表」
- スクレイピングでJSON-LDを拾えば構造化済みデータが手に入る、というKG構築の小技にも触れる(7章の伏線)
-->

---

## RDF vs プロパティグラフ:何が違うか

| 観点 | RDF | プロパティグラフ(Neo4j) |
|---|---|---|
| 基本単位 | トリプル | ノード+リレーション |
| 識別子 | IRI(グローバル一意) | 内部ID+プロパティ(DB内) |
| エッジの属性 | 直接持てない(具体化が必要) | **リレーションにプロパティ可** |
| スキーマ | 語彙・オントロジー(開世界) | ラベル+制約(実質閉世界) |
| クエリ言語 | SPARQL | Cypher |
| 得意領域 | **公開・相互運用・標準化** | **アプリ開発・探索性能** |

> 優劣ではなく**役割分担**:外部との交換はRDF、自社の実装はプロパティグラフ

<!--
- 2章で予告した比較の回収。宗教論争にしないことが大事:「どちらが正しいか」ではなく「どこで使うか」
- エッジ属性の違いは変換時の実務論点:RDFでは「取引(金額付き)」を表すのに中間ノード化(具体化/reification)が要る
- 開世界仮説は6章では深入りしない(5章で説明済みの前提)。「Wikidataに載っていない=偽ではない」という読み方の注意だけ
- 「Neo4j一本でいきたいのになぜ両方…」という不満には:外部知識のパイプを持つ会社と持たない会社でKGの質が変わる、と実利で答える
-->

---

## SPARQL基礎:SQL対比で最短習得

| やりたいこと | SQL | SPARQL |
|---|---|---|
| 取得列の指定 | `SELECT name, date` | `SELECT ?name ?date` |
| データ源 | `FROM companies` | `WHERE { トリプルパターン }` |
| 結合 | `JOIN ... ON ...` | **変数の共有だけで結合される** |
| 条件 | `WHERE price > 100` | `FILTER(?price > 100)` |
| 外部結合 | `LEFT JOIN` | `OPTIONAL { ... }` |
| 集約 | `GROUP BY` + `COUNT` | `GROUP BY` + `COUNT`(同じ) |
| 重複排除・整列 | `DISTINCT` / `ORDER BY` | `DISTINCT` / `ORDER BY`(同じ) |

```sparql
SELECT ?name WHERE {
  ?company a schema:Corporation ;      # 変数?companyがパターンにマッチ
           schema:name ?name .
}
```

<!--
- 「SELECTとWHEREの意味が逆転気味」なのが最初の混乱ポイント:SPARQLのWHEREは条件ではなく「グラフパターン」
- 最大の発想転換は「JOINが消える」こと:同じ変数名を使えば自動的に繋がる。Cypherのパターンマッチと同じ思想
- ?company のように変数は ? で始まる。「マッチするものを全部探す」宣言的クエリ
- 半分は「SQLと同じ」(GROUP BY, ORDER BY, DISTINCT, LIMIT)。新しく覚えるのはトリプルパターンとFILTER/OPTIONALだけ、と負荷を下げる
-->

---

## FILTER / OPTIONAL / 集約の実例

```sparql
# 1990年以降設立の企業と、あれば創業者(なくても企業は出す)
SELECT ?name ?founderName WHERE {
  ?company a schema:Corporation ;
           schema:name ?name ;
           schema:foundingDate ?date .
  FILTER(?date >= "1990-01-01"^^xsd:date)
  OPTIONAL { ?company schema:founder/schema:name ?founderName }
}
```

```sparql
# 業種ごとの企業数(SQLのGROUP BYと同じ感覚)
SELECT ?industry (COUNT(?company) AS ?cnt) WHERE {
  ?company schema:industry ?industry .
} GROUP BY ?industry ORDER BY DESC(?cnt)
```

<!--
- OPTIONALはLEFT JOINそのもの:「創業者情報がない企業も落とさない」。公開KGはデータ欠損が多いのでOPTIONALは必須テク
- OPTIONALを付け忘れると行ごと消える — Wikidataハンズオンでの最頻出トラブルなので強調
- 1つ目の例の schema:founder/schema:name は次スライドのプロパティパスの先出し。「あれ?」と思わせてから次で種明かし
- 集約はSQL経験者なら説明不要なレベル。COUNTのASの位置だけ注意(SELECT句の中で別名を付ける)
-->

---

## プロパティパス:多段探索を1行で

| パス記法 | 意味 | Cypher相当 |
|---|---|---|
| `p1/p2` | p1を辿ってからp2(連結) | `-[:P1]->()-[:P2]->` |
| `p+` | pを1回以上繰り返し | `-[:P*1..]->` |
| `p*` | pを0回以上繰り返し | `-[:P*0..]->` |
| `^p` | pを逆向きに辿る | `<-[:P]-` |

```sparql
# 「情報産業」のサブクラスに(何段でも)属する企業
SELECT ?name WHERE {
  ?company ex:industry/ex:subClassOf* ex:informationIndustry ;
           schema:name ?name .
}
```

<!--
- 4章の可変長パス [:REL*1..3] のSPARQL版。概念はもう知っているので記法の対応だけ
- 実用上いちばん使うのは Wikidataでの wdt:P279*(サブクラス推移閉包)と wdt:P31/wdt:P279*(「〜の一種」判定)。次のWikidataパートで実物を見せる
- SQLの再帰CTEで書いたら何行になるか想像してもらう(3章の実測を思い出させる)
- 注意点:* や + は探索範囲が爆発しうる。公開エンドポイントではタイムアウトの主因になる
-->

---

## Wikidata:世界最大の公開ナレッジグラフ

- **1億超のエンティティ**、多言語、CC0(事実上制約なしで商用利用可)
- SPARQLエンドポイント:`https://query.wikidata.org/`(ブラウザUI付き)
- 独自のID体系(schema.orgではない):

| プレフィックス | 意味 | 例 |
|---|---|---|
| `wd:` | エンティティ(Q番号) | `wd:Q17` = 日本、`wd:Q53268` = トヨタ自動車 |
| `wdt:` | プロパティ(P番号) | `wdt:P112` = 創業者、`wdt:P159` = 本社所在地 |

- Q/P番号の調べ方:Wikidataサイトで検索 → URLの末尾が番号

<!--
- クエリUI(query.wikidata.org)は補完・実行・結果の表やグラフ表示まで揃った優秀な学習環境。ここで練習させる
- wd:とwdt:の使い分けが最初の壁:「wd:はモノ(名詞)、wdt:は関係(動詞)」と覚える
- 正確にはwdt:は「truthyな直接プロパティ」で、修飾子付きの完全形(p:/ps:)は本コースでは扱わない — 深入りしないと宣言
- Q番号の調べ方を実演:Wikidataで「トヨタ」を検索→ Q53268。この「人間が調べてIDを固定する」作業が8章のエンティティリンキングの原型
-->

---

## 【ハンズオン】Wikidata:日本のIT企業と創業者・本社

```sparql
SELECT ?company ?companyLabel ?founderLabel ?hqLabel WHERE {
  ?company wdt:P452 wd:Q11661 ;   # 産業 = 情報技術
           wdt:P17  wd:Q17 .      # 国 = 日本
  OPTIONAL { ?company wdt:P112 ?founder . }   # 創業者
  OPTIONAL { ?company wdt:P159 ?hq . }        # 本社所在地
  SERVICE wikibase:label { bd:serviceParam wikibase:language "ja,en". }
}
LIMIT 100
```

- `SERVICE wikibase:label`:Q番号を日本語ラベルに変換する定型句
- `?companyLabel` のように **変数名+Label** で自動的にラベルが付く
- https://query.wikidata.org/ に貼り付けて実行してみる

<!--
- ここは全員で実際に動かす時間を取る(5分)。結果が返る体験がこの章の山場
- ラベルサービスは「Wikidata方言」だが、これがないと結果が全部Q番号で読めない。定型句として丸暗記でよい
- 結果を見ながら気づきを拾う:創業者が欠けている企業が多い(OPTIONALの効果)、同じ企業が複数行出る(本社・創業者が複数あると直積になる)
- 改造課題を出す:P571(設立日)を追加、FILTER で2000年以降に絞る、COUNTで都道府県別に集計 — 手が速い人向け
- タイムアウトしたらLIMITを付ける・OPTIONALを減らすのが定石と伝える
-->

---

## 外部KGの活用パターン(自社KGを強化する)

| パターン | 内容 | 使う章 |
|---|---|---|
| **エンティティリンキングの参照先** | 「トヨタ」「Toyota Motor」→ `wd:Q53268` に紐付けて名寄せ | 8章 |
| **マスタデータの補完** | 自社KGの企業ノードに業種・所在地・設立日を外部から付与 | 7-8章 |
| **業界標準IDへの接続** | 法人番号・LEI・ISINなど公的IDとの相互変換表として使う | 11章 |

```
(自社KG)  (:Company {name:"トヨタ"})
              │ wikidataId: "Q53268" ← 安定識別子を持たせる
(Wikidata)  wd:Q53268 ─ 業種・本社・法人番号・別名(多言語)…
```

- ポイント:外部KGを**丸ごとコピーしない**。必要な部分だけ、IDで繋ぐ

<!--
- この章の「so what」を示すスライド。RDF/SPARQLを学んだ理由がここで回収される
- エンティティリンキングの威力を具体的に:「トヨタ」の別名(aliases)がWikidataに多言語で登録済み。名寄せ辞書を自前で作らなくてよい
- Wikidataは法人番号(P3225)やLEI(P1278)も持つ。公的IDのハブとして機能する
- アンチパターンを警告:「Wikidata全部インポートしたい」— 鮮度管理もライセンス管理も破綻する。「必要なサブグラフだけ、IDで参照」が原則
- 受講者への問いかけ:「自社のマスタで外部から補完したい項目は何がありますか?」
-->

---

## RDF → Neo4j:変換の考え方

**変換の基本マッピング**

| RDF | プロパティグラフ |
|---|---|
| IRIを持つ主語・目的語 | ノード(IRIは `uri` プロパティに保持) |
| `rdf:type` | ノードラベル |
| 目的語がリテラルのトリプル | ノードのプロパティ |
| 目的語がIRIのトリプル | リレーション |

**2つの実装ルート**

1. **neosemantics(n10s)**:Neo4jプラグインでRDFを直接インポート
2. **SPARQL結果(JSON/CSV)→ Cypherで `MERGE`**:小規模ならこちらが手軽

<!--
- 変換ルールは機械的:「リテラルはプロパティに、IRI同士はエッジに」。この2行で9割説明できる
- IRIをuriプロパティとして必ず残すのが重要:外部KGへの「帰り道」であり、再取り込み時の突合キーになる
- ルート選択の目安:語彙・オントロジーごと取り込むならn10s、SPARQLで絞った結果だけ入れるならCSV/JSON+MERGEで十分
- 演習ではルート2(SPARQL結果→MERGE)を使う。n10sは次スライドで動きを見せるだけに留める
-->

---

## neosemantics(n10s)でRDFを取り込む

```cypher
// 1. 初期化(uri一意制約 + 設定)
CREATE CONSTRAINT n10s_unique_uri
  FOR (r:Resource) REQUIRE r.uri IS UNIQUE;
CALL n10s.graphconfig.init({handleVocabUris: "MAP"});

// 2. Turtleファイルをインポート
CALL n10s.rdf.import.fetch(
  "file:///companies.ttl", "Turtle");

// 3. 結果:RDFのクラスがラベルに、述語がリレーションに
MATCH (c:Corporation)-[:founder]->(p) RETURN c, p;
```

- `handleVocabUris: "MAP"`:`schema:founder` → `founder` のように短縮名に変換
- SPARQLの `CONSTRUCT` 結果をURL指定で直接取り込むことも可能

<!--
- ライブで動かして見せる(用意したcompanies.ttlを使用)。「TurtleがそのままNeo4jのグラフになる」瞬間が伝わればよい
- handleVocabUrisの選択肢:IGNORE(名前空間を捨てる)/ MAP(対応表で変換)/ SHORTEN(prefix付きで保持)。実務はMAPかSHORTEN
- CONSTRUCTクエリ+fetchの組み合わせは「Wikidataから直接Neo4jへ」のパイプになるが、エンドポイントのタイムアウトと相性問題があるため演習では使わない
- n10sの詳細設定は必要になってからでよい。「RDFが来ても慌てない」状態になることが目的
-->

---

## 本コースで深掘りしないもの(必要になったら付録E)

| 技術 | 何をするものか | 深掘りしない理由 |
|---|---|---|
| **OWL推論** | 公理から新事実を自動導出 | LLMソリューションでの費用対効果が低い |
| **SHACL** | RDFグラフの制約検証 | 本コースはNeo4j制約+8章の品質管理で代替 |
| **フェデレーテッドクエリ** | 複数エンドポイント横断検索 | 実務の主流は「取ってきて統合」 |
| **トリプルストア構築** | RDFネイティブDBの運用 | 本コースの主軸はNeo4j |

> 「使う場面が来たら学ぶ」で十分。場面の見分け方と参考資料は**付録E**に

<!--
- 「学ばない」を明示するのはこのコースの設計思想(1章参照)。セマンティックウェブの全体系は広大で、全部やると本題(LLMソリューション)に届かない
- ただし存在は知っておく:顧客や論文がこれらの用語を使ったときに会話できるレベル
- OWL推論に興味を持つ人は多いので一言:「推論器による論理推論と、LLMによる推論は別物。前者は厳密だが表現力の枠内のみ」— 詳細は付録E
- SHACLは「RDF界のスキーマバリデータ」。プロパティグラフ側では8章の品質チェックが同じ役割を担う、と対応づけて安心させる
-->

---

## 【演習】Wikidata → 自社KG(Neo4j)へマージ

**課題:日本のIT企業データをWikidataから取得し、Neo4jにマージする**

1. ハンズオンのSPARQLクエリを改造し、企業・創業者・本社所在地を取得(10分)
   - 結果をJSON(またはCSV)でダウンロード
2. PythonでNeo4jに投入(15分)
   - `MERGE (c:Company {wikidataId: ...})` — **Q番号を突合キーに**
   - 創業者 `(:Person)`、所在地 `(:City)` もノード化しリレーションを張る
3. Neo4j Browserで確認(5分)
   - 「創業者が複数いる企業」「同じ都市に本社を置く企業」をCypherで検索

⏱ 30分(共有・質疑込みで40分)

<!--
- 演習テンプレート(SPARQL雛形+投入用Pythonスクリプト)を配布。ゼロから書かせない
- 最重要ポイントはMERGEのキーをname でなくwikidataId にすること:名前は変わる・重複するがQ番号は安定 — 8章の名寄せの先取り体験
- ありがちなつまずき:SPARQL結果の直積で同一企業が複数行 → MERGEなら自然に吸収される、を体験してもらう
- ステップ3で「Wikidata由来のデータに自社独自ノード(例:自社の取引フラグ)を繋ぐ」拡張を早い人向けに用意
- 共有では「外部KGとの接続キーを持つ自社KG」が完成したことを確認。8章でこのキーを名寄せに使う
-->

---

## 第6章まとめ

- RDFは**外部知識の取り込み手段**:公開KG・標準語彙の世界の共通言語
  - トリプル(主語–述語–目的語)+ IRI + リテラル、読み書きはTurtleで
- SPARQLはSQLの知識で最短習得:**変数共有=JOIN**、`OPTIONAL`=LEFT JOIN
- **Wikidata**は無料で使える1億エンティティの外部マスタ(`wd:` / `wdt:`)
- 活用の型:**丸ごとコピーせず、安定ID(Q番号)で自社KGと接続する**
- RDF→Neo4jは機械的に変換できる(neosemantics / SPARQL結果+MERGE)

**次章**:LLMによるKG自動構築(1)— いよいよ「構築」パート。LLMでテキストからトリプルを抽出する

<!--
- 「基盤パート」はこの章で完了。2〜6章の道具(モデル・DB選定・Cypher・オントロジー・外部知識)が揃ったことを確認
- Q番号による接続は8章のエンティティリンキングで主役になる。「今日作ったwikidataIdプロパティを覚えておいて」
- 次章予告:「ここまでは器の話。次からは中身を自動で作る話。5章で設計したオントロジーがLLMの制御装置として働くのを見ます」
- RDFに深入りしたくなった人には付録EとF(参考資料)を案内して終了
-->
