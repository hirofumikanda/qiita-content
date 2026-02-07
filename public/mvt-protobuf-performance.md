---
title: 'MVT仕様にみる地理空間データ効率化技術'
tags:
  - GIS
  - vectortile
  - ProtocolBuffers
  - MapboxVectorTile
private: false
updated_at: '2026-01-26T12:00:00+09:00'
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

**Mapbox Vector Tile（MVT）** は、現代のWeb地図アプリケーションにおいて標準的なベクトルタイル形式として広く採用されています。MVTの大きな特徴の一つは、**Protocol Buffers** という効率的なバイナリ形式でデータをエンコードしている点です。

MVTの`.proto`ファイルなどでその詳細な仕様をみることができますが、データ量を低く抑えるための様々な工夫が施されていることがわかります。

本記事では、MVTに適用されているそれらの効率化技術について、具定例を交えつつ検証し、巨大化しがちな地理空間データの効率的管理のヒントを探りたいと思います。

## Protocol Buffersとは

MVTでは、**Protocol Buffers**という、Googleが開発したバイナリ形式のスキーマ言語が使われています。スキーマ言語であるため、技術レイヤーとしては、`JSON`や`XML`と同様です。主な特徴は以下の通りです。

- **バイナリ形式**: テキストベースのJSON/XMLと異なり、バイナリでデータを表現
- **スキーマベース**: `.proto`ファイルでデータ構造を事前定義
- **言語中立**: 多数のプログラミング言語で利用可能
- **高効率**: データサイズが小さく、パース速度が速い

これらの特長から、大量の地図タイルデータをWebでやり取りするのに使われるMVTでは、データ容量及びデコード効率の観点から**Protocol Buffers**が採用されていると考えられます。

Protocol Buffersの詳細については、[公式ドキュメント](https://protobuf.dev/)を参照ください。

## MVTのデータ構造

### MVTのProtobuf定義

MVTのデータ構造は、公式の[vector-tile-spec](https://github.com/mapbox/vector-tile-spec)で確認することができます。

Protocol Buffersのスキーマ定義は`*.proto`ファイルで記述されますが、MVTの最新のprotoファイルは[2.1/vector_tile.proto](https://github.com/mapbox/vector-tile-spec/blob/master/2.1/vector_tile.proto)にあります。

データの具体例を見た方が理解が進むと思いますので、仕様の末尾に記載されている例（MVTをテキスト形式にデコードしたもの）をさらに簡略化したものを以下に抜粋します。

```
layers {
  version: 2
  name: "single_point"
  features {
    id: 1
    tags: 0
    tags: 0
    tags: 1
    tags: 0
    tags: 2
    tags: 1
    type: POINT
    geometry: 9
    geometry: 2410
    geometry: 3080
  }
  keys: "hello"
  keys: "h"
  keys: "count"
  values: {
    string_value: "world"
  }
  values: {
    double_value: 1.23
  }
  extent: 4096
}
```

ここで定義されているのはポイントデータ1点のみです。

`features`に各々のフィーチャのプロパティと形状が定義されています。
`tags`フィールドがプロパティで、`geometry`フィールドが形状を表します。

初見では、よくわからないと思いますが、一度理解してしまえば余り難しくはありません。

各値の意味については、のちほど詳しく取り上げます。

## MVTのデータ効率化技術のポイント

上の例でみても、フィーチャの定義値のほとんどが正の整数で表現されており、
データ量として低く抑えられていることがわかるかと思います。

MVTのProtobuf定義には、データサイズを削減するための工夫が複数含まれています。
代表的なものは以下のとおりです。

1. **キー・バリューの辞書化**: 重複する属性名と値を辞書として管理
2. **ジオメトリのコマンドエンコーディング**: 描画コマンドで形状を表現
3. **ジオメトリの差分エンコーディング**: 座標を差分値で保存
4. **座標差分値のZigZagエンコーディング**: 座標差分値にZigZagエンコーディングを利用

これらについて順番に見ていきたいと思います。

### 1. キー・バリューの辞書化

GeoJSONなどのフォーマットでは、各フィーチャに属性名と値の両方を保持します。

```json
{
  "properties": {
    "name": "渋谷駅",
    "type": "station"
  }
},
{
  "properties": {
    "name": "新宿駅",
    "type": "station"
  }
}
```

一方、MVTでは、レイヤーレベルで辞書を作成し、フィーチャではインデックスのみを保持します。

```
features: {
  tags: 0
  tags: 0
  tags: 1
  tags: 2
}
features: {
  tags: 0
  tags: 1
  tags: 1
  tags: 2
}
keys: "name"
keys: "type"
values: {
  string_value: "渋谷駅"
}
values: {
  string_value: "新宿駅"
}
values: {
  string_value: "station"
}
```

ここでは、属性に関するフィールド以外は全て省略しています。

MVTでは、`tags`フィールドで各フィーチャが持つ属性が表現されます。
値は辞書のインデックスの値となっており、上から順番に

- 1つ目の属性のkeyのインデックス値
- 1つ目の属性のvalueのインデックス値
- 2つ目の属性のkeyのインデックス値
- 2つ目の属性のvalueのインデックス値

・・・という順番で格納されます。

上の例の場合、次のようになります。インデックスは0始まりであることに注意してください。

```
- 渋谷駅
  - 1つ目の属性
    - key:"name" → keysの1つ目 → インデックス:0
    - value:"渋谷駅" → valuesの1つ目 → インデックス:0
  - 2つ目の属性
    - key:"type" → keysの2つ目 → インデックス:1
    - value:"station" → valuesの3つ目 → インデックス:2

- 新宿駅
  - 1つ目の属性
    - key:"name" → keysの1つ目 → インデックス:0
    - value:"新宿駅" → valuesの2つ目 → インデックス:1
  - 2つ目の属性
    - key:"type" → keysの2つ目 → インデックス:1
    - value:"station" → valuesの3つ目 → インデックス:2
```

この方式により、同じ属性名や値の重複を排除し属性に関するデータ量を大幅に削減しています。

### 2. コマンドエンコーディング

MVTのジオメトリは、描画コマンドとして表現されます。
描画コマンドには以下の3つがあります。

| コマンド | ID | 意味 |
|---------|----|----|
| MoveTo | 1 | ジオメトリの始点にペンを移動 |
| LineTo | 2 | 線を描く（LineString、Polygonのみ） |
| ClosePath | 7 | パスを閉じる（Polygonのみ） |

コマンドIDと繰り返し回数を1つの整数にエンコードし、geometryフィールドの最初の値（CommandInteger）として指定します。

CommandIntegerは最大32ビットの整数で、最初の3ビットが描画コマンドで、残りの29ビットがそのコマンドの繰り返し回数を表します。
したがって、ビット演算は以下のようになります。

```
CommandInteger = (id & 0x7) | (count << 3)
```

例えば、次のように計算されます。

- `MoveTo` 1回 = `(1 & 0x7) | (1 << 3)` = 9
- `LineTo` 2回 = `(2 & 0x7) | (2 << 3)` = 18

### 3. ジオメトリの差分エンコーディング

GeoJSONなどでは、絶対座標が使用されます。

```json
{
  "type": "LineString",
  "coordinates": [
    [139.7000, 35.6800],
    [139.7100, 35.6850],
    [139.7200, 35.6900]
  ]
}
```

他方、MVTでは座標の差分値を使用します。

`extent`で定義された値（通常4,096）で正規化された座標を用いて、それに基づいた移動量でパラメータ値（ParameterInteger）を定義します。

移動量は必ずXYのペアで定義します。

そのため、1回の`MoveTo`コマンド及び`LineTo`コマンドに対して2個のParamterIntegerが与えられます。

各フィーチャのgeometryフィールドの値は、CommandIntegerの後にそのコマンドのParameterIntegerを列挙します。

例えば、以下のようになります（ジオメトリの形状に関するフィールド以外は省略）。

```
features: {
  type: LINESTRING
  geometry: 9     // CommandInteger（MoveToを1回）
  geometry: 4096  // ParameterInteger（MoveToのパラメータX）
  geometry: 4096  // ParameterInteger（MoveToのパラメータY）
  geometry: 18    // CommandInteger（LineToを2回）
  geometry: 200   // ParameterInteger（1個目のLineToのパラメータX）
  geometry: 100   // ParameterInteger（1個目のLineToのパラメータY）
  geometry: 200   // ParameterInteger（2個目のLineToのパラメータX）
  geometry: 100   // ParameterInteger（2個目のLineToのパラメータY）
}
extent: 4096
```

これは、以下のコマンドを順に実行してラインを描画することを意味します。

```
MoveTo(2048, 2048)      // 開始点への移動
LineTo(+100, +50)       // この座標差分だけ線分を描画
LineTo(+100, +50)       // この座標差分だけ線分を描画
```

このようにコマンドの種別と操作回数を最初に明確化すると同時に正規化座標の差分値でパラメータを指定することで、データ量を抑制しつつ、効率的に描画操作を定義しています。

なお、ParameterIntegerと実際の座標の差分値が異なるのは、次のZigZagエンコーディングが使われているためです。

### 4. ZigZagエンコーディング

MVTでは、負の値を含む座標の差分値を効率的に保存するため、**ZigZagエンコーディング**を使用しています。

座標の差分値（value）からParameterIntegerへの変換式は以下で与えられます。

```
ParameterInteger = (value << 1) ^ (value >> 31)
```

変換例：
- 0 → 0
- -1 → 1
- 1 → 2
- -2 → 3
- 2 → 4

これにより、正負を問わず小さな値を小さいバイト数で表現しています。

## GeoJSONとのパフォーマンス比較

MVTのパフォーマンスの高さを検証するため、GeoJSONとデータサイズ及びデコード速度を比較してみたいと思います。

比較対象のデータとして、[MapLibreデモタイル](https://github.com/maplibre/demotiles)の日本の本州付近のタイル（5/28/12）を使います。

- https://demotiles.maplibre.org/tiles/5/28/12.pbf

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/060269dd-97d4-4c13-b73c-9342a98c01b4.png" width="400px">

### MVTの中身をみる

検証に入る前にデータの中身がどうなっているか確認します。

`protocolbuf`をテキスト形式に変換するため、`protoc`を使います。

`protoc`はGoogleが開発したProtocol Buffersのコンパイラーです。
公式の[インストール手順](https://protobuf.dev/installation/)に従いインストールします。

例えば、WSLのUbuntuでは、以下のコマンドでインストールできます。

```bash
sudo apt install protobuf-compiler
```

これを使ってデモタイルをテキスト形式にデコードします。
この際、MVTのスキーマ定義ファイル（[vector_tile.proto](https://github.com/mapbox/vector-tile-spec/blob/master/2.1/vector_tile.proto)）が必要になることに留意します。

デモタイルとスキーマ定義ファイルをダウンロードして、以下のコマンドでテキストに変換します。

```bash
cat 12.pbf | protoc -I . vector_tile.proto --decode=vector_tile.Tile
```
```
layers {
  name: "centroids"
  features {
    id: 116
    tags: 0
    tags: 0
    tags: 1
    tags: 0
    type: POINT
    geometry: 9
    geometry: 3210
    geometry: 3840
  }
  keys: "NAME"
  keys: "ABBREV"
  values {
    string_value: "Japan"
  }
  extent: 4096
  version: 2
}
layers {
  name: "countries"

＝＝＝＝＝＝＝長いので割愛
```

上でみてきたとおりの仕様で、ポイントとポリゴンデータが異なるレイヤーで入っていることがわかります。

### MVTをGeoJSONに変換

検証の準備のため、ダウンロードしたMVTをGeoJSONに変換します。

[@mapbox/vector-tile](https://github.com/mapbox/vector-tile-js)を使えば、簡単にMVTをパースして、GeoJSONに変換することができます。

手前みそですが、以前変換ツールを作ったことがあるので、それを使います。
https://hirofumikanda.github.io/mvt-viewer/

ファイル名を`5-28-12.pbf`に変更してアップロードすると、自動的にGeoJSONがダウンロードされます。

```
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {
        "type": "Point",
        "coordinates": [
          139.40826416015625,
          36.87962060502676
        ]
      },
      "properties": {
        "NAME": "Japan",
        "ABBREV": "Japan"
      },
      "id": 116
    },
    {
      "type": "Feature",
      "geometry": {
        "type": "MultiPolygon",

＝＝＝＝＝＝＝長いので割愛
```

### ファイルサイズ

準備が整ったので、検証に入ります。

まず、ファイルサイズを比較します。

| ファイル | 容量 |
|---------|----|
| 5-28-12.geojson | 197 KB |
| 5-28-12.pbf | 4.1 KB |

約50倍異なり、かなりの差異があることがわかります。

### デコード速度

次に、デコード速度を比較します。

Node.jsで以下の関数の実行時間を計測します（ファイルI/Oは含みません）。

```javascript
function decodeMvt(buf) {
  return new VectorTile(new Protobuf(buf));
}

function decodeGeojson(text) {
  return JSON.parse(text);
}
```

500回ずつ実行し、その平均値を算出します。

結果は以下のとおりです。

| ファイル | デコード時間 |
|---------|----|
| 5-28-12.geojson | 0.6678 ms |
| 5-28-12.pbf | 0.0365 ms |

こちらもデータ容量と同様に約20倍と大きな差が出ました。

このようにMVTはGeoJSONと比較すると、データ容量及びデコード速度の観点で圧倒的に優位です。

MVTがいかに効率よくデータを保持しているかがわかると思います。

## まとめ

Mapbox Vector Tileは、Protocol Buffers形式の採用及び様々なデータ効率化技術の実装により、
Web地図アプリケーションのパフォーマンスに大きく貢献しています。

具体的には、以下の工夫が施されています。

- **属性のキー/バリューの辞書化による重複データ排除**
- **ジオメトリの（コマンド|差分|ZigZag）エンコーディングによるデータ量削減**

MVTのProtocol Buffers形式は、地図タイル配信に特化して設計された優れたフォーマットといえます。

その効率化の仕組みを理解しておくことで、さまざまな場面で、適切な地理空間データの作成や効率的な管理に役立てることができると思います。
