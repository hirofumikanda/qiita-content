---
title: MapLibre GL JSで国道番号をきれいに表示する
tags:
  - JavaScript
  - GIS
  - web地図
  - フロントエンド
  - MapLibre
private: false
updated_at: '2026-03-10T09:30:00+09:00'
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに

地図上で国道番号（`1`、`20`、`246` など）を表示するとき、背景アイコンをテキストの長さに合わせて単純に伸縮させると、特に1桁番号のときに横幅が狭くなりすぎてアイコン形状がいびつになることがあります。

MapLibre GL JSでは、`addImage` の `stretchX / stretchY / content` に加えて、`textFitWidth` / `textFitHeight` を `proportional` / `stretchOnly` にすることで、**縦横比を一定程度維持しながら伸縮**させることができます。

この記事では、国道番号のシールド表示を題材に、`proportional` の使い方を取り上げたいと思います。

なお、`stretchX / stretchY / content` の設定値を使ったシンプルな（特に制約を設けない）伸縮方法については、[MapLibre GL JSでストレッチ画像（9-slice）を表示する](https://qiita.com/k_hirofumi/items/e3eaf6c271f3e98c4bae)をご参照ください。

# アイコン画像の縦横比を維持する方法

通常の `icon-text-fit` はテキスト量に応じてアイコンを伸縮しますが、画像のメタデータの`textFitWidth` / `textFitHeight`を設定することで、伸縮に一定の制限を課すことができます。

MapLireの公式ドキュメントに[text-fit-properties](https://maplibre.org/maplibre-style-spec/sprite/#text-fit-properties)に詳細な説明が記載されています。

それによると、文字数が少なくても横幅が小さくなりすぎないように（具体的には伸縮前のアイコンの`content`の縦横比より小さくならないように）制約を加えたい場合は、以下のとおり設定することになります。

```
"textFitWidth": "stretchOnly",
"textFitHeight": "proportional"
```

適用例としてもシールドアイコンが挙げられているため、同様のアイコンをテキストにかぶせて表示する場合はこの設定をしておくのがよさそうです。

# 検証

それでは、早速試してみたいと思います。

今回は、[国土数値情報の道路データ](https://nlftp.mlit.go.jp/ksj/gmlold/datalist/gmlold_KsjTmplt-N01.html)を使って、属性から国土番号を拾ってきて道路上に表示したいと思います。

## 1. MVT作成

まず、ogr2ogrでシェープファイルをGeoJSONに変換します。

```bash
ogr2ogr -f GeoJSON N01-07L-2K_Road.geojson N01-07L-2K_Road.shp
```

路線名属性（`N01_002`）から国道の番号部分のみ抽出して、半角にした上で別途国道番号属性を追加します。
[duckdb](https://duckdb.org/)で、GeoJSONに属性を追加します。

```bash
load spatial;
create table road as select * from ST_Read('N01-07L-2K_Road.geojson');
alter table road add column number smallint;

UPDATE road
SET number = TRY_CAST(
    translate(
        regexp_replace(N01_002, '[^0-9０-９]', '', 'g'),
        '０１２３４５６７８９',
        '0123456789'
    ) AS SMALLINT
)
WHERE N01_001 = 2;

copy (select * from road) to 'road.geojson' (format gdal, driver 'GeoJSON');
```

[tippecanoe](https://github.com/felt/tippecanoe)でMVTに変換します。

```bash
tippecanoe -f -Q -o ./road.pmtiles -l road ./road.geojson -Z5 -z14 -pf -pk
```

## 2. sprite作成

画像ストレッチ関連のメタデータは、APIの`addImage`の`options`でも設定できますが、ロジックと分離するため、画像は１つですが、spriteを作成して、`sprite.json`でメタデータを設定したいと思います。

シールド画像はWIKIMEDIA COMMONSのパブリックドメインのアイコン[File:Kokudo.svg](https://commons.wikimedia.org/wiki/File:Kokudo.svg)を使用したいと思います。

spriteの作成は、国連ベクトルタイルツールキットの[sprite-one](https://github.com/unvt/sprite-one)を使います。

```bash
npm install -g @unvt/sprite-one

# iconディレクトリに対象のアイコン画像を格納
# 高DPIディスプレイ用のspriteを出力するため2x版も出力
sprite-one ./sprite -i ./icon -r 1 2
```

以下のとおりspriteが出来上がります。

```
sprite.json
sprite.png
sprite@2x.json
sprite@2x.png
```

`sprite.json`の中身を編集します。

```json
{
  "kokudo": {
    "height": 30,
    "width": 30,
    "x": 0,
    "y": 0,
    "pixelRatio": 1,
    "content": [8, 6, 22, 20],
    "stretchX": [[8, 22]],
    "stretchY": [[6, 20]],
    "textFitWidth": "stretchOnly",
    "textFitHeight": "proportional"
  }
}
```

`textFitWidth`と`textFitHeight`の設定値は冒頭述べたとおりです。
`content`、`stretchX`、`stretchY`の設定値は以下の黒線の領域に相当します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/22c6f7d0-6cc9-46b2-a693-01a808d5c0c1.png" width="50px">

2倍の値で`sprite@2x.json`も編集しておきます。

```json
{
  "kokudo": {
    "height": 60,
    "width": 60,
    "x": 0,
    "y": 0,
    "pixelRatio": 2,
    "content": [16, 12, 44, 40],
    "stretchX": [[16, 44]],
    "stretchY": [[12, 40]],
    "textFitWidth": "stretchOnly",
    "textFitHeight": "proportional"
  }
}
```

## 3. MapLibre GL JSで可視化

作成したMVTとspriteを使ってMapLibre GL JSで道路上に国道番号を表示してみます。

道路ライン上に一定間隔ごとに国道番号を表示するため、シンボル配置に関するスタイルを以下のとおり設定します。

```json
"symbol-placement": "line"
"symbol-spacing": 256
```

また、地図を回転してもアイコンを傾けないようにするため、rotation-alignmentを`viewport`で設定します。

```json
"icon-rotation-alignment": "viewport"
"text-rotation-alignment": "viewport"
```

そして、アイコンを伸縮させるため、`icon-text-fit`を設定しておきます。

```json
"icon-text-fit": "both"
```

これらのスタイル設定をして道路のラインなども表示させると以下のようになります。

<a href="https://hirofumikanda.github.io/road-map/" target="_blank">
  <img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/a16b384d-6ca0-4b15-95c9-acdb97a53f80.png" width="600px">
</a>

※画像をクリックすると地図サイトに飛びます

1桁国道でも扇形がすぼまり過ぎていないことがわかるかと思います。

このように画像の形をきれいに維持しつつ伸縮させたいときに有用な機能といえます。

# まとめ

MapLibre GL JSで国道番号を見やすく表示するなら、次の組み合わせが実用的です。

- 画像のオプションで `stretchX / stretchY / content` を適切に設定
- `textFitWidth` を `stretchOnly`、`textFitHeight` を `proportional` に設定
- `symbol` レイヤーで `icon-text-fit: "both"` を設定

桁数が変わるラベルでも、背景形状の破綻を抑えられます。
国道番号や県道番号のような可変長の表示で、ぜひ使ってみてください。
