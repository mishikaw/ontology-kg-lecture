---
marp: true
theme: default
paginate: true
header: "LLM × Knowledge Graph レクチャー"
footer: "第4章 Neo4j / Cypher 実践"
style: |
  section {
    font-size: 24px;
    justify-content: flex-start;
    padding-top: 70px;
  }
  section.title {
    justify-content: center;
    text-align: center;
  }
  h1 { font-size: 40px; color: #1a5276; }
  h2 { font-size: 32px; color: #1a5276; border-bottom: 2px solid #1a5276; padding-bottom: 4px; }
  table { font-size: 20px; }
  code { font-size: 90%; }
  pre { font-size: 18px; line-height: 1.35; }
  blockquote { background: #eaf2f8; border-left: 6px solid #1a5276; padding: 8px 16px; }
  .columns { display: grid; grid-template-columns: 1fr 1fr; gap: 24px; }
  .small { font-size: 18px; }
---

<!-- _class: title -->
<!-- _paginate: false -->

# LLM × Knowledge Graph
## 効果が出るソリューションを作る

### 第4章 Neo4j / Cypher 実践

<!--
- 3章で「Graph DBを使うべき要件」を判断できるようになった。本章はいよいよ手を動かす章
- ゴールは「Cypherを暗記する」ことではなく「SQLの知識をCypherに変換する回路を作る」こと
- 演習比重が高い章。講義はテンポよく進め、手を動かす時間を確保する
- 環境(Docker の Neo4j)が全員起動しているか冒頭で確認しておく
-->

---

## 本章のゴール

- Cypherで**CRUD**(`MATCH` / `CREATE` / `MERGE` / `SET` / `DELETE`)が書ける
- **SQL→Cypher対応表**で既存知識を最短変換する
- 可変長パス・`shortestPath` で**グラフならではのクエリ**を書ける
- インデックス・制約・`EXPLAIN`/`PROFILE` で基本チューニングができる
- **ベクトルインデックス**を張り、9章ハイブリッドGraphRAGの土台を作る
- PythonドライバとLOAD CSVでデータ投入パイプラインを組める

<!--
- 「今日が終わればNeo4jで一通りの開発ができる」と到達点を明確に伝える
- SQL経験者なら習得は速い。対応表方式で「ゼロから学ぶのではない」と安心させる
- ベクトルインデックスは今日は「張って検索できる」まで。活用の本番は9章と予告
- 受講者に質問:Neo4j Browserにログインできた人?(1章の宿題の確認)
-->

---

## Neo4j エコシステムと標準化動向

| 製品 | 用途 |
|---|---|
| **Neo4j Desktop / Docker** | ローカル開発(本コースはDocker) |
| **AuraDB** | マネージドクラウド(無料枠あり) |
| **Neo4j Browser / Bloom** | クエリ実行UI / ビジュアル探索 |

- 他のプロパティグラフDB:**Amazon Neptune**、**Memgraph**、FalkorDB
- **GQL(ISO/IEC 39075:2024)**:グラフクエリ言語の国際標準が成立
  - Cypherが実質的なベース → Cypher習得は標準への投資
- RDB でいえば:Browser ≒ pgAdmin、AuraDB ≒ RDS

<!--
- 「Neo4jにロックインされない?」という不安に先回り:GQL標準化でCypher系スキルは可搬になった
- SQLも各社方言があるのと同じ構図、と RDB 経験に接続する
- Neptune は openCypher 対応、Memgraph も Cypher 互換。学ぶ言語は一つでよい
- 選定比較の詳細は付録Dにあると案内(今日は深入りしない)
-->

---

## Cypher の核:パターン記法と MATCH

**「グラフの絵」をそのままテキストにする**のがCypher

```cypher
//   (ノード)-[リレーション]->(ノード)
//   (c:Customer)-[:ORDERED]->(o:Order)

// 佐藤さんが注文した商品を列挙
MATCH (c:Customer {name: '佐藤'})-[:ORDERED]->(o:Order)-[:CONTAINS]->(p:Product)
RETURN c.name, o.orderId, p.name;
```

- `()` = ノード、`[]` = リレーション、`->` = 方向
- `c:Customer` = 変数`c`、ラベル`Customer`。`{...}` = プロパティ条件
- SQLなら customers / orders / order_items の **2つのJOIN** に相当

<!--
- 「アスキーアートがそのままクエリになる」がCypher最大の特徴。ホワイトボードの絵と一対一対応すると強調
- 変数名は自由(c, cust, x)。ラベル・リレーション型が本体という区別を丁寧に
- 同じクエリをSQLで書くと何行になるか受講者に想像させてから次の対応表へ繋ぐ
- 方向を逆にすると結果が空になることをその場で実演すると記憶に残る(方向の設計は後半で扱う)
-->

---

## CREATE / SET / DELETE

```cypher
-- 作成(SQL: INSERT)
CREATE (p:Product {sku: 'P-100', name: 'キーボード', price: 8800});

-- 更新(SQL: UPDATE)
MATCH (p:Product {sku: 'P-100'})
SET p.price = 7980, p.updatedAt = datetime();

-- 削除(SQL: DELETE)
MATCH (p:Product {sku: 'P-100'})
DETACH DELETE p;   -- リレーションごと削除
```

- `DELETE` はリレーションが残っていると**エラー**(RDBの外部キー制約に相当)
- `DETACH DELETE` = 「繋がっているエッジも一緒に消す」

<!--
- CREATE は実行するたびに新しいノードを作る(重複チェックなし)ことを必ず強調。次のMERGEへの伏線
- DETACH DELETE は便利だが本番では危険。「ON DELETE CASCADE を毎回書いているようなもの」とRDB語彙で警告
- スキーマレスなので存在しないプロパティにSETしてもエラーにならない。typoに気づきにくい点を注意喚起
- 全消しの定番 `MATCH (n) DETACH DELETE n` は演習環境リセット用として紹介(本番禁止)
-->

---

## MERGE:冪等な書き込み(UPSERT)

```cypher
MERGE (p:Product {sku: 'P-100'})
ON CREATE SET p.name = 'キーボード', p.createdAt = datetime()
ON MATCH  SET p.updatedAt = datetime();
```

- 「**なければ作る、あれば使う**」— SQLの `INSERT ... ON CONFLICT` に相当
- データ投入・再実行・LLM抽出結果の反映は**基本 MERGE**(7章で多用)

**⚠️ MERGEの罠:パターン全体が単位**

```cypher
-- 佐藤ノードが既にあっても、この「パターン全体」が無ければ
-- 佐藤ノードが“もう1個”作られてしまう!
MERGE (c:Customer {name: '佐藤'})-[:ORDERED]->(o:Order {orderId: 'O-1'});
```

<!--
- 本章最大のつまずきポイント。MERGEは「パターン全体をマッチ→無ければ全体を作成」であり部分再利用しない
- 罠の実例をライブで:先にCustomerだけ作っておき、上のMERGEを実行→佐藤が2人になるのを見せる
- 正解パターンは「ノードを個別にMERGEしてからリレーションをMERGE」:
  MERGE (c:Customer {name:'佐藤'}) MERGE (o:Order {orderId:'O-1'}) MERGE (c)-[:ORDERED]->(o)
- MERGEのキーには一意性制約を張るのが鉄則(後述のスライドで回収)
- 7章でLLM抽出結果を流し込むときにこの罠を踏むと重複ノードだらけになる、と実害を予告
-->

---

## SQL → Cypher 対応表(最短習得の地図)

| やりたいこと | SQL | Cypher |
|---|---|---|
| 検索 | `SELECT ... FROM ... WHERE` | `MATCH ... WHERE ... RETURN` |
| **結合** | `JOIN ... ON ...`(N回繰り返し) | **パターンマッチ** `(a)-[:R]->(b)` |
| 集約 | `GROUP BY` + `COUNT/SUM` | `RETURN` 内の集約(**暗黙のグルーピング**) |
| **再帰** | `WITH RECURSIVE`(再帰CTE) | **可変長パス** `[:R*1..3]` |
| 挿入 | `INSERT` | `CREATE` / `MERGE` |
| 更新 | `UPDATE ... SET` | `SET` |
| 削除 | `DELETE` | `DELETE` / `DETACH DELETE` |
| 整列・件数 | `ORDER BY` / `LIMIT` | 同じ(`ORDER BY` / `LIMIT` / `SKIP`) |
| 中間結果 | サブクエリ / CTE(`WITH`) | `WITH`(パイプライン) |

<!--
- この1枚が本章の地図。印刷して手元に置く価値がある(付録Cのチートシートにも収録)
- 最重要は「JOIN→パターンマッチ」と「再帰CTE→可変長パス」の2行。3章の実測比較を思い出させる
- 集約の「暗黙のグルーピング」:RETURN c.name, count(o) と書けば非集約列が自動的にグルーピングキーになる。GROUP BY句は存在しない
- Cypher の WITH は SQL の CTE に相当し「クエリを段階に区切るパイプ」。次の集約例で実演
-->

---

## 集約と WITH:GROUP BY の代わり

```cypher
-- 顧客ごとの注文数トップ5(GROUP BY 相当)
MATCH (c:Customer)-[:ORDERED]->(o:Order)
RETURN c.name, count(o) AS orders
ORDER BY orders DESC LIMIT 5;

-- WITH で段階処理:10回以上注文した顧客の購入商品カテゴリ
MATCH (c:Customer)-[:ORDERED]->(o:Order)
WITH c, count(o) AS orders
WHERE orders >= 10                    -- HAVING に相当
MATCH (c)-[:ORDERED]->()-[:CONTAINS]->(p:Product)
RETURN c.name, collect(DISTINCT p.category);
```

- `count` / `sum` / `avg` / `collect`(リスト集約)
- `WITH` + `WHERE` = SQLの `HAVING`

<!--
- collect() はSQLにない強力な武器(array_agg相当)。「顧客ごとの商品リスト」が1クエリで返る
- WITH の後ろに書いた変数しか次の段に渡らない(SELECT句で絞るのと同じ)点はつまずきやすい
- 「HAVINGはどう書く?」と受講者に問いかけてから WITH+WHERE を見せると定着しやすい
- 2つ目のクエリを口頭で日本語に翻訳させてみる:読めれば書ける
-->

---

## 可変長パスと最短経路

```cypher
-- サーバS1に(1〜3ホップで)依存する全サービス:再帰CTE不要
MATCH (s:Server {name: 'S1'})<-[:DEPENDS_ON*1..3]-(svc:Service)
RETURN DISTINCT svc.name;

-- AさんとBさんの最短の繋がり(6ホップまで)
MATCH p = shortestPath(
  (a:Person {name: 'A'})-[:KNOWS*..6]-(b:Person {name: 'B'}))
RETURN [n IN nodes(p) | n.name] AS chain, length(p);
```

- `*1..3` = 1〜3ホップ、`*` = 無制限(**本番では上限必須**)
- `shortestPath` / `allShortestPaths`:組織分析・影響波及・不正調査の定番
- 3章の再帰CTE(数十行)がここまで縮む — Graph DB採用の主因

<!--
- 3章で書いたPostgreSQLの再帰CTEを画面に並べて見せると効果絶大。「あれがこの1行」
- 上限なし `*` は探索爆発の危険。実務では必ず上限を切る+方向を付けると計面が激減する、と性能面の注意
- shortestPath の質問例:「この2人はどう繋がっている?」はコンプラ調査・営業のリファラル探しで頻出
- リレーション方向を省略(`-`)すると双方向探索になることに触れる
- 演習の影響分析クエリはこの可変長パスを使う、と予告
-->

---

## モデリング実践(1):中間ノードパターン

「注文」をリレーションにするか、ノードにするか?

<div class="columns">
<div>

**リレーション案(簡潔)**

```
(顧客)-[:PURCHASED
   {date, qty}]->(商品)
```

- 購入という**事実そのもの**に他の関係を張れない

</div>
<div>

**中間ノード案(拡張可能)**

```
(顧客)-[:ORDERED]->(注文)
(注文)-[:CONTAINS]->(商品)
(注文)-[:SHIPPED_BY]->(配送業者)
```

- 注文に配送・決済・キャンセルを**後から接続できる**

</div>
</div>

> 判断基準:**その関係に、さらに別の関係が繋がりうるか?**

<!--
- 2章のEC演習で迷ったポイントの回収。RDBの中間テーブル(order_items)と同じ発想で腹落ちさせる
- 「リレーションには別のリレーションを張れない」というプロパティグラフの制約が判断の根拠
- 迷ったら中間ノード寄りに倒すのが安全(後からの分解は大変、統合は簡単)
- 受講者に問いかけ:「社員の部署異動履歴はどっちで作る?」→ 異動イベントノードが答えの一例(次スライドの時系列へ繋ぐ)
-->

---

## モデリング実践(2):時系列とリレーション方向

**時系列の扱い**

| 方式 | 例 | 向く場面 |
|---|---|---|
| リレーションに期間 | `[:WORKS_AT {from, to}]` | 履歴が単純・少量 |
| イベントノード | `(異動 {date})-[:FROM]->(部署)` | 履歴自体を分析したい |

**方向の設計**

- 方向は**1本だけ**張る:`(社員)-[:BELONGS_TO]->(部署)`
  - 逆向きの `HAS_MEMBER` を重ねない(二重管理・不整合の元)
- Cypherは逆方向にも辿れる:`(d:Dept)<-[:BELONGS_TO]-(e)`

<!--
- 「双方向に張りたくなる」は初学者の典型ミス。クエリは方向を無視も逆走もできるので1本で足りる
- 例外的に両方向を張るのは意味が非対称な場合のみ(FOLLOWS など)
- 時系列は「現在の状態だけでよいか、履歴を問うCQがあるか」で決める。5章のコンピテンシークエスチョン駆動の考え方を先出し
- 命名の慣習:リレーションは動詞・大文字スネークケース(WORKS_AT)。5章の命名規約で再登場
-->

---

## インデックスと一意性制約

```cypher
-- 一意性制約(自動的にインデックスも張られる)
CREATE CONSTRAINT product_sku IF NOT EXISTS
FOR (p:Product) REQUIRE p.sku IS UNIQUE;

-- 検索用インデックス
CREATE INDEX customer_name IF NOT EXISTS
FOR (c:Customer) ON (c.name);

SHOW CONSTRAINTS;  SHOW INDEXES;
```

- **MERGEのキーには一意性制約が必須**(重複防止 + 起点検索の高速化)
- インデックスが効くのは**探索の起点**のみ。辿り(トラバース)はindex-free adjacencyで元々速い

<!--
- RDBとの違いの核心:「JOINのためのインデックス」は不要。必要なのは起点ノードを見つけるためのインデックスだけ
- MERGE×制約なしは「重複する上に遅い」最悪の組み合わせ。データ投入前に制約を張る習慣を徹底
- 制約違反時はエラーになる=データ品質のガードレールとしても機能(8章の品質管理に繋がる)
- 複合インデックス・全文インデックスもあると紹介だけ(全文は9章で軽く再登場)
-->

---

## EXPLAIN / PROFILE でチューニング

```cypher
PROFILE
MATCH (c:Customer {name: '佐藤'})-[:ORDERED]->(o:Order)
RETURN count(o);
```

| 演算子 | 意味 | 対処 |
|---|---|---|
| `NodeByLabelScan` | ラベル全件走査(**遅い**) | 起点プロパティにインデックス |
| `NodeIndexSeek` | インデックス起点(**速い**) | ✅ 目標の形 |
| `CartesianProduct` | 直積(**危険**) | パターンを繋げ忘れていないか |

- `EXPLAIN` = 実行せず計画のみ / `PROFILE` = 実行して **db hits** を実測
- SQLの実行計画の読み方がほぼそのまま通用する

<!--
- 「まずPROFILEを付けて db hits を見る」を習慣化させる。数字が減れば改善、増えれば改悪という単純な指標
- NodeByLabelScan は RDB の Seq Scan に相当、と対応づければ一瞬で伝わる
- CartesianProduct は MATCH を2つ書いてパターンを繋ぎ忘れたときに出がち。警告も出るので見逃さない
- ライブデモ:インデックスなし→PROFILE→制約作成→PROFILE で db hits の激減を見せる
-->

---

## ベクトルインデックス(1):embedding を格納する

Neo4j は**ベクトルDBとしても使える**(5.x〜)

```cypher
-- 商品説明文の embedding 用ベクトルインデックス
CREATE VECTOR INDEX product_embedding IF NOT EXISTS
FOR (p:Product) ON (p.embedding)
OPTIONS {indexConfig: {
  `vector.dimensions`: 1536,
  `vector.similarity_function`: 'cosine'
}};

-- embedding はプロパティとして格納(Pythonから設定)
MATCH (p:Product {sku: 'P-100'})
SET p.embedding = $vector;   -- floatの配列
```

<!--
- 「グラフDBなのにベクトル?」という疑問に:9章のハイブリッドGraphRAGは「ベクトルで入口を見つけ、グラフで展開する」構成。同一DBで完結できるのが強み
- 次元数は使うembeddingモデルに合わせる(例:OpenAI text-embedding-3-small = 1536)
- embedding の生成自体はLLM API側の仕事。Neo4jは格納と検索を担当、という役割分担を明確に
- ベクトル専用DB(Pinecone等)との使い分けを聞かれたら:グラフ展開が要件ならNeo4j一本化が運用簡単、と11章の議論を予告
-->

---

## ベクトルインデックス(2):類似検索 + グラフ展開

```cypher
-- 質問文のベクトルで類似商品を検索し、そのままグラフを辿る
CALL db.index.vector.queryNodes('product_embedding', 5, $queryVector)
YIELD node AS p, score
MATCH (p)<-[:CONTAINS]-(:Order)<-[:ORDERED]-(c:Customer)
RETURN p.name, score, count(DISTINCT c) AS buyers
ORDER BY score DESC;
```

- 1つのクエリで **類似検索 → 購入者の集計** まで到達
- これが **9章ハイブリッドGraphRAG** の心臓部:
  - ベクトル検索 = **エントリポイント発見**
  - グラフ探索 = **関係の展開・根拠の提示**

<!--
- この1クエリが「ベクトルDB + グラフDB を別々に持つと2往復かかる処理」であることを強調
- YIELD で受けた node がそのまま MATCH のパターンに入る、という接続部分がキモ
- 「類似検索の結果に、なぜ・誰が・どう繋がるかの文脈を足せる」= ベクトルRAGにない説明可能性
- 今日は仕組みの確認まで。実際にLLMと組み合わせて動かすのは9章、と期待値を設定
-->

---

## Python からの操作:公式ドライバ

```python
from neo4j import GraphDatabase

URI, AUTH = "bolt://localhost:7687", ("neo4j", "password")

with GraphDatabase.driver(URI, auth=AUTH) as driver:
    records, summary, keys = driver.execute_query(
        """
        MATCH (c:Customer {name: $name})-[:ORDERED]->(o:Order)
        RETURN count(o) AS orders
        """,
        name="佐藤", database_="neo4j",
    )
    print(records[0]["orders"])
```

- パラメータは**必ず `$name` 形式**(f-string連結はインジェクションの温床)
- `execute_query` が再試行・接続管理込みの推奨API(v5〜)

<!--
- 「psycopg2でSQLを叩くのと同じ感覚」でOK。ドライバの流儀だけ押さえる
- パラメータ化は性能面でも重要:クエリプランがキャッシュされる(SQLのプリペアドステートメントと同じ)
- f-stringでCypherを組み立てるコードは7章以降のLLM連携で事故の元。今日から禁止と宣言
- 旧来の session.run / トランザクション関数のコードも世間には多い。読めれば良い、書くのは execute_query で統一
-->

---

## バルクロード:LOAD CSV

```cypher
LOAD CSV WITH HEADERS FROM 'file:///orders.csv' AS row
CALL (row) {
  MERGE (c:Customer {customerId: row.customer_id})
  MERGE (o:Order    {orderId: row.order_id})
    ON CREATE SET o.orderedAt = date(row.ordered_at)
  MERGE (c)-[:ORDERED]->(o)
} IN TRANSACTIONS OF 1000 ROWS;
```

- `IN TRANSACTIONS`:1000行ごとにコミット(メモリ枯渇防止)
- CSVの値は**すべて文字列** → `toInteger()` / `date()` で変換
- 事前に**一意性制約を張ってから**実行(MERGE高速化)
- 数千万行級は `neo4j-admin database import` を使う

<!--
- 「RDBからのエクスポートCSV→Neo4j」が実務の最頻パターン。3章で話した併用アーキテクチャの同期手段の一つ
- ありがちな失敗:制約なしでLOAD CSV→MERGEが全件スキャンになり数時間コース。順序が大事
- 型変換忘れで price が文字列として入り、集計が壊れる事故も定番。投入後にサンプル確認する習慣を
- MERGEの罠(パターン全体)がここでも効いてくる:ノードごとにMERGEしている書き方に注目させる
- 演習はこの雛形をベースに進める、と繋ぐ
-->

---

## 【演習】ECデータの投入と分析クエリ

2章でモデリングしたECデータ(顧客・注文・商品・カテゴリ)を使う:

1. **制約を作成し、配布CSVを `LOAD CSV` で投入**(10分)
   - Customer / Order / Product + ORDERED / CONTAINS
2. **共同購買推薦クエリ**(15分):顧客C001への「あなたへのおすすめ」
   - C001と同じ商品を買った顧客が、**他に**買っている商品を人気順に
   - ヒント:`(me)→商品←(他人)→別商品`、`WHERE NOT (me)-...->(rec)`
3. **影響分析クエリ**(10分):商品P-100が回収になった
   - 購入した顧客と、その顧客の**未発送の注文**を可変長パスで列挙
4. 結果共有 + `PROFILE` でdb hits比較(10分)

⏱ 45分

<!--
- 推薦クエリの模範解答の骨子:MATCH (me:Customer {customerId:'C001'})-[:ORDERED]->()-[:CONTAINS]->(p)<-[:CONTAINS]-()<-[:ORDERED]-(other), (other)-[:ORDERED]->()-[:CONTAINS]->(rec) WHERE me<>other AND NOT (me)-[:ORDERED]->()-[:CONTAINS]->(rec) RETURN rec.name, count(*) ORDER BY count(*) DESC
- つまずき予想:①制約を張り忘れて投入が遅い ②MERGEのパターン単位の罠で顧客が重複 ③推薦でme自身を除外し忘れる
- 早く終わった人向け:推薦を「同カテゴリに限定」「直近90日に限定」に拡張させる
- 共有パートでは同じ結果を出す別解を並べ、PROFILEでdb hitsを比較して「速いクエリの形」を体感させる
- 「これがSQLだと何行か」を最後に一度想像させ、4章の学習効果を実感させて締める
-->

---

## 第4章まとめ

- Cypherは**グラフの絵をそのまま書く**言語 — JOINはパターンマッチに
- **MERGE**が実務の主役。ただし**パターン全体が単位**という罠に注意
- 再帰CTE → **可変長パス `[:REL*1..3]`** / `shortestPath` で激減
- 起点に**インデックスと一意性制約**、`PROFILE` で **db hits** を見る
- **ベクトルインデックス**でembedding格納+類似検索 → 9章の土台
- Pythonドライバ(`execute_query` + パラメータ)、`LOAD CSV` で投入

**次章**:LLM時代のオントロジー設計 — 「何をノードにするか」を場当たりでなく、**LLMを制御するスキーマ**として設計する

<!--
- 手が動くようになった今こそ設計の話ができる、という流れで5章へ
- 次章予告:「今日の演習で『何をノードにするか』を各自の勘で決めた。次章はそれを再現可能な設計手順にする。しかもそのスキーマがLLMの制御装置になる」
- MERGEの罠は7章(LLM抽出結果の投入)で全員がもう一度出会う。今日の記憶を維持するよう促す
- 質問受付+休憩。環境で詰まった人は演習ファイルを持ち帰れるとフォロー
-->
