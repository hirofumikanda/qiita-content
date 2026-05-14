---
title: MapLibre Tile のパフォーマンスについて
tags:
  - GIS
  - MLT
  - vectortile
  - MVT
private: false
updated_at: '2026-05-15T00:08:21+09:00'
id: 0bb95c3b8acd94483507
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

ベクトルタイル形式として長年使われてきた **Mapbox Vector Tile (MVT)** に対し、MapLibreコミュニティでは次世代形式の **MapLibre Tile (MLT)** が議論・検討されています。

MapLibreのサイトでは[MapLibre Tile Specification](https://maplibre.org/maplibre-tile-spec/specification/)が公開されており、仕様の全体像を確認することができます。

[GitHub](https://github.com/maplibre/maplibre-tile-spec)ではエンコーダー及びデコーダーが既に提供されており、実際にMLTを作成してみることも可能です。[Imprementation Status](https://maplibre.org/maplibre-tile-spec/implementation-status/)を確認すると、[MapLibre GL JS](https://maplibre.org/maplibre-gl-js/docs/)などに実装されているため、地図描画してみることもできます。

[Introduction](https://maplibre.org/maplibre-tile-spec/)には、MVTをベースとしつつも、圧縮効率及びデコード性能の向上、3D対応や複雑なデータ型のサポートなど、多くの改善点が挙げられており、NEXT MVTとしてかなり期待の持てる形式になっているのではないかと思います。

私自身、仕様を余り読み込めている（理解できている）わけではないのですが、百聞は一見に如かずということで、試しに、エンコーダーでMLTを作成して性能を簡単に調べてみましたので、その結果を共有できればと思います。

## エンコードツールのインストール

GitHubリポジトリの[README](https://github.com/maplibre/maplibre-tile-spec#directory-structure)を確認すると、javaでエンコーダーが提供されているようですので、こちらをビルドして使いたいと思います。

まず、JDK21をインストールします。なお、PC環境は以下のとおりです。

- Windows PC（Windows 11 Pro、Intel N95 1.7GHz、メモリ16GB）
- WSL(Ubuntu)

```bash
sudo apt update
sudo apt install openjdk-21-jdk
```

GitHubをクローンしてビルドします。

```bash
git clone https://github.com/maplibre/maplibre-tile-spec.git
cd maplibre-tile-spec/java

./gradlew :mlt-cli:cli

ls -l mlt-cli/build/libs/
# mlt-cli/build/libs/encode.jarがあればOK
```

## OvertureMapsからサンプルデータ取得

サンプルデータとして、[OvertureMaps](https://overturemaps.org/)を使いたいと思います。

今回は、皇居周辺のデータでテストします。

様々な種別のデータで比較するため、ポリゴン（building、landuse）、ライン（road）、ポイント（place）のデータを取得します。

データ取得には[duckdb](https://duckdb.org/)を使います。

```bash
duckdb

D load spatial;
D COPY(
  SELECT
    id,
    subtype,
    class,
    names.primary AS name,
    num_floors,
    num_floors_underground,
    geometry
  FROM read_parquet('s3://overturemaps-us-west-2/release/2026-04-15.0/theme=buildings/type=building/*', filename=true, hive_partitioning=1)
  WHERE names.primary IS NOT NULL
  AND bbox.xmin BETWEEN 139.69381 AND 139.81715
  AND bbox.ymin BETWEEN 35.66338 AND 35.70599
) TO 'building.geojson' WITH (FORMAT GDAL, DRIVER 'GeoJSON');

D COPY(
    SELECT
       id,
       subtype,
       class,
       surface,
       geometry
    FROM read_parquet('s3://overturemaps-us-west-2/release/2026-04-15.0/theme=base/type=land_use/*', filename=true, hive_partitioning=1)
    WHERE bbox.xmin BETWEEN 139.69381 AND 139.81715
	  AND bbox.ymin BETWEEN 35.66338 AND 35.70599
) TO 'landuse.geojson' WITH (FORMAT GDAL, DRIVER 'GeoJSON');

D COPY(
    SELECT
       id,
       names.primary as name,
       subtype,
       class,
       geometry
    FROM read_parquet('s3://overturemaps-us-west-2/release/2026-04-15.0/theme=transportation/type=segment/*', filename=true, hive_partitioning=1)
    WHERE subtype = 'road'
    AND bbox.xmin BETWEEN 139.69381 AND 139.81715
	  AND bbox.ymin BETWEEN 35.66338 AND 35.70599
) TO 'road.geojson' WITH (FORMAT GDAL, DRIVER 'GeoJSON');

D COPY(
    SELECT
       id,
       categories.primary as category,
       CAST(websites AS JSON) as websites,
       brand.names.primary as brand,
       theme,
       type,
       names.primary as name,
       geometry
    FROM read_parquet('s3://overturemaps-us-west-2/release/2026-04-15.0/theme=places/type=place/*', filename=true, hive_partitioning=1)
    WHERE bbox.xmin BETWEEN 139.69381 AND 139.81715
	  AND bbox.ymin BETWEEN 35.66338 AND 35.70599
	  AND confidence >= 0.99
) TO 'place.geojson' WITH (FORMAT GDAL, DRIVER 'GeoJSON');
```

## MVT作成

エンコーダーの[CLI Usage](https://github.com/maplibre/maplibre-tile-spec/tree/main/java#cli-usage)を見ると、MVTからMLTに変換する仕様となっているため、まず、MVTを作成します。

feltの[tippecanoe](https://github.com/felt/tippecanoe)を使ってGeoJSONからMVTに変換します。

```bash
# building
tippecanoe -z16 -Z9 -f -l building -pf -pk --simplification=5 --simplify-only-low-zooms -o building.mbtiles building.geojson

# landuse
tippecanoe -z16 -Z9 -f -l landuse -pf -pk --simplification=5 --simplify-only-low-zooms -o landuse.mbtiles landuse.geojson

# road
tippecanoe -z16 -Z9 -f -l road -pf -pk  --simplification=5 --simplify-only-low-zooms -o road.mbtiles road.geojson

# place
tippecanoe -z16 -Z9 -f -l place -pf -pk --drop-rate=0 -o place.mbtiles place.geojson
```

## MVT→MLT変換

冒頭ビルドしたエンコーダーを使ってMVTからMLTに変換します。

```bash
# building
java -jar mlt-cli/build/libs/encode.jar \
  --mbtiles building.mbtiles \
  --dir .
# building.mlt.mbtilesが生成される

# landuse
java -jar mlt-cli/build/libs/encode.jar \
  --mbtiles landuse.mbtiles \
  --dir .
# landuse.mlt.mbtilesが生成される

# road
java -jar mlt-cli/build/libs/encode.jar \
  --mbtiles road.mbtiles \
  --dir .
# road.mlt.mbtilesが生成される

# place
java -jar mlt-cli/build/libs/encode.jar \
  --mbtiles place.mbtiles \
  --dir .
# place.mlt.mbtilesが生成される
```

## MapLibre GL JSで可視化

MapLibre GL JSの最新バージョンでは、MLTをデコードして表示することが可能です。

実際に表示してみたのが以下のサイトになります。

<a href="https://hirofumikanda.github.io/mlt-demo/#15/35.68468/139.75548" target="_blank">
<img alt="mlt-dem" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/4a79cb01-6bca-4e3f-a611-4e47d2d59a9f.png" width="400px" />
</a>

`style.json`を以下のように定義することでMLTでデコードしてくれます。

```json
"building": {
  "type": "vector",
  "encoding": "mlt",
  "url": "pmtiles://pmtiles/building.mlt.pmtiles",
  "attribution": "&copy; <a href='https://www.openstreetmap.org/copyright'>OpenStreetMap</a> contributors, <a href='https://docs.overturemaps.org/attribution/'>Overture Maps Foundation</a>",
  "minzoom": 9,
  "maxzoom": 16
}
```

なお、タイルサーバを立てなくてよいように、[protomapsのツール](https://github.com/protomaps/pmtiles#creating-pmtiles)で事前に `mbtiles` を `pmtiles` に変換しています。

```bash
pmtiles convert building.mbtiles building.pmtiles
```

## データサイズ比較

MVTとMLTの、データサイズを比較してみます。

### MLTをgzip圧縮
MVTはgzip圧縮されているのですが、MLTの方は未圧縮で生成されているので、土俵をそろえるため、MLTの方もgzip圧縮しておきます。

`mbtiles` は中身はsqlite3であるため、以下のようなpythonスクリプトでgzip圧縮します。

```python
import sqlite3, gzip, shutil

src = "building.mlt.mbtiles"
dst = "building.mlt.gzip.mbtiles"

shutil.copyfile(src, dst)

con = sqlite3.connect(dst)
cur = con.cursor()

rows = cur.execute("""
SELECT zoom_level, tile_column, tile_row, tile_data
FROM tiles
""").fetchall()

for z, x, y, data in rows:
    if data[:2] == b"\x1f\x8b":
        continue

    gz = gzip.compress(data, compresslevel=9)

    cur.execute("""
    UPDATE tiles
    SET tile_data = ?
    WHERE zoom_level = ? AND tile_column = ? AND tile_row = ?
    """, (gz, z, x, y))

con.commit()
con.close()

print(f"wrote {dst}")
```

### mbtilesのサマリー出力

mbtiles内のタイルサイズを調べるため、Martinの[mbtilesツール](https://maplibre.org/martin/mbtiles/)のsummaryを使います。

インストールが手間なので、Dockerコンテナを使って実施します。

```bash
# building
docker run --rm -v $(pwd):/work -w /work \
	--entrypoint "" \
	ghcr.io/maplibre/martin:latest \
	mbtiles summary building.mbtiles

MBTiles file summary for building.mbtiles
Schema: normalized
File size: 4.22MiB
Page size: 4.00KiB
Page count: 1081

 Zoom |   Count   | Smallest  |  Largest  |  Average  | Bounding Box
    9 |         1 |  135.7KiB |  135.7KiB |  135.7KiB | 139,35,140,36
   10 |         1 |  282.5KiB |  282.5KiB |  282.5KiB | 140,35,140,36
   11 |         2 |  141.0KiB |  299.2KiB |  220.1KiB | 139.6,35.6,139.9,35.7
   12 |         4 |   53.9KiB |  275.2KiB |  123.9KiB | 139.7,35.6,139.8,35.7
   13 |         8 |    4.1KiB |  208.8KiB |   66.7KiB | 139.7,35.6,139.8,35.7
   14 |        21 |    3.8KiB |  104.9KiB |   28.9KiB | 139.68,35.66,139.83,35.71
   15 |        72 |      499B |   42.7KiB |    9.6KiB | 139.69,35.66,139.82,35.71
   16 |       235 |      145B |   28.4KiB |    3.5KiB | 139.69,35.66,139.82,35.71
  all |       344 |      145B |  299.2KiB |   11.6KiB | 139.22,35.46,139.92,36.03
```

これを全ての`MVT`のmbtilesと`MLT`のmbtilesについて実行します。

その結果、すべてのタイルの平均サイズは以下のとおりになりました。

| type | MVT average size | MLT average size | MLT/MVT ratio(%) |
| :--- | :--- | :--- | :--- |
| building | 11.6KiB | 9.4KiB | 81% |
| landuse | 3.6KiB | 3.0KiB | 83% |
| road | 25.4KiB | 20.3KiB | 80% |
| place | 16.0KiB | 12.9KiB | 81% |

## デコード速度比較

次にMVTとMLTのデコード速度を比較してみます。

簡易的に比較するため、以下のJavaScript関数の実行所要時間で比較したいと思います。

```javascript
import Pbf from "https://esm.sh/pbf";
import { VectorTile } from "https://esm.sh/@mapbox/vector-tile";

// MVTデコード
function decodeMvt(buf) {
  return new VectorTile(new Pbf(new Uint8Array(buf)));
}

import * as mlt from "https://esm.sh/@maplibre/mlt";

// MLTデコード
function decodeMlt(buf) {
  return mlt.decodeTile(new Uint8Array(buf));
}
```

単体のタイルで比較するため、皇居を包含する `14/14552/6451` のタイルを抽出します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/42e5dac3-4bba-4a86-9dff-629b0c528fb6.png" width="300px" />

`sqlite3` で以下のとおり抽出します。あらかじめgzipを解凍しておきます。

```bash
sqlite3 building.mbtiles \
"SELECT writefile('building-14-14552-6451.mvt', tile_data)
 FROM tiles
 WHERE zoom_level=14
   AND tile_column=14552
   AND tile_row=9932;"

gunzip -c building-14-14552-6451.mvt > building-14-14552-6451.raw.mvt
```

なお、mbtilesはTMSベースでタイルが格納されているため、XYZベースのタイル座標とはY座標が異なることに注意する必要があります。

具体的には `Y(TMS) = 2**z - Y(XYZ) - 1` となります。

### 結果サマリー

それぞれのタイルで、300回ずつデコード関数を実行したところ、処理完了までの平均時間は以下のとおりとなりました。

| type | MVT average size | MLT average size | MLT/MVT ratio(%) |
| :--- | :--- | :--- | :--- |
| building | 0.54ms | 0.26ms | 48% |
| landuse | 0.40ms | 0.24ms | 60% |
| road | 2.31ms | 0.89ms | 39% |
| place | 1.21ms | 0.17ms | 14% |

## 考察

上記のサイズ比較及びデコード速度比較から、以下のことがいえそうです。

- サイズはいずれも80%ほどの圧縮率となる。
- デコード速度には効果に大きな差異がみられ、place（ポイント）やroad（ライン）でより大きなパフォーマンス上の優位性が観察される。

これらの効果は、[Tile Layout](https://maplibre.org/maplibre-tile-spec/specification/#tile-layout)に記載されているとおり、

- MVTがレコード指向（Featureごとにデータ管理）であるのに対して、MLTが列指向であること
- MVTが固定的なエンコーディング手法（デルタ、zigzag、dictionaryエンコーディングなど）で圧縮されるのに対して、MLTではより柔軟にかつ効果的に様々なエンコーディング手法を適用できること

に主に起因しているものと思われます。

## おわりに

以上、MLTについて実施してみた試行実験を簡単に紹介させていただきました。

今回試したことや検討したこと以外でも多くの論点や特長を有しており、今後のどのように展開していくのか非常に楽しみですし、一技術者として（貢献は難しいかもしれませんが）、引き続きフォローしていきたいと思います。

全体的に踏み込み不足感はあるかと思いますが、本記事がMLT理解の一助になれば幸いです。
