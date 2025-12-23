---
title: Martinのcomposite sourcesの有用性検証
tags:
  - GIS
  - Martin
  - MVT
  - MapLibre
private: false
updated_at: '2025-12-23T21:29:07+09:00'
id: 68e86e9243208f79223e
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

[Martin](https://maplibre.org/martin/)は、PostGISやMBTiles、PMTilesなどのデータソースからMapbox Vector Tiles（MVT）を配信する高速なタイルサーバーです。Rust製で非常に高いパフォーマンスを発揮します。

Martinの特徴的な機能の1つに[Composite Sources](https://maplibre.org/martin/sources-composite.html)があります。これは複数のタイルソースを1つのエンドポイントに統合する機能で、クライアント側のリクエスト数を削減することができます。

本記事では、Martinの**Composite Sources**の使い方とその有用性について検証してみたいと思います。

## Composite Sourcesとは

### 基本概念

通常、複数のタイルセットをデータソースとして地図に表示する場合、タイルセットごとに個別のタイルリクエストが発生します。

例えば、建物、交通、土地利用の3つのタイルセットを表示する場合、1タイル分表示するのに以下の3回のリクエストが発生します。

- /tiles/building/{z}/{x}/{y}
- /tiles/transportation/{z}/{x}/{y}
- /tiles/landuse/{z}/{x}/{y}

**Composite Sources**を使用すると、これらを1つのエンドポイントに統合することができます。

- /tiles/composite/{z}/{x}/{y}

タイルサーバー内部で3つのデータソースのタイルを結合し、1つのMVTとしてクライアントに返却します。その結果、1タイルあたり1回のHTTPリクエストで済むようになります。

実際に、Mapboxが提供しているstyleでは、同様の仕組みが使われています。

例えば、Mapboxの[Streets v12のstyle.json](https://github.com/mapbox/mapbox-gl-styles/blob/master/styles/streets-v12.json)をみると、以下の記述があります。

```
"sources":{
    "composite":{
        "url":"mapbox://mapbox.mapbox-streets-v8,mapbox.mapbox-terrain-v2,mapbox.mapbox-bathymetry-v2",
        "type":"vector"
    }
}
```

compositeソースのURLをみると、カンマ区切りで複数のタイルセットが設定されています。
実際にリクエストすると、統合された一つのタイルが返ってきます。

## MartinでComposite Sourcesを使うには

Martinでも同様に**Composite Sources**を利用できます。

上記の例と同様に、建物、交通、土地利用の3つのタイルセットをMartinで配信することを考えます。
この場合、以下のような設定(**config.yml**)で配信できます。

```yml
listen_addresses: '0.0.0.0:3000'

mbtiles:
  sources:
    building: ./data/building.mbtiles
    transportation: ./data/transportation.mbtiles
    landuse: ./data/landuse.mbtiles
```

各々のタイルURLは以下のようになります。

- **http://{domain}:3000/building/{z}/{x}/{y}**
- **http://{domain}:3000/transportation/{z}/{x}/{y}**
- **http://{domain}:3000/landuse/{z}/{x}/{y}**

これらを**Composite Sources**で配信する場合、[公式ドキュメント](https://maplibre.org/martin/sources-composite.html)をみるとわかるとおり、以下のURLでリクエストすることになります。

- **http://{domain}:3000/building,transportation,landuse/{z}/{x}/{y}**

Martinでは、カンマ区切りでソース名を列挙するだけで、Composite Sourcesを利用できます。
**設定(config.yml)を変更する必要はありません。**

## 実際に使ってみる

それでは、実際に**Composite Sources**でリクエストしてみたいと思います。

### データ（タイルセット）の準備

データは[Overturemaps（© OpenStreetMap contributors, Overture Maps Foundation）](https://docs.overturemaps.org/attribution/)の[building](https://docs.overturemaps.org/guides/buildings/)、[transportation](https://docs.overturemaps.org/guides/transportation/)、[landuse](https://docs.overturemaps.org/schema/reference/base/land_use/)を使用します。

feltの[tippecanoe](https://github.com/felt/tippecanoe)でMVTに変換します。

```bash
$ tippecanoe -f -Q -o overturemaps_building.mbtiles \
-l building --simplification=5 --simplify-only-low-zooms \
./overturemaps_building.geojson \
-Z9 -z16 -pf -pk

$ tippecanoe -f -Q -o ./overturemaps_transportation.mbtiles \
-l transportation --simplification=5 --simplify-only-low-zooms \
./overturemaps_transportation.geojson \
-Z9 -z14 -pf -pk

$ tippecanoe -f -Q -o overturemaps_landuse.mbtiles \
-l landuse --simplification=5 --simplify-only-low-zooms \
./overturemaps_landuse.geojson \
-Z9 -z15 -pf -pk
```

最大ズームレベルが異なる場合にどうなるかも確かめたいので、異なる最大ズームレベルを設定しています。

### Martinの環境構築

先ほど記載したとおり、Martinの設定ファイル(config.yml)を作成して、ローカルホストでMartinを動かしてみます。

[GitHub Releases](https://github.com/maplibre/martin/releases)から最新版(v1.1.0)のパッケージをダウンロードしてインストールします。

なお、テスト環境はWindows PC（Windows 11 Pro、Intel N95 1.7GHz、メモリ16GB）のWSL（Ubuntu）です。

```bash
$ wget https://github.com/maplibre/martin/releases/download/martin-v1.1.0/martin-Debian-x86_64.deb

$ sudo dpkg -i ./martin-Debian-x86_64.deb

$ martin --version
martin 1.1.0

$ martin --config ./config.yml
```

これでMartinがlocalhostで動きます。

### MapLibre GL JSで表示テスト

Composite Sourcesでタイルリクエストしたものを、MapLibre GL JSで表示してみます。
ここで、スタイル(style.json)のソースを以下のとおり設定します。

```json
"sources": {
  "overture_maps": {
    "type": "vector",
    "tiles": [
      "http://localhost:3000/building,landuse,transportation/{z}/{x}/{y}"
    ],
    "maxzoom": 16,
    "minzoom": 9,
    "attribution": "© OpenStreetMap contributors, Overture Maps Foundation"
  }
},
```

準備が整ったので、適当にスタイリングして表示してみると、以下のようになりました。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/ff13e6f6-0f52-432f-ac8f-429495f0c363.png" width="400px">

？？なんと、buildingしか表示されません（涙）
ズームレベル13なので、landuseやtransportationもデータがあるはずですが、なぜか表示されません。

しばらくこの事象の原因がわからなかったのですが、[Martinのissue](https://github.com/maplibre/martin/issues/1488)にヒントがありました。

どうやら圧縮されたタイルデータをデータソースにしている場合、使用するレンダラーによって（？）は2つ目以降のレイヤ（タイルセット）が表示されない事象が発生するようです。

ちなみにQGISで試してみても同様でした。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/31cb4458-c121-4a6b-868e-537bb2e2ea3f.png" width="400px">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/a5ac3a2b-719f-47d9-8237-d903f7ae8f0b.png" width="400px">

### 未圧縮のタイルセットを準備

というわけで、気を取り直して圧縮されていないタイルセットを準備します。

再度、tippecanoeを使って、-pCオプションを追加してタイルを作成してもよいのですが、今回はさきほど作成した圧縮タイルを未圧縮に変換して作成してみます。

tippecanoeに付属しているツールに[tile-join](https://github.com/felt/tippecanoe?tab=readme-ov-file#tile-join)というのがあり、複数データソースを一つのファイルに統合するツールなのですが、実は一対一のデータ変換にも使えます。

圧縮タイルを未圧縮に変換する場合は、-pCオプションをつけて以下のようなコマンドで変換できます。

```bash
$ tile-join --no-tile-size-limit -pC \
-o overturemaps_building_decompressed.mbtiles overturemaps_building.mbtiles

$ tile-join --no-tile-size-limit -pC \
-o overturemaps_transportation_decompressed.mbtiles overturemaps_transportation.mbtiles

$ tile-join --no-tile-size-limit -pC \
-o overturemaps_landuse_decompressed.mbtiles overturemaps_landuse.mbtiles
```

### 未圧縮タイルで再度テスト

未圧縮タイルが準備できたところで、再度、MapLibre GL JSで表示テストを実施してみます。

Martinの設定ファイル（config.yml）に未圧縮タイルのデータソースを追加します。

```yml
listen_addresses: '0.0.0.0:3000'

mbtiles:
  sources:
    building: ./data/overturemaps_building.mbtiles
    building_d: ./data/overturemaps_building_decompressed.mbtiles
    transportation: ./data/overturemaps_transportation.mbtiles
    transportation_d: ./data/overturemaps_transportation_decompressed.mbtiles
    landuse: ./data/overturemaps_landuse.mbtiles
    landuse_d: ./data/overturemaps_landuse_decompressed.mbtiles
```

これでMartinを再起動します。

```bash
$ martin --config ./config.yml
```

また、MapLibre GL JSのスタイル(style.json)のソースを以下のとおり未圧縮版の方に修正します。

```json
"sources": {
  "overture_maps": {
    "type": "vector",
    "tiles": [
      "http://localhost:3000/building_d,landuse_d,transportation_d/{z}/{x}/{y}"
    ],
    "maxzoom": 16,
    "minzoom": 9,
    "attribution": "© OpenStreetMap contributors, Overture Maps Foundation"
  }
},
```

これで再度表示してみると、以下のようになりました。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/a0c6484d-a042-447d-9011-edf535b4e7f4.png" width="400px">

今度は問題なく表示されました！
ついでにQGISでも表示してみます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/82e531f3-5aac-4d07-8ff2-c28e73ab799f.png" width="400px">

こちらも表示されるようになりました！

さきほどのIssueでは事象が再現されなくなったような記載があるのですが、少なくとも正式リリースされているバージョン（v1.1.0）では完全には解消されていないようです。

ということで、Composite Sourcesを前提としてタイルセットを提供する場合は、**ソースとなるタイルデータを未圧縮で用意する必要がある(かも)** といえそうです。

なお、今回試していませんが、PostGISをデータソースにする場合は、動的にMVTを作成するため、上記のような事象は発生しないものと思われます（知らんけど）。

### オーバーズームはしてくれるのか

さきほど述べたとおり、今回作成したデータソースはすべて最大ズームレベルが異なります。

- building・・・z9-16
- transportation・・・z9-14
- landuse・・・z9-15

MapLibre GL JSのスタイル（style.json）では、最も最大ズームレベルが大きいz16（building）に合わせましたが、この場合、他のデータソース（transportation、landuse）のタイルデータはどうなるのでしょうか。

そこで、transportaionやlanduseのデータソースにはタイルが存在しないz15やz16でも表示してみます。なお、スタイルレイヤには最大ズームレベルは一切設定していないので、composite sourcesではなく個別設定であれば、クライアント側のオーバーズーム機能で問題なく表示されるはずです。

- z15
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/ecac4a4a-08f7-46ed-a98b-7204c6d49e5a.png" width="400px">

- z16
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/9baa8bd8-c8b5-490c-9acf-96bb2e221a8c.png" width="400px">

transportationやlanduse由来のデータが表示されなくなりました。

このことから、Martin側ではデータソースに該当のタイルがなければ、compositeには含まない（低zのタイルをオーバーズームしていれるようなことはしない）ことがわかります（挙動として当然かもしれませんが）。

そのため、Composite Sourcesを利用するタイルセットは、**互いに最大ズームレベルを合わせておく必要**がありそうです。

## それで性能はいかほどか

これまでの検証から、

1. 未圧縮でデータソースを用意する
2. 最大ズームレベルは合わせておく

ことに気をつけておけば、Martinで簡単にComposite Sourcesを利用できることがわかりました。
ただ、プロダクション環境で使うには、性能的に使える機能かを確認しておく必要があります。

Composite Sourcesを使うということは、サーバ側で以下の処理が追加で発生することになります。

- **複数タイルを1枚のタイルに合成する処理**
- **合成後に、クライアントからのリクエストに応じてgzip等に圧縮する処理**

1つ目の処理は常に発生しますが、2つ目の処理は、ユーザーからのリクエスト内容に依存します。

具体的には、httpリクエストでヘッダーに**Accept-Encoding**がついている場合に限り、その圧縮方式に応じて2つ目の処理が発生します。

例えば、httpヘッダーに**Accept-Encoding: gzip**をつけてクライアントがリクエストすると、Martinはデータソースのタイルがgzip圧縮されていなくても、gzip圧縮処理をしてレスポンスします。

なぜ、この圧縮処理がcomposite sourcesを選択したことによる追加の処理という位置づけになるかというと、通常の（composite sourcesを使わない）個別タイルセットに対するリクエストでは、あらかじめgzip等で圧縮されたタイルデータをデプロイしておき、それをそのまま返せばよいからです。

composite sourcesを利用する場合に限り、未圧縮でデータソースを用意する必要があることから、この圧縮処理が（httpリクエストにAccept-Encodingがついていれば）必須になります。

ではこれらの処理にどのくらいの時間がかかるのか試してみます。

データは、引き続き、[Overturemaps（© OpenStreetMap contributors, Overture Maps Foundation）](https://docs.overturemaps.org/attribution/)の[building](https://docs.overturemaps.org/schema/reference/buildings/building/)、[transportation](https://docs.overturemaps.org/schema/reference/transportation/segment/)、[landuse](https://docs.overturemaps.org/schema/reference/base/land_use/)を使います。

### 性能テスト実行

簡易的に、curlを使ってhttpリクエストを投げてみて、レスポンスが返ってくるまでの時間を見てみたいと思います。

テストを実施する前にMartinのキャッシュを無効化しておきます。
設定ファイルにcache_size_mb: 0を追記します。

```yml
listen_addresses: '0.0.0.0:3000'

cache_size_mb: 0

mbtiles:
  sources:
    building: ./data/overturemaps_building.mbtiles
    building_d: ./data/overturemaps_building_decompressed.mbtiles
    transportation: ./data/overturemaps_transportation.mbtiles
    transportation_d: ./data/overturemaps_transportation_decompressed.mbtiles
    landuse: ./data/overturemaps_landuse.mbtiles
    landuse_d: ./data/overturemaps_landuse_decompressed.mbtiles
```

これで、Martinのキャッシュが効かなくなります。Martinを再起動しておきます。

```bash
$ martin --config ./config.yml
```

まず、通常の個別のタイルセットへのリクエストでかかる時間を以下のコマンドで測定します。

gzip圧縮されたタイルセットに対してリクエストするのが通常と思われるため、そのようにしています。

```bash
- building
$ time curl 'http://localhost:3000/building/13/7275/3226' \
-H 'Accept-Encoding: gzip' -o 3226.pbf

- transportation
$ time curl 'http://localhost:3000/transportation/13/7275/3226' \
-H 'Accept-Encoding: gzip' -o 3226.pbf

- landuse
$ time curl 'http://localhost:3000/landuse/13/7275/3226' \
-H 'Accept-Encoding: gzip' -o 3226.pbf
```

次に、composite sourcesで、統合されたタイルへのリクエストにかかる時間を以下のコマンドで測定します。対象のタイルセットは未圧縮の方を使います。その上で、1つ目がgzip圧縮をクライアントが要求した場合で、2つ目が要求しなかった場合です。

```bash
- composite sources(gzip圧縮あり)
$ time curl 'http://localhost:3000/building_d,transportation_d,landuse_d/13/7275/3226' \
-H 'Accept-Encoding: gzip' -o 3226.pbf

- composite sources(gzip圧縮なし)
$ time curl 'http://localhost:3000/building_d,transportation_d,landuse_d/13/7275/3226' \
-o 3226.pbf
```

それぞれ10回ずつ実施して、平均値を算出します。

### テスト結果

結果をまとめると下表のようになりました。

- 個別タイルセット

  | 種別 | タイルサイズ | 実行時間 |
  |---------|----------------|---------|
  | building | 110KB | 25±7ms |
  | transportation | 334KB | 22±2ms |
  | landuse | 35KB | 22±3ms |

- composite sources

  | 種別 | タイルサイズ | 実行時間 |
  |---------|----------------|---------|
  | composite(gzip圧縮あり) | 483KB | 137±13ms |
  | composite(gzip圧縮なし) | 955KB | 47±6ms |

ローカルホストで実行しているため、ネットワーク起因のレイテンシーはありません。

**composite sourcesの圧縮なし**と**個別タイルセット**の結果を比較すると、おおよそ20ms強の差があります。これが3つのデータソース（MBTiles）からタイルを取り出して、合成することによりかかる時間の差異といえます。composite sourcesでは、ファイルI/Oが3倍になることを考えると、**合成の処理にはほとんど時間がかかっていない**と思われます。

一方、**composite sourcesの圧縮あり**と**圧縮なし**の結果を比較すると、100ms弱の差があります。
これが合成後のタイルをgzip圧縮するのにかかる時間となります。

### 考察

composite sourcesの圧縮なしの方であれば、パフォーマンス的にもcomposite sourcesを利用することに意味がありそうです。

ただし、圧縮しないためその分トラフィックが増大しますし、モダンブラウザではAccept-Encodingがデフォルトで付加されるため、Webで利用する場合にはこの方法を採用するのは難しいと思われます。

他方、（モダンブラウザがそうであるように）Accept-Encoding付きでリクエストする場合、Martinで圧縮処理が駆動しますが、上の結果で確認したとおりパフォーマンスが大幅に劣化します。

以上から、composite sourcesをWebで利用する場合、gzip圧縮の効率化を別途検討する必要がありそうです。

例えば、CloudFrontの自動圧縮機能の利用などが考えられます。ただし、その場合でも、MartinがレスポンスするContent-typeがapplication/x-protobufであることから、[CloudFrontの自動圧縮が対応しているファイルタイプ](https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/ServingCompressedFiles.html#compressed-content-cloudfront-file-types)（application/protobuf）に含まれていないため、レスポンスヘッダーを書き換えるなどの対応が必要になるかもしれません。

## まとめ

- Martinを使えば、composite sourcesを容易に実現できる（データソース名称をカンマ区切りでリクエストすればOK）。
- ただし、どのレンダラーでも安定して動作させるには、データソースを未圧縮で作成しておく必要がある。また、最大ズームレベルを合わせておいた方がよい。
- （モダンブラウザではいずれもそうであるように）クライアントがAccept-Encodingに圧縮形式をセットしてリクエストすると、Martinから統合されたタイルが圧縮されてレスポンスされる。しかし、Martinの圧縮処理は性能に難がある。
- したがって、Martinのcomposite sourcesを利用する場合は、圧縮をMartinではないところ（Martinのデプロイ環境よりもフロント側）で実施する必要があるかもしれない（そこまで性能を求めない場合はその限りではない）。

以上から、個人的には、MartinでComposite Sourcesを（特にプロダクション環境で）利用するにはまだ制約条件や課題がありそうという印象です。

ただ、性能試験については、ローカル環境で簡易的に実施しただけなので、あくまでそういった課題が出てくるかもしれないというところでご理解いただければと思います。

以上です。ここまで読んでくださりありがとうございました！
