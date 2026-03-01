---
title: 'MapLibre GL JSでストレッチ画像（9-slice）を表示する'
tags:
  - MapLibre
  - JavaScript
  - GIS
  - Web地図
  - フロントエンド
private: false
updated_at: '2026-02-23T12:00:00+09:00'
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに

MapLibre GL JSでは、`addImage` に **stretchX / stretchY / content** を指定することで、  
**画像の角を保ったまま中央だけ伸縮**させる「ストレッチ画像（9-slice的な挙動）」を実現できます。

これを使うと、ラベル背景画像をラベルの文字数に応じて綺麗に伸ばせます。

この記事では、その使い方と設定座標の注意点などを取り上げたいと思います。

# 基本概念（stretchX / stretchY / content）

基本的な使い方のイメージは、MapLibreの公式ドキュメントの[サンプル](https://maplibre.org/maplibre-gl-js/docs/examples/add-a-stretchable-image-to-the-map/)や[addImage](https://maplibre.org/maplibre-gl-js/docs/API/classes/Map/#addimage)のExampleが参考になります。

`map.addImage(name, image, options)` の `options` に以下の値を渡します。

- `stretchX`: 横方向で「伸ばしてよい区間（px）」の配列
- `stretchY`: 縦方向で「伸ばしてよい区間（px）」の配列
- `content`: テキストを配置してよい内側領域 `[left, top, right, bottom]`
- `textFitWidth`: 横方向の引き延ばしに関する制約。収縮を許容しない場合（`stretchOnly`）や縦横比を維持したい場合（`proportional`）に設定
- `textFitHeight`: 縦方向の引き延ばしに関する制約。収縮を許容しない場合（`stretchOnly`）や縦横比を維持したい場合（`proportional`）に設定

`options`の詳細は[StyleImageMetadata](https://maplibre.org/maplibre-gl-js/docs/API/type-aliases/StyleImageMetadata/)に記載があります。

レイヤ定義で`icon-text-fit`プロパティを使うことでストレッチが有効になります。

また、**sprite画像**を使う場合は、`sprite.json`で`options`を定義することも可能です。

詳細はMapLibre Style SpecificationのSpriteの[optional-properties](https://maplibre.org/maplibre-style-spec/sprite/#optional-properties)を参照してください。

# 設定値に関する留意点

`stretchX`、`stretchY`、`content`では、それぞれの対象領域を表すピクセル座標を設定しますが、これが本機能の肝になるため、正確に理解し設定する必要があります。

そのため、設定座標に関する留意点を以下にまとめます。

## ピクセル座標は0はじまり

設定する座標値は左上隅を起点として0始まりになります。

https://github.com/maplibre/maplibre-gl-js/blob/main/src/symbol/quads.ts

を参照すると、以下の記載があります。

```ts
const stretchX = image.stretchX || [[0, imageWidth]];
const stretchY = image.stretchY || [[0, imageHeight]];
```

デフォルトは0始まりになっており、起点は0であることがわかります。

## 範囲の終端座標は含まない

範囲の終端座標（right及びbottom）の値は範囲には含まれません。

すなわち`[start, end)`になります。

上記の`quads.ts`を再度参照すると、以下の記述があります。

```ts
const reduceRanges = (sum, range) => sum + range[1] - range[0];
const stretchWidth = stretchX.reduce(reduceRanges, 0);
const stretchHeight = stretchY.reduce(reduceRanges, 0);
```

範囲長を`range[1] - range[0] = end - start`で計算しています。

例えば、`stretchX: [[16, 584]]`と設定した場合、`x=16`から`(584 - 16) = 568`の長さの範囲ということになり、`x=16-583`の範囲を示します。

したがって、endの座標は包含されません。

# 検証

以上の点を踏まえて、実際にストレッチ画像を試してみたいと思います。

ここでは、[国土数値情報の鉄道データ](https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-N02-2024.html)を使って、駅名の背景画像をストレッチさせてみます。

## 1. MVT作成

### RailroadSection

[tippecanoe](https://github.com/felt/tippecanoe)でGeoJSONからMVTを出力します。

```bash
# 座標変換
ogr2ogr -s_srs EPSG:6668 -t_srs EPSG:4326 \
  N02-24_RailroadSection_4326.geojson N02-24_RailroadSection.geojson

# MVT変換
tippecanoe -f -Q -o ./railroad.pmtiles \
-l railroad ./N02-24_RailroadSection_4326.geojson -Z5 -z14 -pf -pk
```

### Station

Stationは、路線ごとに駅が格納されているので、煩雑さを避けるためグループコードの代表駅のみにデータを絞ります。
また、ラインで入っているため、重心でポイントデータを生成します。

データの加工には、[duckdb](https://duckdb.org/)を使います。

```bash
load spatial;
create table station as select * from ST_Read('N02-24_Station.geojson');

copy (select N02_003 as rail, N02_005 as name, ST_Centroid(geom) as geom \
from station \
where N02_005c in (select N02_005g from station group by N02_005g) \
) to 'station.geojson' (format gdal, driver 'GeoJSON');
```

これをMVTに変換します。

```bash
tippecanoe -f -Q -o ./station.pmtiles \
-l station --drop-rate=0 \
./station.geojson -Z5 -z14 -pf -pk
```

## 2. 画像作成

ストレッチさせる画像を作成します。

ここでは、簡易的に正方の画像（16px x 16px）で、枠線が黒色1pxの画像をペイントで作成します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/7149b17f-7533-4e09-a805-fcc3661244fb.png" width="600px" />

## 3. ストレッチテスト

作成した画像の外枠1pxからさらに内側1px分は余白として残しておき、その内側に文字列を配置するようにしたいと思います。

0始まりのピクセル座標で考えると、文字列を配置する領域は`x=2-13、y=2-13`です。

設定値の終端座標の値は包含されないことを踏まえると、画像のメタデータは次のとおり設定することになります。

```js
content: [2, 2, 14, 14],
stretchX: [[2, 14]],
stretchY: [[2, 14]],
```

シンボルレイヤーのスタイルで`icon-text-fit`を`both`に設定しておきます。

```json
"icon-text-fit": "both"
```

上記の設定で駅名を表示すると以下のとおりです。

<a href="https://hirofumikanda.github.io/railroad-map/" target="_blank">
  <img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/ed2ab5c8-a5cc-43a8-97e1-6e25abaa9d82.png" width="600px" />
</a>

※画像をクリックすると地図サイトに飛びます

文字数に応じて幅が変化しますが、枠線や余白の大きさは均一に保たれています。

このように画像をきれいに伸縮させたいときに便利な機能です。

## まとめ

MapLibre GL JSのストレッチ画像は、以下の2点を押さえるだけで実用できます。

- `addImage` で `stretchX` / `stretchY` / `content` を指定
  - 座標は0始まり
  - 終端座標は包含されない
- `symbol` レイヤーで `icon-text-fit: "both"` を設定

今回は、縦横独立に文字列に応じて伸縮させるようにしましたが、`textFitWidth`又は`textFitHeight`プロパティで`proportional`を使うと縦横比を維持しながらアイコンを伸縮させることも可能です。

ぜひ活用してみてください！