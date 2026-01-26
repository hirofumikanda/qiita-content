---
title: GeoHashを空間インデックスとして使用する
tags:
  - PostgreSQL
  - PostGIS
  - GeoHash
  - 空間データベース
  - 空間インデックス
private: false
updated_at: '2026-01-18T00:00:00+09:00'
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

地理空間データを扱う際、効率的な検索を実現するためには適切なインデックス戦略が不可欠です。PostGISではGiSTインデックスが標準的ですが、**GeoHash**を活用したインデックスも有力な選択肢となります。

GeoHashは、緯度経度を短い文字列にエンコードする手法で、近接する地点が前方一致の類似した文字列になるという特性を持っています。この特性を利用することで、B-treeインデックスを使った空間検索や空間配置に基づく並べ替えが可能になります。

本記事では、実際にPostgreSQL/PostGIS環境でこれらの性能を検証し、GeoHashの有用性と適用場面について考察します。

## GeoHashとは

### 基本原理

GeoHashは、地球を格子状に分割し、各セルに一意の文字列を割り当てる手法です。エンコードは（ざっくりいうと）以下のように行われます。

```
1. 地球全体を緯度経度で32分割する。

2. 各格子にZ字の順番で0-9、b-zの英数字を割り振る。
   ただしアルファベットのうちa,i,l,oは使わない（そのため32文字になる）。

3. 同様に各格子を32分割して次の文字を割り振る。

4. 3.の操作を繰り返して任意の長さの文字列を形成する。
```

百聞は一見に如かずということで、[Geohash Explorer](https://geohash.softeng.co/)をご覧ください。

文字を選択すると、次の格子が表示されます。

例えば、日本の本州の付近は、GeoHashが `xn` になります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/5d6578f1-3627-4191-8e88-93f5adf70d82.png" width="400px">

また、東京駅は、GeoHashが`xn76urx`になります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/5575a455-e68e-4462-b6f3-b909a450a8df.png" width="400px">

このように前方の文字列が一致していれば、矩形範囲の包含関係が成立します。

### GeoHashの特徴

上の例からもわかるとおり、GeoHashには以下のような特徴があります。

1. **階層性**: 文字列が長くなるほど階層的にカバー範囲が狭くなる。前方の文字列が一致していれば包含関係が成立する。
2. **近接性**: 近い地点は似た文字列になる（前方の文字列が一致するため）。ただし、境界に近い地点はその限りではない。

### GeoHashの精度

地球1周が約4万kmであることを考えると、格子の大きさはおおよそ下表のようになります。

| 文字数 | 幅 | 高さ |
|--------|---------|---------|
| 1 | 5000 km | 5000km |
| 2 | 1250 km | 625km |
| 3 | 156.25 km | 156.25 km |
| 4 | 39.06 km | 19.53 km |
| 5 | 4.88 km | 4.88 km |
| 6 | 1.22 km | 0.61 km |
| 7 | 152.59 m | 152.59 m |
| 8 | 38.15 m | 19.07 m |
| 9 | 4.77 m | 4.77 m |
| 10 | 1.19 m | 0.59 m |

これらの値を参考に、求められる精度に応じた文字列長のGeoHashを利用します。

## 性能検証

それでは、GeoHashによる性能を確認してみたいと思います。

### 検証環境

- Windows PC（Windows 11 Pro、Intel N95 1.7GHz、メモリ16GB）
- PostgreSQL 16.11

### PostGIS拡張

まず、PostGIS拡張を入れます。

ジオメトリからGeoHashを生成する関数[ST_GeoHash](https://postgis.net/docs/ja/ST_GeoHash.html)を利用できるようになります。

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
```

### 検証1: 範囲検索

#### テーブル作成

テストテーブルを作成します。

```sql
CREATE TABLE poi_test (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    geom GEOMETRY(Point, 4326),
    geohash VARCHAR(4)  -- 長さ4のGeoHashを使う
);
```

#### テストデータの生成

東京駅周辺（35.681, 139.767）から±0.5度の範囲にランダムに100万件のポイントデータを生成します。

```sql
-- 1000万件のテストデータを挿入
INSERT INTO poi_test (name, geom)
SELECT 
    'POI_' || i,
    ST_SetSRID(
        ST_MakePoint(
            139.767 + random() * 1 - 0.5,
            35.681 + random() * 1 - 0.5
        ), 
        4326
    )
FROM generate_series(1, 10000000) AS i;

-- GeoHashを計算して更新
UPDATE poi_test 
SET geohash = ST_GeoHash(geom, 4); -- 長さ4でGeoHash生成

-- 統計情報を更新
ANALYZE poi_test;
```

#### インデックスの作成

各種インデックスを作成します。

```sql
-- GiSTインデックス（PostGIS標準）
CREATE INDEX idx_poi_gist ON poi_test USING GIST (geom);

-- GeoHash B-treeインデックス
CREATE INDEX idx_poi_geohash ON poi_test (geohash);

-- 統計情報を更新
ANALYZE poi_test;
```

#### 検証内容

それぞれのインデックスを使って、東京駅周辺（35.681, 139.767）から一定の範囲内にあるポイントを検索します。

#### GeoHashインデックスを使用

まず、GeoHash精度4（39.06 km × 19.53 km）を使用して検索を行います。

```sql
-- 対象地点のGeoHash（精度4）を取得
SELECT ST_GeoHash(ST_SetSRID(ST_MakePoint(139.767, 35.681), 4326), 4);
-- 結果: xn76

-- GeoHashで検索
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name, geom
FROM poi_test
WHERE geohash_4 = 'xn76';
"Bitmap Heap Scan on poi_test  (cost=6668.79..308538.22 rows=598756 width=47) (actual time=142.218..3876.279 rows=618304 loops=1)"
"  Recheck Cond: ((geohash)::text = 'xn76'::text)"
"  Rows Removed by Index Recheck: 6006116"
"  Heap Blocks: exact=36883 lossy=65923"
"  Buffers: shared read=103330"
"  ->  Bitmap Index Scan on idx_poi_geohash  (cost=0.00..6519.11 rows=598756 width=0) (actual time=93.704..93.704 rows=618304 loops=1)"
"        Index Cond: ((geohash)::text = 'xn76'::text)"
"        Buffers: shared read=524"
"Planning Time: 0.207 ms"
"Execution Time: 3941.711 ms"

-- インデックスサイズ
SELECT pg_size_pretty(pg_relation_size(indexname::regclass)) as index_size
FROM pg_indexes
WHERE tablename = 'poi_test'
AND indexname = 'idx_poi_geohash'
-- 結果: 66 MB
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/21ad38ac-f0bf-4443-97b3-6107637a4b8f.png" width="400px">

#### GiSTインデックスを使用

次に、ジオメトリのGISTインデックスを使って同様の空間検索を行います。

性能を比較したいので、同じくらいの点数が抽出されるようにします。

```
r = ((39.06 km × 19.53 km / pi) ** 1/2) * 360 度 / 40000 km
  = 0.14 度
```

と計算されるため、半径0.14度で検索します。

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name, geom
FROM poi_test
WHERE ST_DWithin(
    geom,
    ST_SetSRID(ST_MakePoint(139.767, 35.681), 4326),
    0.14
);
"Index Scan using idx_poi_gist on poi_test  (cost=0.54..10453096.85 rows=1000 width=47) (actual time=40.593..19400.499 rows=615425 loops=1)"
"  Index Cond: (geom && st_expand('0101000020E6100000068195438B786140BA490C022BD74140'::geometry, '0.14'::double precision))"
"  Filter: st_dwithin(geom, '0101000020E6100000068195438B786140BA490C022BD74140'::geometry, '0.14'::double precision)"
"  Rows Removed by Filter: 169088"
"  Buffers: shared hit=122538 read=666137"
"Planning Time: 0.450 ms"
"Execution Time: 19496.312 ms"

-- インデックスサイズ
SELECT pg_size_pretty(pg_relation_size(indexname::regclass)) as index_size
FROM pg_indexes
WHERE tablename = 'poi_test'
AND indexname = 'idx_poi_gist'
-- 結果: 398 MB
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/65608354-f9ac-4d59-81d3-ed326610f76d.png" width="400px">

#### 検証結果

どちらも60万点ほどのポイントが抽出されていますが、以下の違いがあります。

- GeoHashインデックスのみで検索すると、GiSTインデックスを使った厳密な空間検索に比べて5倍速く結果が出ており**計算コストが低い**。
- インデックスサイズはGeoHashインデックスの方が**1/5ほど小さい**。
- しかし、GeoHashインデックスは既定の矩形範囲でしか検索できないため、**精度は高くない**。特に、東京駅のような境界に近いポイントはそれが顕著に表れる（上図でも抽出されたポイントの範囲は大きく異なる）。

### 検証2: ソート

次のGeoHashを使ったソート性能について検証します。

#### テーブル作成

東京駅周辺（35.681, 139.767）から±0.5度の範囲にランダムに半径0.005度のバッファー円を10万件生成します。

```sql
-- テスト用データ作成
CREATE TABLE t_union_test AS
SELECT
  i AS id,
  ST_Buffer(
    ST_SetSRID(
        ST_MakePoint(
            139.767 + random() * 1 - 0.5,
            35.681 + random() * 1 - 0.5
        ), 
        4326
    ),
    0.005
  ) AS geom
FROM generate_series(1, 100000) AS i;

-- 統計情報を更新
ANALYZE t_union_test;
```

#### 検証内容

一部重複する部分をもつ10万個のポリゴンを合成します。
`Unary Union`関数を使って100件ずつ段階的に結合することとし、その際、GeoHashで事前にソートした場合とそうでない場合でどの程度の差異が出るか確認します。

#### GeoHashで事前にソートする場合

ジオメトリをGeoHashの値でソートした上で、100件ずつのチャンクに分けて結合し、最後にそれらをまとめて統合します。

```sql
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
"Aggregate  (cost=159518.45..159530.96 rows=1 width=32) (actual time=11083.151..11083.243 rows=1 loops=1)"
"  Buffers: shared hit=2258 read=5600, temp read=19391 written=24654"
"  ->  HashAggregate  (cost=156863.83..159490.83 rows=200 width=40) (actual time=1340.785..10148.580 rows=1001 loops=1)"
"        Group Key: (row_number() OVER (?) / 100)"
"        Batches: 37  Memory Usage: 8296kB  Disk Usage: 57704kB"
"        Buffers: shared hit=2258 read=5600, temp read=19391 written=24654"
"        ->  WindowAgg  (cost=54717.18..143113.83 rows=100000 width=608) (actual time=743.483..1042.494 rows=100000 loops=1)"
"              Buffers: shared hit=2258 read=5600, temp read=7168 written=7183"
"              ->  Gather Merge  (cost=54717.18..66363.83 rows=100000 width=600) (actual time=743.462..990.465 rows=100000 loops=1)"
"                    Workers Planned: 2"
"                    Workers Launched: 2"
"                    Buffers: shared hit=2258 read=5600, temp read=7168 written=7183"
"                    ->  Sort  (cost=53717.15..53821.32 rows=41667 width=600) (actual time=573.960..641.998 rows=33333 loops=3)"
"                          Sort Key: (st_geohash(st_centroid(t_union_test.geom), 6))"
"                          Sort Method: external merge  Disk: 19712kB"
"                          Buffers: shared hit=2258 read=5600, temp read=7168 written=7183"
"                          Worker 0:  Sort Method: external merge  Disk: 18816kB"
"                          Worker 1:  Sort Method: external merge  Disk: 18816kB"
"                          ->  Parallel Seq Scan on t_union_test  (cost=0.00..39410.92 rows=41667 width=600) (actual time=0.715..242.999 rows=33333 loops=3)"
"                                Buffers: shared hit=2144 read=5600"
"Planning:"
"  Buffers: shared hit=2"
"Planning Time: 1.825 ms"
"Execution Time: 11105.754 ms"
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/5693b0d2-f662-40f1-a2ad-2cc8165a225e.png" height="400px">

およそ11sかかりました。

#### GeoHashで事前にソートしない場合

次は、ソートしないで愚直にジオメトリを順番に結合します。

```sql
EXPLAIN (ANALYZE, BUFFERS)
WITH numbered AS (
  SELECT geom,
         (row_number() OVER ()) / 100 AS grp
  FROM t_union_test
),
chunked AS (
  SELECT ST_UnaryUnion(ST_Collect(geom)) AS g
  FROM numbered
  GROUP BY grp
)
SELECT ST_UnaryUnion(ST_Collect(g)) FROM chunked;
"Aggregate  (cost=25648.62..25661.13 rows=1 width=32) (actual time=24802.314..24802.318 rows=1 loops=1)"
"  Buffers: shared hit=2240 read=5504, temp read=12223 written=17471"
"  ->  HashAggregate  (cost=22994.00..25621.00 rows=200 width=40) (actual time=522.238..14720.162 rows=1001 loops=1)"
"        Group Key: (row_number() OVER (?) / 100)"
"        Batches: 37  Memory Usage: 8296kB  Disk Usage: 57704kB"
"        Buffers: shared hit=2240 read=5504, temp read=12223 written=17471"
"        ->  WindowAgg  (cost=0.00..10244.00 rows=100000 width=576) (actual time=0.099..196.882 rows=100000 loops=1)"
"              Buffers: shared hit=2240 read=5504"
"              ->  Seq Scan on t_union_test  (cost=0.00..8744.00 rows=100000 width=568) (actual time=0.084..127.730 rows=100000 loops=1)"
"                    Buffers: shared hit=2240 read=5504"
"Planning Time: 0.156 ms"
"Execution Time: 24858.397 ms"
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/5693b0d2-f662-40f1-a2ad-2cc8165a225e.png" height="400px">

25sほどかかりました。

#### 検証結果

- GeoHashで事前にソートした方がソートしない場合より**2倍処理が早い**。
- これはソートした方が、**近接するポリゴン同士が近い位置に並ぶ**ため、より効率的に結合できたことによる。

## まとめ

以上の検証から、GeoHashの空間インデックスとしての有用性は以下が考えられます。

1. **範囲検索**
    - GeoHashのB-Treeインデックスは**サイズが小さく検索処理も早いため、大規模データで粗く絞り込みたいときに有用**である
    - しかし、GiSTインデックスはジオメトリに合わせてバウンディングボックスが動的に作成されるのに対して、**GeoHashは既定の矩形範囲が使われるため、検索精度は劣後**する。特に境界に近い地点では注意が必要
2. **ソート**
    - GeoHashでソートすると、**空間的に近接するフィーチャを近くに並べることができる**
    - ただし、境界付近で大きくずれる可能性があるため、**厳密な近接性が求められる場面には向かない**

GeoHashは万能ではありませんが、適切な場面で使用すれば強力なツールになります。ユースケースで適合しそうな場面があればGiSTと組み合わせて利用を検討してみるとよいかもしれません。
