---
title: イコールアース投影のウェブ地図を作る
tags:
  - Openlayers
  - EqualEarth
  - webGIS
  - PostGIS
private: false
updated_at: '2026-09-13T13:43:20+09:00'
id: 1c54bcbc56e72f1269f2
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

2026年9月4日に国連総会で「イコールアース投影の世界地図の使用を推奨する」決議が採択されました。

一般的に使用されているメルカトル投影は、高緯度地域で面積の歪みが大きくなるという欠点がありますが、イコールアース投影は、面積比を正確に表現できるため、例えば、アフリカ大陸が実際よりも相対的に小さく描画されることはありません。

そのため、アフリカ諸国を中心に、地図による認知的歪みを解消するべく、今回の決議が提起され採択されるに至ったようです。

ウェブ地図の世界でも、現在はメルカトル投影（をウェブ地図に最適化したウェブメルカトル投影）が支配的ですが、今後イコールアース投影が使われる場面もあるかもしれないというところで、今の技術で簡便にイコールアース投影のウェブ地図を作成する方法を紹介します。

# PostGISでMVTを作成

まず、イコールアース投影の地図データを `MVT(Mapbox Vector Tile)` 形式で作成します。

`MVT` はウェブ地図の世界においてデファクトスタンダードとなっている地図フォーマットです。

`MVT` を作成するツールとしては、 [tippecanoe](https://github.com/felt/tippecanoe) や [planetiler](https://github.com/onthegomap/planetiler) がよく知られていますが、いずれも標準ではウェブメルカトル投影（EPSG:3857）のXYZピラミッドに依存しており、イコールアース投影を出力グリッドとして指定する仕組みは用意されていません。

そのため、今回は、PostgreSQLの地理空間拡張モジュールである `PostGIS` を使って `MVT` を作成します。

`PostGIS` であれば、タイル範囲のジオメトリを任意の投影座標範囲から切り出すことができるなど、比較的柔軟かつ容易に異なる座標系の地図データを処理することができます。

# Natural EarthのCountriesデータをPostGISに投入

地図データとしては、[Natural EarthのCountries](https://www.naturalearthdata.com/downloads/10m-cultural-vectors/10m-admin-0-countries/)を使用します。

Natural Earthはパブリックドメインで提供されており、ライセンスフリーで利用することができるグローバルな地図ベクターデータです。

ダウンロードすると、シェープファイルが格納されているので、それをPostgreSQLのテーブルに投入します。

ただ、その前にPostgreSQLで格納用のテーブルを作成しておきます。

```sql
-- データベースでPostGISを有効化（未実施の場合）
CREATE EXTENSION postgis;

-- テーブル作成（プロパティに国名を入れておきたいのでnameカラムも作成しておく）
CREATE TABLE public.countries (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name text,
    geom geometry(MultiPolygon, 8857)
);

-- 空間インデックス作成
CREATE INDEX countries_geom_gix
    ON public.countries
    USING GIST (geom);
```

ここで、ジオメトリの座標系はEPSG:8857に設定しています。

EPSG:8857はイコールアース投影（中央経線0度）の識別コードです。

詳細は [epsg.io](https://epsg.io/8857) を参照してください。

テーブルが作成できたら、GDALの [ogr2ogr](https://gdal.org/en/stable/programs/ogr2ogr.html) でシェープファイルをPostgreSQLに変換・格納します。

```bash
ogr2ogr \
  -f PostgreSQL \
  PG:"host=localhost port=5432 dbname=postgres user=postgres" \
  ne_10m_admin_0_countries.shp \
  -nln public.countries \
  -append \
  -t_srs EPSG:8857 \
  -nlt MULTIPOLYGON \
  -sql "SELECT NAME AS name FROM ne_10m_admin_0_countries"
```

問題なくジオメトリがインポートされたか確認するため、座標範囲を見ておきます。

```sql
SELECT
    ST_XMin(extent) AS xmin,
    ST_YMin(extent) AS ymin,
    ST_XMax(extent) AS xmax,
    ST_YMax(extent) AS ymax
FROM (
    SELECT ST_Extent(geom) AS extent
    FROM public.countries
) t;

>> -16921334.187924642	-8392927.59846645	17125347.349335045	8315958.489934842
```

上記のような範囲になっていれば問題ないです。

# 正方グリッドでタイル化する

MVTに変換するにあたり、一つ留意点があります。

MVTのエコシステムは、正方タイルを前提としているため、タイルを正方形で切り出す必要があります（必須ではありませんが、それが無難です）。

一般的に広く使われているウェブメルカトル投影は、全体がちょうど正方形に収まるように設計されているため、特にそこを意識することはありませんが、イコールアース投影はそのままでは横長の矩形になるため、正方形にするには縦方向に余白を入れる必要があります。

具体的には以下のようなイメージです。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/ebe57416-68b3-4e21-8985-9c1e025cb21e.png" alt="EqualEarth" width="400px" />

イコールアースの投影座標の範囲は

```
X: -17,243,959.06 ～ 17,243,959.06
Y:  -8,392,927.60 ～  8,392,927.60
```

であるため、正方形にしたタイルグリッドの範囲は

```
X: -17,243,959.06 ～ 17,243,959.06
Y: -17,243,959.06 ～ 17,243,959.06
```

になります。

# MVTに変換する

変換イメージを共有できたかと思いますので、実際にMVTに変換していきます。

以下の手順で変換します。

1. タイル座標(z, x, y) --> 投影座標範囲を計算
2. ジオメトリを投影座標範囲でクリップしてMVT座標に変換
3. ST_AsMVTでMVT仕様のProtocolBuffersに変換

## 1. タイル座標(z, x, y) --> 投影座標範囲を計算

以下のSQLでタイル座標（z, x, y）から当該タイルが包含する投影座標の範囲を計算します。

```sql
WITH bounds AS (
    SELECT
        ST_TileEnvelope(
            z,
            x,
            y,
            ST_MakeEnvelope(
                -17243959.06,
                -17243959.06,
                17243959.06,
                17243959.06,
                8857
            )
        ) AS geom,

        ST_TileEnvelope(
            z,
            x,
            y,
            ST_MakeEnvelope(
                -17243959.06,
                -17243959.06,
                17243959.06,
                17243959.06,
                8857
            ),
            margin => 0.015625
        ) AS query_geom
)
```

ここで、`query_geom` は検索で使用する座標範囲のジオメトリになります。

マージンを入れているのは、MVTでは、タイル範囲よりも少し広めにジオメトリを保有するからです。少し広めにジオメトリを格納しておくことで、フィーチャがタイル境界で滑らかに描画されるようにしています。

MVTは通常4,096 x 4,096の正規化座標でジオメトリを扱います。

タイル周囲に64座標分マージンを取るとすると、割合にして `64 / 4,096 = 0.015625` になるため、ここではそのようにmarginを設定しています。

## 2. ジオメトリを投影座標範囲でクリップしてMVT座標に変換

この検索用の座標範囲を使って空間インデックス（bbox）でタイルに格納するジオメトリ候補を抽出した上で、[ST_AsMVTGeom](https://postgis.net/docs/ja/ST_AsMVTGeom.html) でタイル座標範囲（＋バッファー）に切り出したMVT座標のジオメトリを作成します。

```sql
countries_geom AS (
    SELECT
        c.id,
        c.name,

        ST_AsMVTGeom(
            c.geom,
            b.geom,
            4096,
            64,
            true
        ) AS geom

    FROM public.countries AS c
    CROSS JOIN bounds AS b

    WHERE c.geom && b.query_geom
)
```

最後の引数の `true` はタイル境界でクリップするという意味です。

## 3. ST_AsMVTでMVT仕様のProtocolBuffersに変換

最後に[ST_AsMVT](https://postgis.net/docs/ja/ST_AsMVT.html)でMVT仕様のProtocolBuffers形式にエンコードします。

```sql
SELECT
    ST_AsMVT(
        q,
        'countries',
        4096,
        'geom',
        'id'
    ) AS data

FROM (
    SELECT
        id,
        name,
        geom

    FROM countries_geom

    WHERE geom IS NOT NULL
) AS q
```

`id` をフィーチャIDとして格納し、`name` を属性値として格納しています。
レイヤー名は `countries` にしています。

以上の処理をすることで、MVTを作成できます。

これらの処理を必要なすべてのタイルについて実行します。

PMTilesでアーカイブする処理まで入れたPythonスクリプトを以下におきました。

https://github.com/hirofumikanda/equal-earth-pmtiles-generator

# OpenLayers + proj4 で可視化する

データを作れたので、ウェブ地図として表示します。

ウェブ地図ライブラリは、[OpenLayers](https://openlayers.org/)を使用します。

MVTの描画には、[MapLibre GL JS](https://maplibre.org/maplibre-gl-js/docs/)が使われることが多いですが、`MapLibre GL JS` はウェブメルカトル投影によるタイル座標を前提とした仕様となっているため、その他の投影法で作成されたタイルを表示するのは標準機能では困難です（ただしイコールアース投影に対応したプラグインである [maplibre-gl-equal-earth](https://github.com/pka/maplibre-gl-equal-earth)を利用すれば可能と思われます）。

他方、`OpenLayers` であれば、異なる投影法やタイルグリッドを独自に定義することができるため、標準機能でイコールアース投影のタイルデータを表示することも可能です。

以下の手順で処理します。

1. イコールアース投影を porj4 で定義
2. タイルグリッド作成
3. MVT読み込み

## 1. イコールアース投影を porj4 で定義

`OpenLayers` は、標準ではイコールアース投影を認識できないため、 `proj4.defs()` で定義した後、 `register()` でOpenLayersに登録します。

```javascript
import proj4 from "proj4";
import { register } from "ol/proj/proj4.js";
import { get as getProjection } from "ol/proj.js";

const equalEarthDefinition = [
  "+proj=eqearth",
  "+lon_0=0",
  "+x_0=0",
  "+y_0=0",
  "+datum=WGS84",
  "+units=m",
  "+no_defs",
  "+type=crs",
].join(" ");

proj4.defs("EPSG:8857", equalEarthDefinition);
register(proj4);

const projection = getProjection("EPSG:8857");

const projectionExtent = [
  -17243959.06,
  -8392927.60,
  17243959.06,
  8392927.60,
];

projection.setExtent(projectionExtent);

// 表示範囲を制限する場合
const view = new View({
  projection,
  center: [0, 0],
  zoom: 1,
  extent: projectionExtent,
});
```

表示範囲をイコールアースの投影範囲に制限したい場合（余白部分を見せたくない場合）は、`View` の `extent` に投影範囲を設定します。

## 2. タイルグリッド作成

MVT作成時にイコールアース投影の世界幅を基準にした正方のタイルグリッドを採用しているため、それに合わせてここでもタイルグリッドを定義します。

```javascript
import TileGrid from "ol/tilegrid/TileGrid.js";

const tileSize = 256;
const worldHalf = 17243959.06;

const tileGrid = new TileGrid({
  extent: projectionExtent,
  origin: [-worldHalf, worldHalf],
  tileSize,
  minZoom: 0,
  maxZoom: 10,
  resolutions: Array.from(
    { length: 11 },
    (_, zoom) => (worldHalf * 2) / tileSize / Math.pow(2, zoom),
  ),
});
```

## 3. MVT読み込み

MVTデータを読み込むため、`VectorTileSource` オブジェクトでデータソースを定義します。
フォーマットは `MVT` として、投影法及びタイルグリッドはこれまで定義してきたものを設定します。

PMTilesからのタイル取得には `PMTiles API` を使います。

```javascript
import PMTiles from "pmtiles";
import VectorTileSource from "ol/source/VectorTile.js";
import MVT from "ol/format/MVT.js";

const archive = new PMTiles(
  `${import.meta.env.BASE_URL}countries.pmtiles`,
);

const format = new MVT();

const source = new VectorTileSource({
  format,
  projection,
  tileGrid,

  tileUrlFunction(tileCoord) {
    const [z, x, y] = tileCoord;
    return `${z}/${x}/${y}`;
  },

  tileLoadFunction(tile, url) {
    const [z, x, y] = url.split("/").map(Number);

    archive
      .getZxy(z, x, y)
      .then((response) => {
        if (!response) {
          tile.setFeatures([]);
          return;
        }

        const features = format.readFeatures(response.data, {
          extent: tile.getExtent(),
          featureProjection: projection,
        });

        tile.setFeatures(features);
      })
      .catch((error) => {
        console.error("Failed to load PMTiles tile:", error);
        tile.setFeatures([]);
      });
  },
});
```

これらの処理で作成したウェブ地図サイトは以下になります（画像をクリックするとリンクに飛びます）。

<a href="https://hirofumikanda.github.io/equal-earth-projection-map/" target="_blank">
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/894347cd-6282-465d-b512-77843d44ee38.png" alt="EqualEarth" width="400px" />
</a>

これでイコールアース投影の地図をウェブ地図として表示することができました。

# まとめ

国連決議で話題になったイコールアース投影について、以下の方法でウェブ地図として表示するためのデータ及びビューワを作成できます。

- PostGISでイコールアース投影のMVTを作成
- OpenLayers+proj4で投影法とタイルグリッドを独自定義して可視化

MapLibre GL JSのプラグインなどを使えばより簡便に描画できるかもしれませんが、投影法への理解を深めるためにも、一度データ作成からタイルグリッドの定義まで自身で挑戦してみてもよいかもしれません。
