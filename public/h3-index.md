---
title: H3インデックスを使った空間分析
tags:
  - H3
  - PostGIS
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
# はじめに

[H3インデックス](https://github.com/zachasme/h3-pg)はUberが乗用車位置の分析などを効率的に実施するために開発したグリッドシステムです（[Uber Blog](https://www.uber.com/gb/en/blog/h3/)）。

セルが六角形（hexagon）の形状をしており、三角形や正方形の場合と異なり、隣接セルの中心点との距離がすべて等距離になります（[H3 Aggregation](https://h3geo.org/docs/highlights/aggregation)）。

そのため、周辺セルを使った空間分析が容易になるという特長があります。

H3では、0-15の解像度（resolution）が定義されており、一つ細かい解像度のセルは、粗い解像度のセルの1/7の面積を持ちます（[H3 Indexing](https://h3geo.org/docs/highlights/indexing)）。

それぞれのセル配置は[H3 Viewer](https://h3.chotard.com/)で確認することができます。

# H3を使ってみる

実際にH3インデックスを使って日本の学校分布を可視化してみたいと思います。

データは[国土数値情報の学校データ](https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-P29-2023.html)を使います。

H3は、各種プログラミング言語及びアプリケーションでAPIが開発・提供されています（[H3 Bindings](https://h3geo.org/docs/community/bindings)）。

私のローカル環境ではPostGISをインストールした際にH3拡張機能もインストールされていたのでそれを使います。

# PostgreSQLで拡張機能を有効化する

作業用のデータベースを作成し、H3関連の拡張機能を有効化します。

```sql
-- データベース作成
CREATE DATABASE h3test;

-- PostGIS拡張機能追加
CREATE EXTENSION postgis;

-- h3-pg拡張機能追加（H3インデックスを求めるのに使う）
CREATE EXTENSION h3;

-- h3_postgis拡張機能追加（セルのPostGISジオメトリを作成するのに使う）
CREATE EXTENSION h3_postgis CASCADE;
```

# 学校データをPostGISに投入

[ogr2ogr](https://gdal.org/en/stable/programs/ogr2ogr.html)を使って、DBにデータをインポートします。

```bash
ogr2ogr \
  -f PostgreSQL \
  PG:"host=localhost port=5432 dbname=h3test user=postgres" \
  P29-23.shp \
  -nln school \
  -lco GEOMETRY_NAME=geom \
  -nlt POINT
```

# H3インデックス列を追加

resolution1-8のH3インデックス列をschoolテーブルに追加します。

H3インデックスは、H3拡張機能の `h3_lat_lng_to_cell` 関数を使って出力します。

ただし、H3インデックスは座標参照系としてWGS84/EPSG:4326を使っている（[H3 Internals Overview](https://h3geo.org/docs/core-library/overview)）ため、`ST_Transform`を使って元データのEPSG:6668からEPSG:4326に座標変換します。

```sql
ALTER TABLE school ADD COLUMN h3_r1 h3index;
ALTER TABLE school ADD COLUMN h3_r2 h3index;
ALTER TABLE school ADD COLUMN h3_r3 h3index;
ALTER TABLE school ADD COLUMN h3_r4 h3index;
ALTER TABLE school ADD COLUMN h3_r5 h3index;
ALTER TABLE school ADD COLUMN h3_r6 h3index;
ALTER TABLE school ADD COLUMN h3_r7 h3index;
ALTER TABLE school ADD COLUMN h3_r8 h3index;

UPDATE school SET h3_r1 = h3_lat_lng_to_cell(ST_Transform(geom, 4326)::point, 1);
UPDATE school SET h3_r2 = h3_lat_lng_to_cell(ST_Transform(geom, 4326)::point, 2);
UPDATE school SET h3_r3 = h3_lat_lng_to_cell(ST_Transform(geom, 4326)::point, 3);
UPDATE school SET h3_r4 = h3_lat_lng_to_cell(ST_Transform(geom, 4326)::point, 4);
UPDATE school SET h3_r5 = h3_lat_lng_to_cell(ST_Transform(geom, 4326)::point, 5);
UPDATE school SET h3_r6 = h3_lat_lng_to_cell(ST_Transform(geom, 4326)::point, 6);
UPDATE school SET h3_r7 = h3_lat_lng_to_cell(ST_Transform(geom, 4326)::point, 7);
UPDATE school SET h3_r8 = h3_lat_lng_to_cell(ST_Transform(geom, 4326)::point, 8);
```

# 集計＋六角形ポリゴン生成

各インデックスごとに学校数を集計した集計テーブルを別途作成します。

その際に当該インデックスの六角形ポリゴンも生成します。

H3インデックスの六角形ポリゴンはh3_postgis拡張機能の `h3_cell_to_boundary_geometry` 関数で作成できます。

```sql
-- 以下のSQLをr2-r8も同様に実行
CREATE TABLE school_h3_r1 AS
SELECT
    h3_r1 as h3,
    1 as resolution,
    count(*) AS school_count,
    count(*) FILTER (WHERE p29_003 = '16001') AS elementary_school_count, -- 小学校
    count(*) FILTER (WHERE p29_003 = '16002') AS junior_high_school_count, -- 中学校
    count(*) FILTER (WHERE p29_003 = '16004') AS high_school_count, -- 高等学校
    count(*) FILTER (WHERE p29_003 = '16005') AS technical_college_count, -- 高等専門学校
    count(*) FILTER (WHERE p29_003 = '16006') AS junior_college_count, -- 短期大学
    count(*) FILTER (WHERE p29_003 = '16007') AS university_count, -- 大学
    count(*) FILTER (WHERE p29_003 = '16011') AS kindergarten_count, -- 高等学校
    h3_cell_to_boundary_geometry(h3_r1) AS geom
FROM school
GROUP BY h3_r1;
```

# GeoJSON出力

`ogr2ogr` を使って上記の集計結果をセルを表す六角形ポリゴンとともにGeoJSONで出力します。

```bash
# r2-r8も同様に実行
ogr2ogr \
  -f GeoJSON \
  school_h3_r1.geojson \
  PG:"host=localhost port=5432 dbname=h3test user=postgres" \
  school_h3_r1
```

# MVT変換

[tippecanoe](https://github.com/felt/tippecanoe)を使ってGeoJSONをMVTに変換します。

ズームレベルに応じてresolutionを使い分けます。

大きいresolutionほど粒度が細かくなるので、大きいズームレベルには大きいresolutionのデータを使います。

今回は、以下の対応で変換してみました。

| z | resolution |
| --: | --: |
| 0-2 | 2 |
| 3-4 | 3 |
| 5-6 | 4 |
| 7-8 | 5 |
| 9-9 | 6 |
| 10-11 | 7 |
| 12-13 | 8 |

```bash
tippecanoe -o r2.mbtiles -l school_h3 -Z0 -z2 -pf -pk -f school_h3_r2.geojson
tippecanoe -o r3.mbtiles -l school_h3 -Z3 -z4 -pf -pk -f school_h3_r3.geojson
tippecanoe -o r4.mbtiles -l school_h3 -Z5 -z6 -pf -pk -f school_h3_r4.geojson
tippecanoe -o r5.mbtiles -l school_h3 -Z7 -z8 -pf -pk -f school_h3_r5.geojson
tippecanoe -o r6.mbtiles -l school_h3 -Z9 -z9 -pf -pk -f school_h3_r6.geojson
tippecanoe -o r7.mbtiles -l school_h3 -Z10 -z11 -pf -pk -f school_h3_r7.geojson
tippecanoe -o r8.mbtiles -l school_h3 -Z12 -z13 -pf -pk -f school_h3_r8.geojson

tile-join -f --no-tile-size-limit -o school_h3.mbtiles r2.mbtiles r3.mbtiles r4.mbtiles r5.mbtiles r6.mbtiles r7.mbtiles r8.mbtiles
```

# MapLibre GL JSで表示

学校ポイントの元データやラベルデータもMVTに追加して、MapLibre GL JSで適当にスタイリングして表示してみたのが以下のサイトです（画像をクリックするとリンクに飛びます）。

<a href="https://hirofumikanda.github.io/school-h3-map/" target="_blank">
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/788a50e3-9599-47dd-9645-fab8550d110d.png" alt="school-h3-map" />
</a>

学校の空間配置を直感的に捉えることができるかと思います。

# まとめ

H3インデックスを使うことで、矩形や三角形のグリッドシステムにはない均質性・等質性を利用した効率的な空間分析ができるようになります。

また、空間分布の可視化でも有用なツールとなります。

他にも多種多様なグリッドシステムがあるため、それぞれのメリデメを考慮して、用途に応じて使い分けられるとよさそうです。