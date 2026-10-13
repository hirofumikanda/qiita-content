---
title: MartinがComposite Sourcesで結合したgzipタイルを単一のgzip memberで返すようになったことについて
tags:
  - Martin
  - MVT
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

Martin2.0から、Composite Sourcesで複数のgzip圧縮タイルを連結する時に単一のgzip memberに変換して返却するようになりました。

https://github.com/maplibre/martin/pull/3382

これにより、複数のgzip memberに対応していないクライアントでも正常に処理できるようになりました。

この対応が実施される前は、MapLibre GL JSやQGISでも最初のgzip memberのレイヤしか表示できませんでした。

詳しくは以下の記事をご覧ください。

https://qiita.com/k_hirofumi/items/68e86e9243208f79223e

この記事では、本事象が解消されているのかを実際に確認してみたいと思います。

# テストデータの準備

国土数値情報の[湖沼データ](https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-W09-2005.html)と[行政区域データ](https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-N03-2026.html)でテストします。

それぞれダウンロードして、MVTに変換します。

```bash
# 湖沼データをUTF-8、EPSG:4326に変換してGeoJSONで出力
ogr2ogr \
  -f GeoJSON \
  lakes.geojson \
  W09-05-g_Lake.shp \
  -s_srs EPSG:4612 \
  -t_srs EPSG:4326 \
  -oo ENCODING=CP932

# MVTに変換
tippecanoe \
  -o tiles/lakes.mbtiles \
  -l lakes \
  -Z4 \
  -z14 \
  -pf \
  -pk \
  lakes.geojson

# 行政区域をEPSG:4326に変換
ogr2ogr \
  -f GeoJSON \
  admin.geojson \
  N03-20260101.shp \
  -s_srs EPSG:6668 \
  -t_srs EPSG:4326

# MVTに変換
tippecanoe \
  -o tiles/admin.mbtiles \
  -l admin \
  -Z4 \
  -z14 \
  -pf \
  -pk \
  admin.geojson
```

# Martinを起動

tilesディレクトリ配下に格納したMBTilesをMartinで配信します。

バージョンは最新の2.0.0-beta.3を使います。

```bash
docker run --rm \
  -p 3000:3000 \
  -v "$(pwd)/tiles:/tiles" \
  ghcr.io/maplibre/martin:2.0.0-beta.3 \
  /tiles
```

Composite Sourcesには以下のようにソースIDをカンマ区切りで連結したURLでアクセスできます。

```
http://localhost:3000/lakes,admin/{z}/{x}/{y}
```

# MapLibre GL JSで表示

例えば、以下のようにMapオブジェクトを定義します。

```javascript
const map = new Map({
  container: 'map',
  hash: true,
  style: {
    version: 8,
    sources: {
      composite: {
        type: 'vector',
        tiles: ['http://localhost:3000/lakes,admin/{z}/{x}/{y}'],
        minzoom: 4,
        maxzoom: 14,
        attribution: "<a href=\"https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-N03-2026.html\" target=\"_blank\">国土数値情報-行政区域データ</a> | <a href=\"https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-W09-2005.html\" target=\"_blank\">国土数値情報-湖沼データ</a>"
      }
    },
    layers: [
      {
        id: 'admin-fill',
        type: 'fill',
        source: 'composite',
        'source-layer': 'admin',
        paint: {
          'fill-color': '#eeeeee',
          'fill-opacity': 0.15
        }
      },
      {
        id: 'admin-boundary',
        type: 'line',
        source: 'composite',
        'source-layer': 'admin',
        paint: {
          'line-color': '#666666',
          'line-width': [
            'interpolate',
            ['linear'],
            ['zoom'],
            4, 0.4,
            8, 0.8,
            12, 1.2,
            16, 2
          ]
        }
      },
      {
        id: 'lakes-fill',
        type: 'fill',
        source: 'composite',
        'source-layer': 'lakes',
        paint: {
          'fill-color': '#9ecae1',
          'fill-opacity': 1.0
        }
      }
    ]
  },
  center: [139.767125, 35.681236],
  zoom: 10
});
```

ブラウザで表示すると下図のようになります。

![MapLibre GL JS](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/a9b07678-48ec-4a01-8a1f-f5899629f7be.png)

行政区域データと湖沼データの両方が問題なく表示されていることを確認できました。

# まとめ

Martin2.0からgzip圧縮したタイルのデータソースでもComposite Sourcesで互換性の高いタイルを供給できるようになりました。

タイル配信する際に、Composite Sourcesは効率性・保守性・性能などの面で非常に強力なツールになりえるため、地味な修正ではありますが、インパクトは大きいのではないかと思います。