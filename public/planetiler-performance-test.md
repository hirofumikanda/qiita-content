---
title: Planetilerとtippecanoeの性能比較
tags:
  - planetiler
  - tippecanoe
  - MVT
private: false
updated_at: '2026-09-22T19:15:31+09:00'
id: e59c94ecff99dfb458f8
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
# はじめに

[Planetiler](https://github.com/onthegomap/planetiler)は、OpenStreetMapsなどをデータソースとした地理空間データをMVT（Mapbox Vector Tile）又はMLT（MapLibre Tile）に変換するツールです。

そのほかのMVT生成するツールに比べて、大容量・大規模な地理空間データを高速かつ効率的に処理できることを特長としています。

本記事では、MVTの生成にしばしば使用される[tippecanoe](https://github.com/felt/tippecanoe)との比較を通じて、Planetilerの性能を検証します。

# 検証環境

検証環境として、Windows11のWSL（Ubuntu）を使います。

スペックは以下のとおりです。

| 項目 | 内容 |
| -- | -- |
| CPU | Intel Core i5-7300U @ 2.60GHz（2コア4スレッド） |
| メモリ | 合計3.7GiB |
| スワップ | 合計1.0GiB |
| カーネル | 6.6.87.2-microsoft-standard-WSL2 |
| ディストリビューション | Ubuntu 24.04.3 LTS |

# Natural Earth

検証に使用するデータとして、[Natural EarthのOcean](https://www.naturalearthdata.com/downloads/10m-physical-vectors/10m-ocean/)を使用します。

データをダウンロードして解凍します。

```bash
wget https://www.naturalearthdata.com/http//www.naturalearthdata.com/download/10m/physical/ne_10m_ocean.zip

unzip ne_10m_ocean.zip -d ne_10m_ocean
```

# tippecanoeでMVT作成

まず、**tippecanoe** でMVTを作成してみます。

```bash
cd /usr/local/src
git clone https://github.com/felt/tippecanoe.git
cd tippecanoe
sudo apt update
sudo apt install g++ make git zlib1g-dev libsqlite3-dev
sudo make -j$(nproc)
sudo make install
tippecanoe --version
>> tippecanoe v2.80.0
```

tippecanoeはデータソースとしてGeoJSONを入力する必要があるため、ダウンロードしたシェープファイルをGeoJSONに変換します。

変換には、GDALの[ogr2ogr](https://gdal.org/en/stable/programs/ogr2ogr.html)を使います。

```bash
ogr2ogr -f GeoJSON ne_10m_ocean/ne_10m_ocean.geojson ne_10m_ocean/ne_10m_ocean.shp
```

その上でtippecanoeでMVTをPMTilesで出力します。最大ズームレベルは12とします。

所要時間を測定するため、`time`をつけておきます。

```bash
time tippecanoe \
  -o data/ocean_tippecanoe.pmtiles \
  -l ocean \
  -Z0 -z12 \
  ne_10m_ocean/ne_10m_ocean.geojson
```

# PlanetilerでMVT作成

次に **Planetiler** でMVTを作成します。

```bash
# JDKのインストール
sudo apt update
sudo apt install openjdk-25-jdk

# 最新版のjarのダウンロード
wget https://github.com/onthegomap/planetiler/releases/latest/download/planetiler.jar
```

YAML設定ファイルを用意します。

```yaml
schema_name: ocean
schema_description: Natural Earth Ocean

sources:
  ocean:
    type: shapefile
    local_path: ne_10m_ocean/ne_10m_ocean.shp

layers:
  - id: ocean
    features:
      - source: ocean
        geometry: polygon
        min_zoom: 0
        max_zoom: 12
```

その上で以下を実行します。

```bash
java -Xmx2g -jar planetiler.jar \
  generate-custom \
  --schema=ocean.yml \
  --output=data/ocean_planetiler.pmtiles
```

# 結果比較

双方での結果は以下のとおりです。

| ツール　| 所要時間 | PMTilesファイルサイズ |
| -- | -- | -- |
| tippecanoe | 74分12秒 | 56MiB |
| Planetiler | 1分19秒 | 33MiB |

ファイルサイズ・所要時間ともにPlanetilerの方が優位な結果となりました。

この結果を考察していきます。

## ファイルサイズの違い

まず、ファイルサイズの違いについて、考察します。

同じ海タイルを生成しているにもかかわらず、ファイルサイズには乖離があります。

z12よりも大きなズームレベルで作成するとその差はさらに顕著になります。

その原因を確認するため、[PMTiles CLI](https://docs.protomaps.com/pmtiles/cli)の `pmtiles show` でメタデータを確認します。

- **ocean_tippecanoe.pmtiles**
```
addressed tiles count: 13970283
tile entries count: 7195936
tile contents count: 376574
```

- **ocean_planetiler.pmtiles**
```
addressed tiles count: 13969174
tile entries count: 463142
tile contents count: 374140
```

`tile contents count`(ユニークなタイルコンテンツの数)はほぼ同じですが、`tile entries count`(ディレクトリエントリ数)がtippecanoeの方がかなり多いです。

## PlanetilerはRLEをフル活用して効率的に圧縮している

PMTilesは、RLE（Run Length Encoding）を採用しており、連続するタイルID間で同じタイルデータを参照する場合は、ディレクトリエントリを一つにまとめる仕組みがあります（詳しくは[Protomaps Blog](https://protomaps.com/blog/pmtiles-v3-hilbert-tile-ids/)を参照）。

Planetilerでは、この仕組みをうまく利用してディレクトリエントリを効率的に格納できていますが、tippecanoeではそれがうまくできていません。

実際にtippecanoeで出力したPMTilesのディレクトリエントリをみると次のようになっています。

```
tile_id=626772, run_length=2, offset=437058, length=108
tile_id=626774, run_length=2, offset=437058, length=108
tile_id=626786, run_length=2, offset=437058, length=108
tile_id=626788, run_length=2, offset=437058, length=108
...（以下同様に2個ずつ）
```

ここの全エントリが `offset=437056, length=108` を参照しているため、本当はすべて一つのディレクトリエントリに結合できるはずですが、それができていません。

tippecanoeでは、PMTilesの書き出しロジックに非効率な部分があることが示唆されます。

## 性能差異

また、処理時間でもtippecanoeに比べてPlanetilerの方がかなり優位な結果になっています。

これは、それぞれのツールで採用されているアルゴリズムの違いに起因するものと思われます。

## tippecanoeのタイル生成アルゴリズム

リポジトリのREADMEには明示されていませんが、tippecanoeのソースを生成AIに解析させると、以下の手順で処理していることがわかります。

- インデックス付け
  - フィーチャごとに代表点からインデックスを計算(デフォルトMorton符号、-ahでHilbert曲線)
- 並べ替え
  - 全フィーチャをインデックス順にソート(--coalesce/--reorder等の指定時は別の比較関数)
- タイル処理(ズームを低い方から高い方へ1段ずつ)
  - 1タイルごとに:
    - 各フィーチャをタイル境界(+バッファ)でクリップ
    - クリップ後、次ズームでも必要なジオメトリを子タイル用の一時ファイルへ転送(rewrite)
    - 残りのフィーチャに対してフィルタ適用・間引きを判定
    - タイル内の全フィーチャをまとめて簡略化(+必要なら結合・並べ替え)
    - サイズ/フィーチャ数上限を超えた場合、原則そのズーム全体を閾値を変えて再処理(--force-feature-limitのみタイル単位)
    - MVTへエンコードしgzip圧縮して書き込み

## Planetilerのタイル生成アルゴリズム

GitHubリポジトリの[ARCHITECTURE.md](https://github.com/onthegomap/planetiler/blob/main/ARCHITECTURE.md)を参照すると、以下の手順で処理されることがわかります。

- 入力ファイル処理
  - 各入力ソースからSourceFeatureを読み込む
  - プロファイルのprocessFeatureを呼び出し、ベクタータイルフィーチャを生成
  - フィーチャごとにすべてのズームレベルのタイルについて以下を処理
    - ジオメトリをズームレベルに合わせて拡大縮小する
    - 画面ピクセル座標で簡略化する
    - タイル境界でジオメトリを分割(180度またぎは複製して対応)
    - ジオメトリをタイル精度(4096 x 4096)に丸め、ポリゴンはトポロジーエラーを修正
    - ソート可能なlongキーを付加してエンコード
    - エンコードされたフィーチャをワークキューに追加
- 並べ替え
  - longキーでフィーチャをソート(チャンクごとにメモリ内ソート→ディスクへ書き戻し)
- ベクタータイル出力
  - k分割マージでソート済みフィーチャを読み出し
  - タイル単位でグルーピング
    - プロファイル指定のグループ上限に応じてフィーチャ間引き(ラベル密度制限)
  - タイルをバッチにまとめてワーカーへ
    - 直前のタイルと内容が同一なら再エンコードをスキップ
    - 出力フォーマット(MVT or MLT)に応じてエンコード
    - 必要に応じてgzip圧縮、重複排除用のコンテンツハッシュ計算
  - ファイル書き込み(MBTilesならプリペアドステートメントでバッチ書き込み)

## タイル生成のアプローチが異なる

**tippecanoe** は、ズームを低い方から高い方へフィーチャを逐次的・再帰的に分割しながら、その都度クリップ・簡略化・間引きを行います。
そのため、同じフィーチャを分割粒度を上げていきながら何度も読み出すことになります。

その一方、**Planetiler** は、全ズームレベルのベクタータイルフィーチャを先に生成して、1回の外部マージソートでタイルID順に整列させ、各タイルフィーチャをソート済みストリームから1回読むだけで完結させます。

その結果、何度も同じフィーチャをロードする必要がないため、特に海ポリゴンのような巨大フィーチャを処理する際に大きな性能上のアドバンテージが生まれるものと考えられます。

# まとめ

Planetilerは、OSMなどのプラネット規模の大容量の地理空間データを高速かつ効率的に処理できるように最適化されて設計されているため、今回検証したようなグローバルスケールの海ポリゴンの処理などでは、特に高い性能を発揮します。

他方、tippecanoeは、性能面ではPlanetilerに劣後する部分があるものの、様々なオプションが豊富に提供されており、細やかなチューニングを容易に実施できる利点があります。

したがって、処理対象とする地理空間データの種類や規模、作成したいベクタータイルの内容に応じて、その用途に適したツールを選定することが重要と思われます。
