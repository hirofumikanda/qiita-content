---
title: 'PostGISの空間インデックスと統計情報（ANALYZE）の基本に立ち返る'
tags:
  - PostgreSQL
  - PostGIS
  - Database
  - 空間データベース
private: false
updated_at: '2026-01-03T00:00:00+09:00'
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

PostGISは、PostgreSQLに地理空間データの処理機能を追加する拡張機能です。大量の地理空間データを効率的に扱うためには、適切なインデックスと統計情報の管理が必要となります。

本記事では、PostGISにおける空間インデックスと統計情報の更新の効果をつぶさに検証し、本来の性能を発揮するためにこれらをどのように管理すべきかを検討したいと思います。

なお、基本的な内容のみを取り上げており、細かなパラメータの調整などについては言及していないため、PostgreSQLをまだ使い慣れていない方向けであることを先に断っておきます。

## PostGISの空間インデックスとは

空間インデックスは、地理空間データのクエリを高速化するためのデータ構造です。

通常のB-treeインデックスは1次元データに最適化されていますが、地理空間データは2次元（または3次元）であるため、専用のインデックス構造が必要になります。

主に利用されるインデックスは**GiST（Generalized Search Tree）** インデックスです。これは、R-treeアルゴリズムに基づいており、2次元の**バウンディングボックス（MBR: Minimum Bounding Rectangle）** を使用してデータを階層的に管理します。

仕組みの詳細はPostGISドキュメントの[どのように空間インデックスが動作するか](https://postgis.net/workshops/ja/postgis-intro/indexing.html#how-spatial-indexes-work)を参照ください。

### 空間インデックスを作成してみる

まずは、空間インデックスがどのように動作するか確認してみたいと思います。

簡単にテストするため、ポイントデータを扱います。

テスト環境は以下のとおりです。

- Windows PC（Windows 11 Pro、Intel N95 1.7GHz、メモリ16GB）
- PostgreSQL 16.11

最初に、テスト用のテーブルを作成します。

```sql
CREATE TABLE large_poi (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    geom GEOMETRY(Point, 4326)
);
```

このテーブルにテスト用のダミーデータを挿入します。空間インデックスの効果を検証するため、ランダムに1000万個のポイントデータを入れておきます。

```sql
INSERT INTO large_poi (name, geom)
SELECT 
    'POI_' || i,
    ST_SetSRID(
        ST_MakePoint(
            139.0 + random() * 2,  -- 経度: 139.0～141.0
            35.0 + random() * 2     -- 緯度: 35.0～37.0
        ), 
        4326
    )
FROM generate_series(1, 10000000) AS i;

-- Query returned successfully in 1 min 19 secs.
```

このポイントデータのうち、ある地点（140.0, 36.0）から0.1度の範囲にあるポイントを抽出することを考えます。
空間インデックスなしでこの検索を実施すると、シーケンシャルスキャンになり、非常に時間がかかることが容易に想像できるかと思います。

そのため、空間インデックスを作成します。後で考察に使うためサイズも調べておきます。

```sql
CREATE INDEX idx_large_poi_geom ON large_poi USING GIST (geom);

-- Query returned successfully in 41 secs 257 msec.

SELECT
  pg_size_pretty(pg_relation_size('idx_large_poi_geom'));

-- 396 MB
```

ANALYZEもしておきます（ANALYZEについては、のちほど取り上げます）。

```sql
ANALYZE large_poi;

-- Query returned successfully in 1 secs 11 msec.
```

準備が整ったので、上記の検索を実行します。インデックスの利用状況が見たいので、EXPLAIN ANALYZEします。なお、キャッシュの効果を除去するため、2回実施して、2回目の結果のみをみることにします（以下同様）。

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM large_poi
WHERE ST_DWithin(geom, ST_SetSRID(ST_MakePoint(140.0, 36.0), 4326), 0.1);

"Gather  (cost=3713.66..610319.81 rows=1000 width=47) (actual time=116.882..774.816 rows=78692 loops=1)"
"  Workers Planned: 2"
"  Workers Launched: 2"
"  Buffers: shared read=62205"
"  ->  Parallel Bitmap Heap Scan on large_poi  (cost=2713.66..609219.81 rows=417 width=47) (actual time=38.916..582.260 rows=26231 loops=3)"
"        Filter: st_dwithin(geom, '0101000020E610000000000000008061400000000000004240'::geometry, '0.1'::double precision)"
"        Rows Removed by Filter: 7146"
"        Heap Blocks: exact=28879"
"        Buffers: shared read=62205"
"        ->  Bitmap Index Scan on idx_large_poi_geom  (cost=0.00..2713.41 rows=97716 width=0) (actual time=90.296..90.296 rows=100130 loops=1)"
"              Index Cond: (geom && st_expand('0101000020E610000000000000008061400000000000004240'::geometry, '0.1'::double precision))"
"              Buffers: shared read=556"
"Planning Time: 0.417 ms"
"Execution Time: 780.803 ms"
```

1sもかからずに完了しました。

`Index Cond: (geom && st_expand('0101000020E610000000000000008061400000000000004240'::geometry, '0.1'::double precision))`の部分がMBRを使った交差チェックを行っているところです。

これだけでも、空間インデックスが非常に強力なツールであることが見て取れるかと思います。

### 空間インデックスは即時更新されるのか

さて、この空間インデックスはいつ更新されるのでしょうか。

PostgreSQLの[ドキュメント](https://www.postgresql.jp/document/16/html/indexes-intro.html)には以下の記載があります。

「**いったんインデックスを作成すれば、それ以上の処理は必要はありません。** システムは、**テーブルが変更される時インデックスを更新**し、シーケンシャルスキャンよりもインデックススキャンを行うことがより効率的と判断した場合、問い合わせでインデックスを使用します。」

実際にこの記載のとおりインデックスが即時更新されるか確認してみたいと思います。

先ほど挿入したテストデータを別のテーブルに退避します。

```sql
CREATE TABLE large_poi2 AS SELECT * FROM large_poi;

-- Query returned successfully in 23 secs 592 msec.
```

いったん、元のテーブルからレコードを削除してから、退避しておいたデータを使って再度入れなおします。

```sql
TRUNCATE large_poi;

INSERT INTO large_poi SELECT * FROM large_poi2;

-- Query returned successfully in 5 min 47 secs.
```

ANALYZEした上で、再度検索を実施します。

```sql
ANALYZE large_poi;

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM large_poi
WHERE ST_DWithin(geom, ST_SetSRID(ST_MakePoint(140.0, 36.0), 4326), 0.1);

"Gather  (cost=3816.49..626617.33 rows=1000 width=47) (actual time=176.758..876.545 rows=78692 loops=1)"
"  Workers Planned: 2"
"  Workers Launched: 2"
"  Buffers: shared hit=20 read=62197 written=70"
"  ->  Parallel Bitmap Heap Scan on large_poi  (cost=2816.49..625517.33 rows=417 width=47) (actual time=58.775..696.053 rows=26231 loops=3)"
"        Filter: st_dwithin(geom, '0101000020E610000000000000008061400000000000004240'::geometry, '0.1'::double precision)"
"        Rows Removed by Filter: 7146"
"        Heap Blocks: exact=26652"
"        Buffers: shared hit=20 read=62197 written=70"
"        ->  Bitmap Index Scan on idx_large_poi_geom  (cost=0.00..2816.24 rows=100760 width=0) (actual time=127.542..127.542 rows=100130 loops=1)"
"              Index Cond: (geom && st_expand('0101000020E610000000000000008061400000000000004240'::geometry, '0.1'::double precision))"
"              Buffers: shared read=568"
"Planning Time: 0.401 ms"
"Execution Time: 882.525 ms"
```

問題なく空間インデックスが使われているように見えます。
念のため、インデックスのサイズも確認してみます。

```sql
SELECT
  pg_size_pretty(pg_relation_size('idx_large_poi_geom'));

-- 399 MB
```

当初作成された空間インデックスとほぼ同じサイズになっています。

ここから、TRUNCATEでデータを空にして再度1000万レコードを入れなおす操作をしても、即時に空間インデックスが更新された状態になっていることがわかります。

したがって、新規にレコードを挿入しても、インデックスを作成しなおす必要は基本的にないといえます。

### 大量レコード挿入時には注意が必要

ただし、上記の一連の操作でかかった時間に着目していただくとわかるとおり、インデックスがある状態で大量のレコードを挿入すると、非常に時間がかかります。

後でインデックスを作成する場合と比較すると以下のとおりです。

```
- レコードを挿入した後でインデックスを作成した場合
→ レコード挿入にかかった時間（1min 19sec） ＋ インデックス作成にかかった時間（41sec）
→ 2min

- インデックスがある状態でレコードを挿入した場合
→ 5min 47sec
```

そのため、大量のデータをロードする場合は、いったんインデックスを削除して、データロード完了後にインデックスを再作成した方が効率的です。

### ANALYZEを手動実行する必要があるか

次に、ANALYZEについても調査してみます。

PostgreSQLのクエリプランナーは、統計情報を使用して最適な実行計画を選択します。PostGISの空間データについても、ANALYZEを実行することで統計情報を収集・最新化することができます。

ANALYZEの効果を確認するため、テスト用のテーブルに再度データを入れなおします。

```sql
TRUNCATE large_poi;

INSERT INTO large_poi SELECT * FROM large_poi2;
```

今度は、あえてANALYZEせずに検索を実行してみます。

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM large_poi
WHERE ST_DWithin(geom, ST_SetSRID(ST_MakePoint(140.0, 36.0), 4326), 0.1);

"Index Scan using idx_large_poi_geom on large_poi  (cost=0.54..1538575.33 rows=1009 width=47) (actual time=27.784..2533.280 rows=78692 loops=1)"
"  Index Cond: (geom && st_expand('0101000020E610000000000000008061400000000000004240'::geometry, '0.1'::double precision))"
"  Filter: st_dwithin(geom, '0101000020E610000000000000008061400000000000004240'::geometry, '0.1'::double precision)"
"  Rows Removed by Filter: 21438"
"  Buffers: shared hit=17263 read=83331 written=14675"
"Planning Time: 0.386 ms"
"Execution Time: 2548.065 ms"
```

実行計画がさきほどインデックスの検証で実施した時と変わっていることに気づきましたでしょうか。

念のため、ANALYZEして再度検索してみます。

```sql
ANALYZE large_poi;

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM large_poi
WHERE ST_DWithin(geom, ST_SetSRID(ST_MakePoint(140.0, 36.0), 4326), 0.1);

"Gather  (cost=3423.01..550886.86 rows=1000 width=47) (actual time=150.743..779.441 rows=78692 loops=1)"
"  Workers Planned: 2"
"  Workers Launched: 2"
"  Buffers: shared hit=19 read=62196 written=75"
"  ->  Parallel Bitmap Heap Scan on large_poi  (cost=2423.01..549786.86 rows=417 width=47) (actual time=50.626..594.158 rows=26231 loops=3)"
"        Filter: st_dwithin(geom, '0101000020E610000000000000008061400000000000004240'::geometry, '0.1'::double precision)"
"        Rows Removed by Filter: 7146"
"        Heap Blocks: exact=26414"
"        Buffers: shared hit=19 read=62196 written=75"
"        ->  Bitmap Index Scan on idx_large_poi_geom  (cost=0.00..2422.76 rows=86696 width=0) (actual time=98.713..98.714 rows=100130 loops=1)"
"              Index Cond: (geom && st_expand('0101000020E610000000000000008061400000000000004240'::geometry, '0.1'::double precision))"
"              Buffers: shared read=566"
"Planning Time: 0.373 ms"
"Execution Time: 785.367 ms"
```

両者を比較するとわかるとおり、明らかに実行計画に変化が生じており、実行時間にも3倍の差が出ています。

ANALYZEする前は、単一のプロセスで行ごとのIndex Scanを実施していますが、ANALYZEした後は2つの子プロセスを起動して3プロセス並列でBitmap Scanを実施しています。

これは、ANALYZEで統計情報を更新したことで、行数の見積もりが実際の値に近づき、行ごとに処理するよりもBitmapで一括処理した方が早い、かつ複数プロセスで並列処理した方が早いとクエリプランナーが判断した結果と考えられます。

以上のことからもわかるように、**特に大量のレコードを挿入・更新・削除した場合は、正確な実行計画をクエリプランナーに算定してもらうために明示的にANALYZEしておくことが肝要です。**

なお、ANALYZE時に詳細情報も表示したい場合は、以下のように`VERBOSE`をつけます。

```sql
ANALYZE VERBOSE large_poi;

--INFO:  "public.large_poi"を解析しています
--INFO:  "large_poi": 93458ページの内30000をスキャン。3209983の有効な行と0の不要な行が存在。30000行をサンプリング。推定総行数は9999953
```

### 自動バキューム機能

PostgreSQLには、[自動バキューム機能](https://www.postgresql.jp/document/16/html/routine-vacuuming.html#AUTOVACUUM)があり、テーブルの更新状況に応じて、VACUUMとANALYZEを自動実行してくれます。

自動バキューム機能を有効にするには、autovacuumパラメータ及びtrack_countsパラメータが有効(on)になっている必要があります。

有効になっているかは、以下で確認できます。

```sql
SHOW autovacuum;
-- on

show track_counts;
-- on
```

有効になっていれば、自動バキュームデーモンは、autovacuum_naptimeパラメータ値に設定された時間間隔で各テーブルの更新状況をチェックします。

```sql
show autovacuum_naptime;
-- 1min
```

上記の場合は、1分ごとにチェックします。

公式ドキュメントの[自動バキュームデーモン](https://www.postgresql.jp/document/16/html/routine-vacuuming.html#AUTOVACUUM)の説明に記載があるとおり、更新状況のチェックの結果、前回のANALYZEの後に挿入、更新、削除されたタプル数が解析閾値を超える場合に対象テーブルのANALYZEを自動実行します。解析閾値は以下の式で計算されます。

```
解析閾値 = 解析基礎閾値 + 解析規模係数 * タプル数
```

それぞれのパラメータは以下で確認できます。

```sql
-- 解析基礎閾値
SHOW autovacuum_analyze_threshold;
-- 50

-- 解析規模係数
SHOW autovacuum_analyze_scale_factor;
-- 0.1
```

例えば、100行更新した場合は

```
解析閾値 = 50 + 0.1 * 100 = 60
```

で解析閾値を超えるため、自動ANALYZEされます。

それでは、実際に自動ANALYZEされるか試してみます。

さきほどと同様にデータを入れなおします。

```sql
TRUNCATE large_poi;

INSERT INTO large_poi SELECT * FROM large_poi2;
```

しばらく待って自動ANALYZEされたかを確認します。以下のSQLで最終実行日時を確認できます。

```sql
SELECT
  last_autoanalyze
FROM pg_stat_user_tables
WHERE relname = 'large_poi';
```

自動ANALYZEが実行されていることを確認できたら、検索を実行します。

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM large_poi
WHERE ST_DWithin(geom, ST_SetSRID(ST_MakePoint(140.0, 36.0), 4326), 0.1);

"Gather  (cost=3740.51..612371.49 rows=1000 width=47) (actual time=155.199..926.716 rows=78692 loops=1)"
"  Workers Planned: 2"
"  Workers Launched: 2"
"  Buffers: shared read=62222"
"  ->  Parallel Bitmap Heap Scan on large_poi  (cost=2740.51..611271.49 rows=417 width=47) (actual time=51.535..709.616 rows=26231 loops=3)"
"        Filter: st_dwithin(geom, '0101000020E610000000000000008061400000000000004240'::geometry, '0.1'::double precision)"
"        Rows Removed by Filter: 7146"
"        Heap Blocks: exact=28003"
"        Buffers: shared read=62222"
"        ->  Bitmap Index Scan on idx_large_poi_geom  (cost=0.00..2740.26 rows=98096 width=0) (actual time=103.951..103.951 rows=100130 loops=1)"
"              Index Cond: (geom && st_expand('0101000020E610000000000000008061400000000000004240'::geometry, '0.1'::double precision))"
"              Buffers: shared read=573"
"Planning Time: 0.554 ms"
"Execution Time: 939.975 ms"
```

Bitmap index scanの想定行数が98096になっており、先ほど明示的にANALYZEした後の値（86696）に近い値が出ています。このことから、自動ANALYZEでも手動でANALYZEした場合と同等の効果があることがわかります。

ただし、いつも自動ANALYZEされるわけではない（解析閾値を超えている場合に限る）ことと、更新後すぐに自動ANALYZEされるわけではない（チェックはautovacuum_naptimeパラメータ値の時間間隔）ため、自動バキューム機能が有効になっている場合でも、**大量レコードの更新直後に同一テーブルに対してクエリを発行する場合は、手動ANALYZEするべき**といえます。

## まとめ

以上の議論をまとめると以下のとおりです。

### 空間インデックス

- 一定数以上のジオメトリに対してクエリを実行する場合はほぼ必須
- テーブルが変更される時にインデックスも即時更新されるため、基本的には再作成不要
- ただし、大量データをロードするときは、その前にインデックスを一旦削除して、ロード後に再作成した方が効率的

### 統計情報

- 自動バキューム機能が有効になっていてもいつでもすぐに統計情報が更新されるわけではない
- 大量データの更新後は手動でANALYZEしておくのが無難

想定した性能が出ないときは、まずはこういった基本に立ち返ってみると課題解決につながるかもしれません。