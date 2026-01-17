---
title: 'PostGISでジオメトリを結合したいときに結局どの関数を使えばよいのか'
tags:
  - PostgreSQL
  - PostGIS
  - GIS
  - 空間データベース
  - SQL
private: false
updated_at: '2026-01-10T12:00:00+09:00'
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

PostGISでは、複数のジオメトリを1つにまとめる操作を行う関数がいくつか用意されていますが、関数名からだけでは機能の違いが判然としない部分もあるため、結局どれを使うのが最適なのかわからないケースもあるかと思います。

本記事では、PostGISにおけるジオメトリ統合の主要な関数について、ポリゴンとラインそれぞれの場合に分けて検証し、用途に応じた最適な関数の選定基準を提示できればと思います。

## テストデータの準備

検証は以下の環境で実施します。

- Windows PC（Windows 11 Pro、Intel N95 1.7GHz、メモリ16GB）
- PostgreSQL 16.11

まず、テストデータを用意します。

```sql
-- テスト用テーブルの作成
CREATE TABLE test_polygons (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50),
    geom GEOMETRY(Polygon, 4326)
);

CREATE TABLE test_lines (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50),
    geom GEOMETRY(LineString, 4326)
);

-- ポリゴンデータの挿入（隣接・重複する3つの四角形）
INSERT INTO test_polygons (name, geom) VALUES
('Polygon A', ST_GeomFromText('POLYGON((0 0, 2 0, 2 2, 0 2, 0 0))', 4326)),
('Polygon B', ST_GeomFromText('POLYGON((2 0, 4 0, 4 2, 2 2, 2 0))', 4326)),
('Polygon C', ST_GeomFromText('POLYGON((1 1, 3 1, 3 3, 1 3, 1 1))', 4326));

-- ラインデータの挿入（接続可能な3つのライン）
INSERT INTO test_lines (name, geom) VALUES
('Line A', ST_GeomFromText('LINESTRING(0 0, 2 2)', 4326)),
('Line B', ST_GeomFromText('LINESTRING(2 2, 3 1)', 4326)),
('Line C', ST_GeomFromText('LINESTRING(0 2, 2 0)', 4326));
```

ジオメトリを表示すると以下のとおりです。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/df218a0f-8efb-46bc-b237-9dfdce64b7a5.png" width="400px">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/24faab14-75b5-4209-a905-39011f36352a.png" width="400px">

## ポリゴンの結合

まず、ポリゴンの結合について取り上げます。

最初は、最もシンプルなST_Collectを紹介します。

### ST_Collect - 集約してMultiPolygonを作成

[ST_Collect](https://postgis.net/docs/ja/ST_Collect.html)は、複数のジオメトリをそのままグループ化して、マルチジオメトリにする関数です。
ポリゴンの場合はマルチポリゴンになります。
集約関数のため、**複数データ行を入力とすることができます。**
単にひとつにまとめるだけで、**オーバーラップを併合して境界を溶かしたりはしません。**

```sql
SELECT ST_AsText(ST_Collect(geom)) 
FROM test_polygons;
```

**結果:**
```
MULTIPOLYGON(((0 0,2 0,2 2,0 2,0 0)),((2 0,4 0,4 2,2 2,2 0)),((1 1,3 1,3 3,1 3,1 1)))
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/bc9b9b6e-2cf9-4bef-b6e2-818a4f11e719.png" width="400px">

なお、[ST_Dump](https://postgis.net/docs/ja/ST_Dump.html)は`ST_Collect`の逆の操作になります。

```sql
SELECT ST_AsText((ST_Dump(ST_Collect(geom))).geom) 
FROM test_polygons;
```

**結果:**
```
POLYGON((0 0,2 0,2 2,0 2,0 0))
POLYGON((2 0,4 0,4 2,2 2,2 0))
POLYGON((1 1,3 1,3 3,1 3,1 1))
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/eede505a-9c35-43c4-83bd-4b194e7df19c.png" width="400px">

マルチポリゴンにしたいだけであれば、`ST_Collect`でよいのですが、オーバーラップを併合して境界を溶かしたい場合は、ST_Union系の関数を使います。

### ST_MemUnion - 境界を溶かして結合（逐次統合）

ST_Union系の関数のうち、内部のアルゴリズム的に最もシンプルな関数が、[ST_MemUnion](https://postgis.net/docs/ja/ST_MemUnion.html)になります。
`ST_MemUnion`を実行すると、オーバーラップを併合して境界を溶かしたジオメトリを生成します。
`ST_Collect`と同様、**複数データ行を入力とすることができる集約関数です。**

```sql
SELECT ST_AsText(ST_MemUnion(geom)) 
FROM test_polygons;
```

**結果:**
```
POLYGON((0 2,1 2,1 3,3 3,3 2,4 2,4 0,2 0,0 0,0 2))
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/faeae1b8-7ebb-46e4-9478-56529d35f46c.png" width="400px">

オーバーラップ部分の境界がなくなり、一つのポリゴンに統合されていることがわかるかと思います。

この統合操作を実行するためのアルゴリズムはいくつかありますが、`ST_MemUnion`では、**Iterated Union（逐次加算的結合）** という方法が使われています。

これは、1つ1つ対象のジオメトリを読み込んで、逐次結合していく方法です。

メモリ使用量を小さく抑えられますが、処理時間は長くなります。

そこで、通常は、より高効率なアルゴリズムで処理する`ST_Union`を使用します。

### ST_Union - 境界を溶かして結合（カスケード統合）

[ST_Union](https://postgis.net/docs/ja/ST_Union.html)も、`ST_MemUnion`と同様に**オーバーラップするジオメトリの境界を溶かして統合したジオメトリを作成する集約関数**です。

```sql
SELECT ST_AsText(ST_Union(geom)) 
FROM test_polygons;
```

**結果:**
```
POLYGON((0 0,0 2,1 2,1 3,3 3,3 2,4 2,4 0,2 0,0 0))
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/faeae1b8-7ebb-46e4-9478-56529d35f46c.png" width="400px">

`ST_MemUnion`の結果と始点が若干異なるものの、形状は同じです。

しかし、結合のアルゴリズムが異なります。
`ST_Union`では、[Cascaded Union](https://lin-ear-th-inking.blogspot.com/2007/11/fast-polygon-merging-in-jts-using.html)と呼ばれるアルゴリズムが使われています。

これは、あらかじめすべての対象ジオメトリをメモリに読み込んで、**STRツリー**と呼ばれる静的なRツリーインデックス（**Sort-Tile-Recursive packed R-Tree index**）を作成し、このインデックスにしたがって、下位ノードから段階的に統合処理をすることで、近接するポリゴン同士から効率的に結合するというアプローチをとります。

これにより、高速な統合処理を実現しますが、事前にすべてのジオメトリをメモリ上に展開し**STRツリー**を作成するため、メモリ使用量が大きくなる傾向があります。

そのため、使用できるメモリが限定的な環境では、`ST_Union`ではメモリ不足でエラーになることがあります。

その場合、`ST_UnaryUnion`と`ST_Collect`を組み合わせてチャンク（一定のまとまり）ごとに結合処理をする方法が考えられます。

### ST_UnaryUnion - 境界を溶かして結合（単一入力のカスケード統合）

[ST_UnaryUnion](https://postgis.net/docs/ja/ST_UnaryUnion.html)は、単一入力のジオメトリ（マルチポリゴンやジオメトリコレクション）内のオーバーラップを併合する関数です。

`ST_Union`や`ST_MemUnion`と異なり、**集約関数ではないため、複数のデータ行を入力にとることはできません**。

したがって、複数データ行のジオメトリを統合する場合は、以下のとおり、`ST_Collect`を合わせて用いることになります。

```sql
SELECT ST_AsText(ST_UnaryUnion(ST_Collect(geom))) 
FROM test_polygons;
```

**結果:**
```
POLYGON((0 0,0 2,1 2,1 3,3 3,3 2,4 2,4 0,2 0,0 0))
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/faeae1b8-7ebb-46e4-9478-56529d35f46c.png" width="400px">

結合のアルゴリズムはST_Unionと同様、[Cascaded Union](https://lin-ear-th-inking.blogspot.com/2007/11/fast-polygon-merging-in-jts-using.html)が使われるため、`ST_MemUnion`より高速に処理されます。

また、一定数量ずつ小分けにして`ST_UnaryUnion`と`ST_Collect`を使って段階的にジオメトリを結合することで、メモリ使用量を抑えつつ、効率的に結合処理を実行することができます。

例えば、以下のようなSQLを実行します。

```sql
WITH numbered AS (
  SELECT geom,
         (row_number() OVER (ORDER BY ST_GeoHash(ST_Centroid(geom), 6))) / 100 AS grp
  FROM t_union_test
),
chunked AS (
  SELECT ST_UnaryUnion(ST_Collect(geom)) AS g
  FROM numbered
  GROUP BY grp
)
SELECT ST_UnaryUnion(ST_Collect(g)) FROM chunked;
```

ここでは、重心座標ベースで[ST_GeoHash](https://postgis.net/docs/ja/ST_GeoHash.html)順にソートした上で、100個ずつのチャンクを作って、チャンクごとに結合し、最後に、チャンクごとに結合したジオメトリを一つに統合しています。

### ST_Union系関数の性能比較

結合処理の効率性を検証するため、ST_Union、ST_MemUnion、ST_UnaryUnionのそれぞれを使った場合の処理時間を比較してみます。

```sql
-- テスト用データ作成
CREATE TABLE t_union_test AS
SELECT
  i AS id,
  -- 15m間隔に隣同士が重なり合う半径10mのバッファ
  ST_Transform(ST_Buffer(
    ST_SetSRID(ST_MakePoint((i % 80) * 15, (i / 80) * 15), 3857),
    10
  ), 4326) AS geom
FROM generate_series(1, 2000) AS i;

ANALYZE t_union_test;

-- (A) ST_Union 統合
EXPLAIN (ANALYZE, BUFFERS)
SELECT ST_Union(geom) FROM t_union_test;
"Aggregate  (cost=229.50..229.51 rows=1 width=32) (actual time=566.683..566.686 rows=1 loops=1)"
"  Buffers: shared hit=192"
"  ->  Seq Scan on t_union_test  (cost=0.00..212.00 rows=2000 width=568) (actual time=0.046..0.702 rows=2000 loops=1)"
"        Buffers: shared hit=192"
"Planning:"
"  Buffers: shared hit=20"
"Planning Time: 2.408 ms"
"Execution Time: 567.317 ms"

-- (B) ST_MemUnion 統合
EXPLAIN (ANALYZE, BUFFERS)
SELECT ST_MemUnion(geom) FROM t_union_test;
"Aggregate  (cost=25212.00..25212.01 rows=1 width=32) (actual time=38329.418..38329.419 rows=1 loops=1)"
"  Buffers: shared hit=192"
"  ->  Seq Scan on t_union_test  (cost=0.00..212.00 rows=2000 width=568) (actual time=0.035..8.795 rows=2000 loops=1)"
"        Buffers: shared hit=192"
"Planning Time: 0.115 ms"
"Execution Time: 38329.766 ms"

-- (C) ST_UnaryUnion + ST_Collect チャンクごと統合
EXPLAIN (ANALYZE, BUFFERS)
-- 例：GeoHash順にソートした上で100件ずつ unary union
WITH numbered AS (
  SELECT geom,
         (row_number() OVER (ORDER BY ST_GeoHash(ST_Centroid(geom), 6))) / 100 AS grp
  FROM t_union_test
),
chunked AS (
  SELECT ST_UnaryUnion(ST_Collect(geom)) AS g
  FROM numbered
  GROUP BY grp
)
SELECT ST_UnaryUnion(ST_Collect(g)) FROM chunked;
"Aggregate  (cost=6291.28..6303.79 rows=1 width=32) (actual time=722.121..722.125 rows=1 loops=1)"
"  Buffers: shared hit=192"
"  ->  HashAggregate  (cost=3636.66..6263.66 rows=200 width=40) (actual time=36.299..351.440 rows=21 loops=1)"
"        Group Key: (row_number() OVER (?) / 100)"
"        Batches: 1  Memory Usage: 3104kB"
"        Buffers: shared hit=192"
"        ->  WindowAgg  (cost=1821.66..3361.66 rows=2000 width=608) (actual time=14.338..15.571 rows=2000 loops=1)"
"              Buffers: shared hit=192"
"              ->  Sort  (cost=1821.66..1826.66 rows=2000 width=600) (actual time=14.310..14.488 rows=2000 loops=1)"
"                    Sort Key: (st_geohash(st_centroid(t_union_test.geom), 6))"
"                    Sort Method: quicksort  Memory: 1220kB"
"                    Buffers: shared hit=192"
"                    ->  Seq Scan on t_union_test  (cost=0.00..1712.00 rows=2000 width=600) (actual time=0.086..12.351 rows=2000 loops=1)"
"                          Buffers: shared hit=192"
"Planning:"
"  Buffers: shared hit=2"
"Planning Time: 1.598 ms"
"Execution Time: 723.638 ms"
```

処理時間は、

(A) ST_Union 統合 < (C) ST_UnaryUnion + ST_Collect チャンクごと統合 <<< (B) ST_MemUnion 統合

となります。

メモリ使用量は逆順になるため、実務では、(A) ST_Union 統合→(C) ST_UnaryUnion + ST_Collect チャンクごと統合→(B) ST_MemUnion 統合の順番に試行して最初にメモリ不足にならなかった方法を選定すればよいと考えられます。

## ラインの結合

次にラインの結合について検討します。

### ST_Collect - 集約してMultiLineStringを作成

ポリゴンと同様に、[ST_Collect](https://postgis.net/docs/ja/ST_Collect.html)は複数のラインをそのままグループ化します。

```sql
SELECT ST_AsText(ST_Collect(geom)) 
FROM test_lines;
```

**結果:**
```
MULTILINESTRING((0 0,2 2),(2 2,3 1),(0 2,2 0))
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/24faab14-75b5-4209-a905-39011f36352a.png" width="400px">

`ST_Dump`が`ST_Collect`と逆の挙動になるのも同様です。

```sql
SELECT ST_AsText((ST_Dump(ST_Collect(geom))).geom) 
FROM test_lines;
```

**結果:**
```
LINESTRING(0 0,2 2)
LINESTRING(2 2,3 1)
LINESTRING(0 2,2 0)
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/24faab14-75b5-4209-a905-39011f36352a.png" width="400px">

端点が一致するライン同士を結合して、一つのラインにしたい場合は別の関数を使う必要があります。

まずは、ポリゴンの場合と同様に`ST_Union`を試してみます。

### ST_Union - 集約してMultiLineStringを作成（交差する地点で分割）

実は、[ST_Union](https://postgis.net/docs/ja/ST_Union.html)をラインに対して実施すると、接続するライン同士を結合したりはせずに、`ST_Collect`と同様に複数ラインをグループ（マルチ）化します。

ただし、`ST_Collect`と異なり、グループ化する際に、互いに交差するラインについては交差するところでラインを分割します。

```sql
SELECT ST_AsText(ST_Union(geom)) 
FROM test_lines;
```

**結果:**
```
MULTILINESTRING((0 0,1 1),(1 1,2 2),(2 2,3 1),(0 2,1 1),(1 1,2 0))
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/24faab14-75b5-4209-a905-39011f36352a.png" width="400px">

`ST_Collect`の結果と比較すると、`ST_Union`では(1,1)の交点でラインが分割されています。

そのため、ラインを結合するという目的で`ST_Union`を使用すると、想定していない処理結果になる可能性があり注意が必要です。

接続されたラインを結合したい場合は、`ST_LineMerge`を使う必要があります。

### ST_LineMerge - 接続されたラインを1本に結合

[ST_LineMerge](https://postgis.net/docs/ja/ST_LineMerge.html)は、**端点が接続されているライン**を1本の連続したラインに統合します。

ただし、**集約関数ではない**ため、複数データ行のラインを結合したい場合は、`ST_Collect`と合わせて使います。

```sql
SELECT ST_AsText(
    ST_LineMerge(
        ST_Collect(geom)
    )
) 
FROM test_lines;
```

**結果:**
```
MULTILINESTRING((0 0,2 2,3 1),(0 2,2 0))
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/24faab14-75b5-4209-a905-39011f36352a.png" width="400px">

ライン(0 0,2 2)とライン(2 2,3 1)が一つのライン(0 0,2 2,3 1)に統合されています。

## まとめ

以上をまとめると次のようになります。

**ポリゴンの場合**
- 単純にグループ化したい場合は`ST_Collect`を使う
- 境界を溶かしてオーバーラップを併合したい場合は`ST_Union`を使う
- ただし、メモリ不足で落ちる場合は`ST_UnaryUnion`+`ST_Collect`または`ST_MemUnion`を使う

**ラインの場合**
- 単純にグループ化したい場合は`ST_Collect`を使う
- 接続されたラインを結合したい場合は`ST_LineMerge`+`ST_Collect`を使う

データの性質と処理の目的に応じて関数を使い分けることで、効率的な地理空間データ処理が可能になるかと思いますので、その際の参考になれば幸いです。