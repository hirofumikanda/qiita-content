---
title: 'FlatGeobuf: Cloud-Optimized な地理空間データフォーマットとは'
tags:
  - GIS
  - geojson
  - 地理空間情報
  - flatgeobuf
  - CloudOptimized
private: false
updated_at: '2025-11-29T16:46:41+09:00'
id: 11fa6e0fe72546f9fbb1
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに

地理空間データの世界では、GeoJSONやShapefileなどのフォーマットが広く使われてきました。しかし、これらのフォーマットには大規模データの扱いやクラウド環境での利用において課題があります。

本記事では、これらの課題を解決する**FlatGeobuf**というフォーマットと、その基盤となる**Cloud-Optimized**の概念について詳しく解説します。

# Cloud-Optimizedとは何か

## 従来のデータフォーマットの課題

従来の地理空間データフォーマット（GeoJSON、Shapefileなど）は、データ全体を読み込まないと一部のデータにアクセスできない構造になっています。

```
┌─────────────────────────────────┐
│  GeoJSON ファイル (100MB)        │
│  ・全データをダウンロード必要   │
│  ・メモリに全展開が必要         │
│  ・部分的な読み取り不可         │
└─────────────────────────────────┘
         ↓ ダウンロード
    100MB すべて取得
```

これには以下の問題があります。

1. **全データのダウンロードが必要** - 一部のデータだけ欲しくても、ファイル全体をダウンロードする必要がある
2. **メモリ消費が大きい** - 大規模データを扱う際にメモリ不足になりやすい
3. **初期表示が遅い** - データの読み込みが完了するまで表示できない

## Cloud-Optimizedの概念

**Cloud-Optimized（クラウド最適化）**とは、HTTPの**Range Request**機能を活用して、ファイルの必要な部分だけを効率的に取得できるようにしたデータフォーマットの設計思想です。一般的な構造は以下のとおりです。

```
┌─────────────────────────────────┐
│  Cloud-Optimized ファイル        │
│  ┌──────┬──────┬──────┬──────┐ │
│  │Header│Index │Data  │Data  │ │
│  │ (4KB)│(100KB)│Chunk1│Chunk2│ │
│  └──────┴──────┴──────┴──────┘ │
└─────────────────────────────────┘
         ↓ Range Request
    必要な部分のみ取得 (例: ヘッダー+インデックス(104KB) → 必要なデータのみを随時)
```

### Cloud-Optimizedの特徴

1. **ヘッダーとインデックスを先頭に配置**
   - ファイルの最初にメタデータとインデックスを格納
   - 小さなRange Requestでファイル構造を把握できる

2. **空間インデックスの内包**
   - 空間検索を高速化するインデックス（R-tree、Hilbert曲線など）を含む
   - 必要なデータの位置を素早く特定できる

3. **Range Requestによる部分取得**
   - HTTPのRange Requestヘッダーを使用
   - 必要なバイト範囲のみをダウンロード

### Range Requestの仕組み

HTTPのRange Requestは、以下のようにリクエストヘッダーで取得範囲を指定します。

```http
GET /data.fgb HTTP/1.1
Host: example.com
Range: bytes=0-8191
```

サーバーは指定された範囲のデータのみを返します。

```http
HTTP/1.1 206 Partial Content
Content-Range: bytes 0-8191/10485760
Content-Length: 8192
```

## Cloud-Optimizedフォーマットの例

Cloud-Optimizedの概念を採用したフォーマットには以下があります。

- **Cloud-Optimized GeoTIFF (COG)** - ラスターデータ
- **FlatGeobuf** - ベクターデータ
- **Cloud-Optimized Point Cloud (COPC)** - 点群データ
- **PMTiles** - タイルデータ

# FlatGeobufとは

**FlatGeobuf**は、Cloud-Optimized設計のベクター地理空間データフォーマットです。Björn Harrtell氏によって開発され、2019年にリリースされました。

## FlatGeobufの特徴

### 1. コンパクトなバイナリ形式

[FlatGeobuf](https://flatgeobuf.org/)は、Google社の[FlatBuffers](https://flatbuffers.dev/)ライブラリを使用したバイナリフォーマットです。

```javascript
// GeoJSON (テキスト形式)
{
  "type": "Feature",
  "geometry": {
    "type": "Point",
    "coordinates": [139.7671, 35.6812]
  },
  "properties": {
    "name": "Tokyo"
  }
}

// FlatGeobuf (バイナリ形式)
// 同じデータがより少ないバイト数で表現される
```

### 2. 空間インデックス内蔵

FlatGeobufは[Packed Hilbert R-tree](https://guide.cloudnativegeo.org/flatgeobuf/hilbert-r-tree.html)という空間インデックスを内蔵しています。

```
FlatGeobuf ファイル構造:
┌──────────────────────────────────┐
│ マジックバイト (8 bytes)          │
├──────────────────────────────────┤
│ ヘッダー (可変長)                │
│  - ジオメトリタイプ              │
│  - 属性スキーマ                  │
│  - CRS情報                       │
│  - メタデータなど                 │
├──────────────────────────────────┤
│ 空間インデックス (オプション)     │
│  - Packed Hilbert R-tree         │
│  - 各ノードの境界ボックス        │
│  - データへのオフセット          │
├──────────────────────────────────┤
│ フィーチャーデータ               │
│  ┌────────────────────┐          │
│  │ Feature 1          │          │
│  ├────────────────────┤          │
│  │ Feature 2          │          │
│  ├────────────────────┤          │
│  │ Feature 3          │          │
│  └────────────────────┘          │
└──────────────────────────────────┘
```

### 3. 多言語対応

主要なプログラミング言語でライブラリが提供されています。

- JavaScript/TypeScript
- Python
- Java
- Rust
- Go

## FlatGeobuf vs 他フォーマット

| フォーマット | ファイルサイズ | 読み込み速度 | 空間検索 | Cloud-Optimized |
|------------|--------------|------------|---------|----------------|
| GeoJSON | ★☆☆ (大) | ★★☆ | ★☆☆ | ★☆☆ |
| Shapefile | ★★☆ (中) | ★★★ | ★☆☆ | ★☆☆ |
| GeoPackage | ★★★ (小) | ★★★ | ★★★ | ★★☆ |
| FlatGeobuf | ★★★ (小) | ★★★ | ★★★ | ★★★ |

# FlatGeobufの使い方

## 1. データの変換

### GDAL/OGRを使用する場合

```bash
# GeoJSONからFlatGeobufへ変換
ogr2ogr -f FlatGeobuf output.fgb input.geojson

# 空間インデックスを作成して変換(デフォルトはYESなのでつけなくても同じ)
ogr2ogr -f FlatGeobuf -lco SPATIAL_INDEX=YES output.fgb input.geojson

```

## 2. Webアプリケーションでの利用

### JavaScript/TypeScriptでの読み込み

```javascript
import { deserialize } from 'flatgeobuf/lib/mjs/geojson';

// HTTPからストリーミング読み込み
async function loadFlatGeobuf(url) {
  const response = await fetch(url);
  
  // すべてのフィーチャーを取得
  const features = [];
  for await (const feature of deserialize(response.body)) {
    features.push(feature);
  }
  
  return {
    type: 'FeatureCollection',
    features: features
  };
}

// 使用例
const geojson = await loadFlatGeobuf('https://example.com/data.fgb');
```

### 境界ボックスでのフィルタリング

```javascript
import { deserialize } from 'flatgeobuf/lib/mjs/geojson';

// 特定の範囲のデータのみを取得
async function loadFlatGeobufBbox(url, bbox) {
  const response = await fetch(url);
  
  // bbox: [minX, minY, maxX, maxY]
  const features = [];
  for await (const feature of deserialize(response.body, bbox)) {
    features.push(feature);
  }
  
  return {
    type: 'FeatureCollection',
    features: features
  };
}

// 東京周辺のデータのみ取得
const bbox = [139.5, 35.5, 139.9, 35.8];
const tokyoData = await loadFlatGeobufBbox(
  'https://example.com/japan.fgb',
  bbox
);
```

### Leafletでの使用例

```html
<!DOCTYPE html>
<html>
<head>
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
  <script src="https://unpkg.com/flatgeobuf/dist/flatgeobuf-geojson.min.js"></script>
  <style>
    #map { height: 600px; }
  </style>
</head>
<body>
  <div id="map"></div>
  <script>
    const map = L.map('map').setView([35.6812, 139.7671], 10);
    
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '© OpenStreetMap contributors'
    }).addTo(map);
    
    // FlatGeobufデータを読み込み
    async function loadData() {
      const response = await fetch('https://example.com/data.fgb');
      
      // 現在の表示範囲を取得
      const bounds = map.getBounds();
      const bbox = [
        bounds.getWest(),
        bounds.getSouth(),
        bounds.getEast(),
        bounds.getNorth()
      ];
      
      // 表示範囲内のデータのみ取得
      const iter = flatgeobuf.deserialize(response.body, bbox);
      
      for await (const feature of iter) {
        L.geoJSON(feature).addTo(map);
      }
    }
    
    loadData();
    
    // 地図移動時にデータを再読み込み
    map.on('moveend', loadData);
  </script>
</body>
</html>
```

# まとめ

## FlatGeobufの利点

✅ **ファイルサイズの削減** - GeoJSONの50-70%程度に圧縮
✅ **高速な読み込み** - ストリーミングと空間インデックスにより高速化
✅ **Cloud-Optimized** - 必要な部分のみをHTTPで取得可能
✅ **メモリ効率** - 全データをメモリに展開する必要がない
✅ **空間検索の高速化** - 内蔵された空間インデックスを活用
✅ **エコシステムの充実** - 主要GISツール・ライブラリで対応

## 使用を検討すべきケース

- 大規模ベクターデータ（数万件以上）の配信
- 範囲指定でのデータ取得が頻繁に発生するケース
- サーバーレスアーキテクチャでの静的配信

## 注意点

- ファイルの書き込み・更新には向かない（読み取り専用での使用を推奨）
- 小規模データ（数百件以下）では効果が限定的
- シームレスなウェブ地図配信はタイルの方がパフォーマンス優位

# 参考リンク

- [FlatGeobuf公式サイト](https://flatgeobuf.org/)
- [FlatGeobuf GitHub](https://github.com/flatgeobuf/flatgeobuf)
- [Cloud Optimized GeoTIFF](https://www.cogeo.org/)
- [GDAL FlatGeobuf driver](https://gdal.org/drivers/vector/flatgeobuf.html)
- [FlatBuffers](https://google.github.io/flatbuffers/)

---

FlatGeobufをはじめとするCloud-Optimizedのフォーマットを活用することで、より効率的な地理空間データの配信・処理が可能になります。ぜひプロジェクトで活用してみてください！
