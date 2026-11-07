---
title: GeoParquet v2.0 について
tags:
  - geoparquet
  - Parquet
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

GeoParquetはApache Parquetで地理空間データを保存する際に、地理空間ツールが処理しやすいように実装されるべきメタデータなどの仕様を定めたものです。

以前は、Apache Parquetに地理空間データ型は定義されていなかったため、独自のメタデータで地理空間データの意味付けをした上で各行にstruct型のbbox列を持つことで、通常のmin/max統計を使って疑似的な空間統計を実現していました。

しかしながら、2025年3月にリリースされたApache Parquet v2.11に地理空間データ型と空間統計が定義されたことで、GeoParquet v2.0（現時点ではRC）では、このParquetネイティブな地理空間データ型と空間統計を使用する形に仕様が変わりました。

本記事では、この仕様変更による違いを[geoparquet-io](https://geoparquet.io/)を使って確認していきたいと思います。

# 国土数値情報をダウンロード

サンプルデータとして国土数値情報の以下のデータを使います。

https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-N03-2025.html

https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-N02-2025.html

https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-P29-2023.html

ダウンロードしたら個別のディレクトリに展開しておきます。

```bash
unzip data/N03-20250101_GML.zip -d data/n03

unzip data/N02-25_GML.zip -d data/n02

unzip data/P29-23_GML.zip -d data/p29
```

# geoparquet-io をインストール

以下のコマンドで geoparquet-io をインストールします。

geoparquet-io は、GeoParquet に変換したり、GeoParquet の内容やメタデータなどを表示するツールです。

```bash
python -m venv .venv
source .venv/bin/activate

pip install geoparquet-io
```

# GeoParquet v1.1 で変換

geoparquet-io を使ってシェープファイルを GeoParquet に変換します。

比較のため、まず、v1.1 仕様で変換します。

`--geoparquet-version` オプションをつけることでバージョンを指定できます。

```bash
gpio convert \
  data/n03/N03-20250101.shp \
  output/n03_v1.1.parquet \
  --geoparquet-version 1.1

gpio convert \
  data/n02/N02-25_GML/UTF-8/N02-25_RailroadSection.shp \
  output/n02_v1.1.parquet \
  --geoparquet-version 1.1

gpio convert \
  data/p29/P29-23_GML/P29-23.shp \
  output/p29_v1.1.parquet \
  --geoparquet-version 1.1
```

# GeoParquet v2.0 で変換

v2.0 でも変換します。

```bash
gpio convert \
  data/n03/N03-20250101.shp \
  output/n03_v2.0.parquet \
  --geoparquet-version 2.0

gpio convert \
  data/n02/N02-25_GML/UTF-8/N02-25_RailroadSection.shp \
  output/n02_v2.0.parquet \
  --geoparquet-version 2.0

gpio convert \
  data/p29/P29-23_GML/P29-23.shp \
  output/p29_v2.0.parquet \
  --geoparquet-version 2.0
```

# ファイルサイズを比較

それぞれのファイルサイズは以下のようになりました。

| データ | v1.1 | v2.0 |
| --- | --- | --- |
| 行政区域 | 166 MB | 163 MB |
| 鉄道 | 3.5 MB | 3.1 MB |
| 学校 | 3.7 MB | 2.1 MB |

いずれも v2.0 の方がファイルサイズが小さくなっています。

とりわけ、学校のポイントデータの削減率が大きくなっていることがわかります。

これは、データの中身をみるとよくわかります。

geoparquet-io を使って最初の5件のデータを表示してみます。

GeoParquet v1.1 のデータには

```bash
gpio inspect head output/p29_v1.1.parquet 5 --no-truncate
```

```
Preview (first 5 rows):
┏━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ OGC_FID ┃ P29_001 ┃ P29_002       ┃ P29_003 ┃ P29_004                  ┃ P29_005                            ┃ P29_006 ┃ P29_007 ┃ P29_008 ┃ P29_009 ┃ geom                                          ┃ bbox                                          ┃
┡━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┩
│ 55983   │ 47382   │ B147238200013 │ 16001   │ 与那国町立与那国小学校   │ 沖縄県八重山郡与那国町与那国1025   │ 3       │ 1       │ 00      │ NULL    │ POINT (123.00805551509191 24.467627918577815) │ [123.008056, 24.467628, 123.008056, 24.467... │
│ 56132   │ 47382   │ C147238200011 │ 16002   │ 与那国町立与那国中学校   │ 沖縄県八重山郡与那国町字与那国1049 │ 3       │ 1       │ 00      │ NULL    │ POINT (123.00849000049952 24.468517999185213) │ [123.008490, 24.468518, 123.008490, 24.468... │
│ 55533   │ 47382   │ A147238200015 │ 16011   │ 与那国町立よなぐに幼稚園 │ 沖縄県八重山郡与那国町与那国1025   │ 3       │ 0       │ 00      │ NULL    │ POINT (123.0088379965453 24.46749099652396)   │ [123.008838, 24.467491, 123.008838, 24.467... │
│ 55534   │ 47382   │ A147238200024 │ 16011   │ 与那国町立くぶら幼稚園   │ 沖縄県八重山郡与那国町与那国4022   │ 3       │ 0       │ 00      │ NULL    │ POINT (122.94472218834129 24.453729890843977) │ [122.944722, 24.453730, 122.944722, 24.453... │
│ 56133   │ 47382   │ C147238200020 │ 16002   │ 与那国町立久部良中学校   │ 沖縄県八重山郡与那国町与那国4022   │ 3       │ 1       │ 00      │ NULL    │ POINT (122.94354200134072 24.45506900392177)  │ [122.943542, 24.455069, 122.943542, 24.455... │
└─────────┴─────────┴───────────────┴─────────┴──────────────────────────┴────────────────────────────────────┴─────────┴─────────┴─────────┴─────────┴───────────────────────────────────────────────┴───────────────────────────────────────────────┘
```

bbox 列がありますが、GeoParquet v2.0 のデータには

```bash
gpio inspect head output/p29_v2.0.parquet 5 --no-truncate
```

```
Preview (first 5 rows):
┏━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ OGC_FID ┃ P29_001 ┃ P29_002       ┃ P29_003 ┃ P29_004                  ┃ P29_005                            ┃ P29_006 ┃ P29_007 ┃ P29_008 ┃ P29_009 ┃ geom                                          ┃
┡━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┩
│ 55983   │ 47382   │ B147238200013 │ 16001   │ 与那国町立与那国小学校   │ 沖縄県八重山郡与那国町与那国1025   │ 3       │ 1       │ 00      │ NULL    │ POINT (123.00805551509191 24.467627918577815) │
│ 56132   │ 47382   │ C147238200011 │ 16002   │ 与那国町立与那国中学校   │ 沖縄県八重山郡与那国町字与那国1049 │ 3       │ 1       │ 00      │ NULL    │ POINT (123.00849000049952 24.468517999185213) │
│ 55533   │ 47382   │ A147238200015 │ 16011   │ 与那国町立よなぐに幼稚園 │ 沖縄県八重山郡与那国町与那国1025   │ 3       │ 0       │ 00      │ NULL    │ POINT (123.0088379965453 24.46749099652396)   │
│ 55534   │ 47382   │ A147238200024 │ 16011   │ 与那国町立くぶら幼稚園   │ 沖縄県八重山郡与那国町与那国4022   │ 3       │ 0       │ 00      │ NULL    │ POINT (122.94472218834129 24.453729890843977) │
│ 56133   │ 47382   │ C147238200020 │ 16002   │ 与那国町立久部良中学校   │ 沖縄県八重山郡与那国町与那国4022   │ 3       │ 1       │ 00      │ NULL    │ POINT (122.94354200134072 24.45506900392177)  │
└─────────┴─────────┴───────────────┴─────────┴──────────────────────────┴────────────────────────────────────┴─────────┴─────────┴─────────┴─────────┴───────────────────────────────────────────────┘
```

bbox 列がありません。

特にポイントの場合は、ジオメトリがもつ座標とまったく同じ座標で bbox の各値をもつことになるため、冗長に感じられます。

# GeoParquet v1.1 では空間統計のため bbox列 が必要

GeoParquet v1.1 では、Parquet ネイティブの Geometry 型や空間統計は使っていないため、独自の方法で空間統計を実現しています。

それが先ほど見た bbox 列です。

これを各レコードに入れておくことで、row group ごとの min/max 統計で地理的な範囲に関する統計値を得ています。

例えば、行政区域データのGeoParquet v1.1 には以下のような統計値が入っています。

```bash
gpio inspect meta output/n03_v1.1.parquet --geo-stats --row-groups 3
```

```
Per-Row-Group geo_bbox Statistics
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
┏━━━━━━━━━━━┳━━━━━━━━┳━━━━━━━━━━━━┳━━━━━━━━━━━┳━━━━━━━━━━━━┳━━━━━━━━━━━┓
┃ Row Group ┃   Rows ┃       xmin ┃      ymin ┃       xmax ┃      ymax ┃
┡━━━━━━━━━━━╇━━━━━━━━╇━━━━━━━━━━━━╇━━━━━━━━━━━╇━━━━━━━━━━━━╇━━━━━━━━━━━┩
│         0 │ 49,152 │ 122.932606 │ 20.422746 │ 138.637105 │ 38.332223 │
│         1 │ 49,152 │ 132.176433 │ 32.908894 │ 141.972958 │ 39.409743 │
│         2 │ 25,790 │ 139.249444 │ 24.224532 │ 153.986675 │ 45.557243 │
└───────────┴────────┴────────────┴───────────┴────────────┴───────────┘
```

ここで、例えば

```sql
SELECT COUNT(*)
FROM read_parquet("output/n03_v1.1.parquet")
WHERE bbox.xmin <= 139.78
AND bbox.xmax >= 139.75
AND bbox.ymin <= 35.70
AND bbox.ymax >= 35.67
AND ST_Intersects(geometry, ST_MakeEnvelope(139.75, 35.67, 139.78, 35.70));
```

というクエリを実行すると、bboxでの絞り込みで Row Group 0 は検索対象から除外することができます。

これにより、計算コストがかかる ST_Intersects の処理対象を削減するという通常の空間統計を用いたクエリ実行の効率化と同様の効果を得ています。

# GeoParquet v2.0 では Parquet ネイティブな空間統計を保持

GeoParquet v2.0 には v1.1 のような bbox 列はありませんが、row group ごとの空間統計をもっています。

これを確認するため、Parquet ネイティブの地理空間メタデータを参照してみます。

```bash
gpio inspect meta output/n03_v2.0.parquet --parquet-geo --row-groups 3
```

```
Parquet Geo Metadata
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Reading from 3 row group(s)

  geom:
    Type: Geometry
    Geometry Type: Not present - geometry types are unknown
    CRS: {'$schema': 'https://proj.org/schemas/v0.5/projjson.schema.json', 'type': 'GeographicCRS', 'name': 'JGD2011', 'datum': {'type': 'GeodeticReferenceFrame', 'name': 'Japanese Geodetic Datum 2011',
'ellipsoid': {'name': 'GRS 1980', 'semi_major_axis': 6378137, 'inverse_flattening': 298.257222101}}, 'coordinate_system': {'subtype': 'ellipsoidal', 'axis': [{'name': 'Geodetic latitude', 'abbreviation':
'Lat', 'direction': 'north', 'unit': 'degree'}, {'name': 'Geodetic longitude', 'abbreviation': 'Lon', 'direction': 'east', 'unit': 'degree'}]}, 'scope': 'Horizontal component of 3D system.', 'area': 'Japan -
onshore and offshore.', 'bbox': {'south_latitude': 17.09, 'west_longitude': 122.38, 'north_latitude': 46.05, 'east_longitude': 157.65}, 'id': {'authority': 'EPSG', 'code': 6668}}
    Edges: N/A (only applies to Geography type)
    Overall Bbox: [122.932606, 20.422746, 153.986675, 45.557243]
    Row Group Statistics:
      Row Group 0:
        Bbox: [122.932606, 20.422746, 138.637105, 38.332223]
      Row Group 1:
        Bbox: [132.176433, 32.908894, 141.972958, 39.409743]
      Row Group 2:
        Bbox: [139.249444, 24.224532, 153.986675, 45.557243]
```

確かに、Row Group ごとに Bbox が設定されています。

このように v2.0 では、Parquet の方で定義された空間データ型と空間統計を利用することで、従前利用されていたbbox列を持つ必要がなくなっており、地理空間データとしてより洗練された仕様になっています。

# まとめ
- Apache Parquet v2.11 から地理空間データ型と空間統計が策定された。
- それを踏まえて GeoParquet v2.0 では Parquet ネイティブの地理空間データ型と空間統計を利用する仕様になった。
- その結果、v1.1 で疑似的に空間統計を実現するために使われていた bbox 列が不要になった。
- これにより冗長となっていたデータ構造が洗練された。
- ただし、v2.0 はいまだ rc 仕様であり、リーダーの対応状況を見て利用する必要がある点には留意が必要。