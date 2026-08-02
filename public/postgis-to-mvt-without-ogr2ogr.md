---
title: PostGISからPMTiles（MVT）を効率的に作成する方法
tags:
  - MVT
  - ogr2ogr
  - PMtiles
  - PostGIS
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに

PostGISのジオメトリをMVTとして静的配信する場合、MBTileないしはPMTilesでタイルを事前に生成しておくことが多いと思います。

その際、以下の手順で作成することが通例かと思います。

1. ogr2ogrでGeoJSONに変換
2. （任意）ndjson-cliでminzoom, maxzoom, layer_nameをtippecanoeオブジェクトに付け替え
3. tippecanoeでMVT（MBTiles or PMTiles）に変換

この場合、フィーチャ数が多くなると、ディスクI/Oが多くなり、性能面の課題が顕在化することが少なくありません。

実際、私も1億を超えるフィーチャ数のジオメトリを変換しようとした際、処理に数時間かかったということがありました。

しかし、同僚から、この手順を見直すことで、大幅に処理を効率化させる方法を教わりましたので、本記事で共有したいと思います。

# 従前の方法（ogr2ogr方式）

本題に入る前に、まずは通常のogr2ogr方式をおさらいしたいと思います。

先日、個人的に作成した観光資源マップのPMTilesを例として説明したいと思います。

## 1. PostGISからGeoJSONに変換
まず、PostGISのジオメトリをGeoJSONに変換します。

ここでは、フィーチャ個別に最小ズームレベル、最大ズームレベルを設定するため、minzoomとmaxzoomも出力しています。

また、tippecanoeで-Pオプションで並列処理するため、GeoJSONSeqフォーマットで出力しています。

  ```bash
  ogr2ogr \
  -f GeoJSONSeq \
  tourism_resource.ndjson \
  PG:"host=localhost port=5432 dbname=postgres user=postgres" \
  -sql "
    SELECT
        prefecture,
        municipality,
        resource_type,
        category_name,
        resource_name,
        resource_rank,
        source,
        ST_Transform(geom, 4326) AS geom,
        minzoom,
        maxzoom
    FROM tourism_resources 
    WHERE geom IS NOT NULL
  " \
  -nln tourism_resource \
  -lco RS=NO
  ```

## 2. tippecanoeオブジェクト付与
フィーチャ個別に最小ズームレベル及び最大ズームレベルを設定するため、minzoom及びmaxzoomをtippecanoeオブジェクトに付け替えます。

ツールとして、ndjson-cliを使います。

```bash
cat tourism_resource.ndjson \
| ndjson-map 'd.tippecanoe = {
    minzoom: +d.properties.minzoom,
    maxzoom: +d.properties.maxzoom
  }, delete d.properties.minzoom,
     delete d.properties.maxzoom, d' \
> tourism_resource.tippecanoe.ndjson
```

## 3. MVTに変換
最後に、tippecanoeを使ってMVTに変換します。

Webでの取り扱いが簡単にできる、PMTilesでアーカイブします。

```bash
tippecanoe -f -Q -P \
    -o "tourism_resource.pmtiles"  \
    -l "tourism_resource" --drop-rate=0 \
    -Z4 -z14 -pf -pk \
    tourism_resource.tippecanoe.ndjson
```

# 新しい方法（psql方式）

それでは、ogr2ogrを使わないでMVT生成を効率化する方法をご紹介します。

## 1. GeoJSONを出力するSQL作成

このステップはなくてもよいのですが、見通しをよくするため、psqlに実行してもらうSQLをファイルとして作成しておきます。

tippecanoeに直接パイプで出力を渡すため、GeoJSONSeq形式でジオメトリ及びプロパティを出力するSQLを作成します。

`json_build_object` 関数で、JSONオブジェクトを作成し、`ST_AsGeoJSON` でPostGISのジオメトリをGeoJSONのgeometryオブジェクトに変換します。

ここで、minzoom及びmaxzoomをtippecanoeオブジェクトを組み入れておくことで、ogr2ogr方式での「2. tippecanoeオブジェクト付与」の工程をも省くことができます。

- `export_geojson.sql`

```sql
SELECT json_build_object(
  'type', 'Feature',
  'geometry', ST_AsGeoJSON(ST_Transform(geom, 4326))::json,
  'properties', json_build_object(
    'prefecture', prefecture,
    'municipality', municipality,
    'resource_type', resource_type,
    'category_name', category_name,
    'resource_name', resource_name,
    'resource_rank', resource_rank,
    'source', source
  ),
  'tippecanoe', json_build_object('minzoom', minzoom, 'maxzoom', maxzoom)
)::text
FROM tourism_resources
WHERE geom IS NOT NULL
```

## 2. SQL実行結果をtippecanoeに渡してMVT生成

psqlで上記で作成したSQLを実行し、その出力をtippecanoeにパイプで渡してMVT生成（PMTiles）まで一気通貫で処理します。

この際、psqlが余計な出力を含まないよう、オプションを工夫します。

```bash
psql \
	-X \
	-A \
	-t \
	-q \
	-v FETCH_COUNT=1000 \
	-v ON_ERROR_STOP=1 \
	-h "localhost" -p "5432" -d "postgres" -U "postgres" \
	-f "export_geojson.sql" \
| tippecanoe -f -Q -P \
    -o "tourism_resource.pmtiles"  \
    -l "tourism_resource" --drop-rate=0 \
    -Z4 -z14 -pf -pk
```

psqlのオプションの意味は以下のとおりです。

| オプション | 意味 |
| --- | --- |
| -X | `.psqlrc` を読み込まない |
| -A | 非整形出力 |
| -t | ヘッダーや件数表示を出さない |
| -q | 不要なメッセージを抑制 |
| -v FETCH_COUNT=1000 | 1000件ごとに取得（メモリ使用量を抑えるため） |

これにより、以下の効果が得られます。

- GeoJSONの読み書き相当のディスクI/Oを削減できる
- ndjson-cliを使ったtippecanoeオブジェクトを付与する処理が不要になる

# さいごに

私の実体験として、ご紹介した新しい方式を採用したことで、以前は数時間かかっていたPMTilesの出力処理が数十分で終わるようになったという事例がありました。

大量のフィーチャを扱う場合で、同様の課題を抱えているようなことがありましたら、参考にしてみていただければと思います。