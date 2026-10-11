---
title: MartinでGeoParquetからMVTを配信する
tags:
  - Martin
  - MapLibre
  - mvt
  - GeoParquet
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

Martinのv2.0からDuckDB GeoParquetソースが追加されました。

https://github.com/maplibre/martin/releases#release-martin-v2.0.0-beta.1

これは、DuckDB経由でGeoParquetから動的にMVTを生成して配信する機能です。

本記事では、この機能について紹介したいと思います。

# DuckDBソース機能オプションをつけてMartinをビルドする

[こちら](https://maplibre.org/martin/sources-duckdb/)に記載があるとおり、本機能は現在unstableであるため、デフォルトビルドには含まれていません。

試すには、unstableフィーチャオプションをつけてビルドする必要があります。

私のWSL（Ubuntu）の環境では、依存解決のため、事前にいくつかパッケージをインストール必要がありました。

それと合わせて実行すると以下のようになります。

```bash
sudo apt update

sudo apt install -y \
  build-essential \
  pkg-config \
  libssl-dev \
  clang \
  cmake \
  libcurl4-openssl-dev \
  libjpeg-dev \
  libpng-dev \
  libuv1-dev \
  libwebp-dev \
  glslang-dev \
  spirv-tools
  
cargo install martin --locked --features=unstable-duckdb
```

# 動作確認用に国土数値情報の学校データを準備

以下からデータをダウンロードします。

https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-P29.html

文字コードがShift-JISになっているため、UTF-8に変換します。

また、座標系がJGD2000のため、WGS84に変換しておきます。

変換には[ogr2ogr](https://gdal.org/en/stable/programs/ogr2ogr.html)を使い、GeoJSONとして出力します。

```bash
ogr2ogr \
  -f GeoJSON \
  schools.geojson \
  P29-13.shp \
  -oo ENCODING=CP932 \
  -s_srs EPSG:4612 \
  -t_srs EPSG:4326
```

# GeoParquetに変換

[DuckDB](https://duckdb.org/)を使って、GeoParquetに変換します。

DuckDBの最新版をインストールします。

```bash
curl https://install.duckdb.org | bash
```

GeoParquetを作成する際のrow groupサイズの参考にするため、データの件数を調べておきます。

```bash
duckdb
```
```sql
INSTALL spatial;
LOAD spatial;

SELECT COUNT(*) FROM ST_Read('schools.geojson');
> 39689
```

約4万件のため、GeoParquetのrow groupを5000件ずつで8グループに分割することにします。

row group pruningの効果を最大化するため、ヒルベルト順にソートして空間的に近い地物が連続して並ぶようにします。

```sql
COPY (
    SELECT *
    FROM ST_Read('schools.geojson')
    ORDER BY ST_Hilbert(geom)
)
TO 'schools.parquet'
(
    FORMAT PARQUET,
    GEOPARQUET_VERSION 'V2',
    COMPRESSION 'ZSTD',
    ROW_GROUP_SIZE 5000
);
```

# Martin設定ファイル（config.yaml）を作成

Martinの設定ファイルにDuckDBソースの設定と定義を記載します。

書き方は[こちら](https://maplibre.org/martin/sources-duckdb/#run-martin-with-configuration-file)を参照します。

```yaml
duckdb:
  pool_size: 4
  threads: 2
  memory_limit_mb: 1024
  auto_bounds: quick

  sources:
    - geoparquet: schools.parquet
      layer_id: schools
      geometry_column: geom
      srid: 4326
      minzoom: 0
      maxzoom: 14
      extent: 4096
      buffer: 64
      clip_geom: true
```

# Martin起動

以下のコマンドでMartinを起動します。

```bash
martin --config config.yaml
```

タイルリクエストのエンドポイントは次のようになります。

```
http://localhost:3000/schools/{z}/{x}/{y}
```

# MapLibreで可視化してみる

MapLibre GL JSで、例えば、以下のようにMapオブジェクトを作成するとOpenStreetMapsをベースマップとしつつ学校のポイントデータを表示できます。

```javascript
const map = new Map({
  container: 'map',
  hash: true,
  style: {
    version: 8,
    glyphs: 'https://tile.openstreetmap.jp/fonts/{fontstack}/{range}.pbf',
    sources: {
      basemap: {
        type: 'raster',
        tiles: ['https://tile.openstreetmap.org/{z}/{x}/{y}.png'],
        tileSize: 256,
        maxzoom: 19,
        attribution: '&copy; <a href="https://www.openstreetmap.org/copyright" target="_blank">OpenStreetMap contributors</a>'
      },
      schools: {
        type: 'vector',
        tiles: ['http://localhost:3000/schools/{z}/{x}/{y}'],
        minzoom: 0,
        maxzoom: 14,
        attribution: "<a href=\"https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-P29.html\" target=\"_blank\">国土数値情報-学校データ</a>"
      }
    },
    layers: [
      {
        id: 'basemap',
        type: 'raster',
        source: 'basemap',
        paint: {
          'raster-opacity': 0.3
        }
      },
      {
        id: 'schools',
        type: 'circle',
        source: 'schools',
        'source-layer': 'schools',
        paint: {
          'circle-radius': 5,
          'circle-color': '#e63946'
        }
      },
      {
        id: 'school-labels',
        type: 'symbol',
        source: 'schools',
        'source-layer': 'schools',
        minzoom: 13,
        layout: {
          'text-field': ['get', 'P29_005'],
          'text-font': ['Noto Sans JP Regular'],
          'text-size': 12,
          "text-variable-anchor": ["top", "left"],
          "text-justify": "auto",
          "text-radial-offset": 0.5
        },
        paint: {
          'text-color': '#222222',
          'text-halo-color': '#ffffff',
          'text-halo-width': 1.5
        }
      }
    ]
  },
  center: [139.767125, 35.681236],
  zoom: 10
});
```

ブラウザで確認すると以下のような表示になります。

![MapLibreで可視化](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/9ca55a26-9e7b-49d2-b553-b6f31e3bc7a2.png)

ローカルで動かしているというのもありますが、このくらいのデータであればほとんど間断なく表示されます。

# まとめ

GeoParquet対応はリリースされたばかりで、unstable機能の位置づけであるため、現状ではテスト的に使うのが無難かと思われます。

しかしながら、事前にタイル化しておく必要がなく、データベースも必要ないため、管理コストや性能などの観点で非常に有用な機能になる可能性があります。

今後の動向を注視しながら実際の運用に耐えうる機能になるか引き続き検証していきたいと思います。