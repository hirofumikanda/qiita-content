---
title: 'GIS入門: 座標参照系、測地系、投影法の違いを理解する'
tags:
  - GIS
  - 座標参照系
  - CRS
  - 測地系
  - 投影法
private: false
updated_at: '2025-12-01T00:00:00+09:00'
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに

GIS(地理情報システム)を学び始めると、「座標参照系」「測地系」「地球楕円体」「投影法」「座標系」など、似たような用語が数多く登場して混乱することがあります。

本記事では、これらの概念の違いと関係性を、GIS入門者向けに段階的に解説します。

# 地球の形状を表現する

GISで建物や道路などの地理空間的事物（地物）を表現するには位置情報が必要になります。
そして、位置情報を与えるには、そのための基準が必要です。

地球上の現象を所定の縮尺に従って正確に表現するために、位置の基準にも正確性が求められます。

位置の基準は**測地系**と呼ばれ、様々な測地系が提案されています。
測地系を定めるには、まず地球の形状を模した**地球楕円体**を定義する必要があります。

## 1. 地球楕円体(回転楕円体)

地球は完全な球体ではなく、赤道方向にわずかに膨らんだ**回転楕円体**に近い形をしています。

楕円体の形は以下の値で定義されます。

- 長半径(赤道半径): a
- 短半径(極半径): b
- 扁平率 = (a - b) / a

### 主要な地球楕円体

| 楕円体名 | 長半径(a) | 短半径(b) | 扁平率 | 主な用途 |
|---------|----------|----------|-------|------|
| **WGS84** | 6,378,137 m | 6,356,752 m | 1/298.257223563 | GPS |
| **GRS80** | 6,378,137 m | 6,356,752 m | 1/298.257222101 | 日本測地系2011 |
| **ベッセル** | 6,377,397 m | 6,356,079 m | 1/299.152813 | 旧日本測地系 |

- WGS84とGRS80は数値がほぼ同じですが、厳密には扁平率が異なります(ただし実用上の差はごくわずかです)。
- 余談ですが、国際海里の1海里=1,852メートルは、ベッセル楕円体の緯度1分の距離（極から赤道の距離の1/90x60）に由来するそうです。

## 2. 測地系(測地基準系)

**測地系(Geodetic Datum)** は、地球楕円体を実際の地球にどう位置合わせするかを定めたものです。

### 測地系の構成要素

1. **地球楕円体**: どの楕円体を使うか
2. **原点**: 楕円体の中心位置
3. **方向**: 楕円体の軸の向き

### 日本における測地系の変遷

```
旧日本測地系(Tokyo Datum)
  ├─ 地球楕円体: ベッセル楕円体
  ├─ 原点: 東京都港区麻布台(旧東京天文台)
  └─ 時期: 明治時代～2002年3月

世界測地系(JGD2000)
  ├─ 地球楕円体: GRS80
  ├─ 原点: 地球の重心
  └─ 時期: 2002年4月～2011年

世界測地系(JGD2011)
  ├─ 地球楕円体: GRS80
  ├─ 原点: 地球の重心
  ├─ 時期: 2011年10月～(現在)
  └─ 注: 東日本大震災の地殻変動を反映
```

**測地系が変わると座標値が変わります。**
- 旧日本測地系と世界測地系では、同じ地点でも経度・緯度が約400m程度ずれます。詳細は[国土地理院のサイト](https://www.gsi.go.jp/LAW/G2000-g2000-h3.htm)を参照ください。
- JGD2000とJGD2011でも、東北地方では最大数mのずれがあります。

# 投影法: 3次元を2次元に変換する

地球は3次元の立体ですが、地図は2次元の平面です。3次元の地球を2次元の平面に変換する方法を**投影法(Map Projection)** といいます。

## 投影の種類

### 1. 正角図法(角度を保つ)

- 角度が正確に保たれるため、方位の測定や航海図に適しています。
- 参考：[QGIS Documentation: Map projections with angular conformity](https://docs.qgis.org/3.40/en/docs/gentle_gis_introduction/coordinate_reference_systems.html#map-projections-with-angular-conformity)

### 2. 正距図法(距離を保つ)

- 特定の点からの距離が正確に保たれます。
- 参考：[QGIS Documentation: Map projections with equal distance](https://docs.qgis.org/3.40/en/docs/gentle_gis_introduction/coordinate_reference_systems.html#map-projections-with-equal-distance)

### 3. 正積図法(面積を保つ)

- 面積が正確に保たれるため、統計地図や分布図に適しています。
- 参考：[QGIS Documentation: Projections with equal areas](https://docs.qgis.org/3.40/en/docs/gentle_gis_introduction/coordinate_reference_systems.html#projections-with-equal-areas)

## 投影による歪み

**すべての投影法には必ず歪みがあります。** 3次元を2次元に変換する際、角度・面積・距離のすべてを同時に正確に保つことは数学的に不可能です。

```
投影法の選択基準:
┌─────────────────────┐
│ 何を優先するか？      │
└─────────────────────┘
        ↓
  ┌─────┴───────┐
  │             │
 角度          面積
  │             │
正角図法      正積図法
(メルカトル)  (モルワイデ)
```

# 座標系: 位置を数値で表現する

## 1. 地理座標系(Geographic Coordinate System)

地球楕円体面上の位置を**経度(Longitude)** と**緯度(Latitude)** で表現します。
経度・緯度の定義は国土地理院の[日本での位置及び高さの基準となる測地系](https://www.gsi.go.jp/sokuchikijun/datum-main.html#p1)を参照ください。

```
        北極(90°N)
          ●
    西経  |  東経
    ←─────●────→
        赤道(0°)
          |
          ●
        南極(90°S)

経度: -180° ~ +180° (または 0° ~ 360°)
緯度: -90° ~ +90°

単位: 度(°)
```

## 2. 投影座標系(Projected Coordinate System)

投影法を使って2次元平面上の位置を**X座標**と**Y座標**で表現します。**X座標とY座標の取り方は座標系によって異なります(下記は一例です)。**

```
Y軸(北・中心子午線)
  ↑
  |    ● (X=50000, Y=30000)
  |   
  |  
  └──────────→ X軸(東)
 原点(0, 0)

単位: メートル
```

### 日本の平面直角座標系

日本の国土を19の座標系に分割した座標系です。それぞれを横メルカトル図法で投影していますが、原点の位置が異なっており、都道府県・地域ごとに使用すべき座標系が決まっています。詳細は国土地理院の[わかりやすい平面直角座標系](https://www.gsi.go.jp/sokuchikijun/jpc.html)を参照ください。

### UTM図法(Universal Transverse Mercator)

世界を経度6度ごとに分割し、1-60のゾーンを割り当てます。こちらも平面直角座標系と同様、横メルカトル図法で投影した座標系ですが、原点や座標軸の定義、縮尺係数が異なります。詳細は[wikipedia](https://ja.wikipedia.org/wiki/%E3%83%A6%E3%83%8B%E3%83%90%E3%83%BC%E3%82%B5%E3%83%AB%E6%A8%AA%E3%83%A1%E3%83%AB%E3%82%AB%E3%83%88%E3%83%AB%E5%9B%B3%E6%B3%95)を参照ください。

```
UTMゾーン(日本周辺)
経度  ゾーン番号  対象地域
────────────────────────
120°E～126°E  51  沖縄西部
126°E～132°E  52  沖縄、九州
132°E～138°E  53  中国、四国、近畿、中部
138°E～144°E  54  関東、北陸、東北、北海道
144°E～150°E  55  北海道東部
150°E～156°E  56  南鳥島
```

# 座標参照系(CRS): すべてを統合する概念

**座標参照系(Coordinate Reference System, CRS)** は、位置を一意に特定するために必要なすべての情報をまとめたものです。

## CRSの階層構造

```
座標参照系(CRS)
├─ 測地系(Datum)
│   ├─ 地球楕円体(Ellipsoid)
│   │   ├─ 長半径
│   │   └─ 扁平率
│   └─ 原点・方向
│
└─ 座標系(Coordinate System)
    ├─ 地理座標系
    │   └─ 単位: 度
    └─ 投影座標系
        ├─ 投影法
        └─ 単位: メートル

```

## CRSの種類

### 1. 地理座標系CRS

測地系 + 地理座標系

```
例: WGS84地理座標系
├─ 測地系: WGS84
│   └─ 地球楕円体: WGS84楕円体
└─ 座標系: 地理座標系(経度・緯度)
    └─ 単位: 度
```

### 2. 投影座標系CRS

測地系 + 投影法 + 投影座標系

```
例: UTM Zone 54N (WGS84)
├─ 測地系: WGS84
│   └─ 地球楕円体: WGS84楕円体
├─ 投影法: UTM図法
│   └─ ゾーン: 54
└─ 座標系: 投影座標系(X・Y)
    └─ 単位: メートル
```

# EPSGコード: CRSの識別番号

**EPSGコード**は、EPSG(European Petroleum Survey Group)が定めたCRSの識別番号です。2005年にEPSGが**IOGP(International Association of Oil & Gas Producers)** に統合されたため、現在はIOGPのGeomatics Committeeが管理しています。

## よく使うEPSGコード

### 世界の地理座標系

| EPSGコード | 名称 | 説明 |
|-----------|------|------|
| **4326** | WGS84 | GPS |

### 世界の投影座標系

| EPSGコード | 名称 | 説明 |
|-----------|------|------|
| **3857** | Web Mercator | Google Maps、Webマップのデファクトスタンダード |

### 日本の地理座標系

| EPSGコード | 名称 | 説明 |
|-----------|------|------|
| **4612** | JGD2000 | 日本測地系2000(地理座標系) |
| **6668** | JGD2011 | 日本測地系2011(地理座標系) |

### 日本の投影座標系

| EPSGコード | 名称 | 説明 |
|-----------|------|------|
| **2443～2461** | 平面直角座標系(JGD2000) | 1～19系 |
| **3097～3101** | JGD2000 / UTM | 51～55ゾーン |
| **6669～6687** | 平面直角座標系(JGD2011) | 1～19系 |
| **6688～6692** | JGD2011 / UTM | 51～55ゾーン |

## EPSGコードの確認方法

### Webサイト

- [EPSG.io](https://epsg.io/) - CRSの詳細情報を検索
- [Spatial Reference](https://spatialreference.org/) - CRS定義を様々な形式で取得

### コマンドライン(GDAL)

```bash
# EPSGコードからCRS情報を取得
$ gdalsrsinfo EPSG:4326

PROJ.4 : +proj=longlat +datum=WGS84 +no_defs

OGC WKT2:2019 :
GEOGCRS["WGS 84",
    ENSEMBLE["World Geodetic System 1984 ensemble",
        MEMBER["World Geodetic System 1984 (Transit)"],
        MEMBER["World Geodetic System 1984 (G730)"],
        MEMBER["World Geodetic System 1984 (G873)"],
        MEMBER["World Geodetic System 1984 (G1150)"],
        MEMBER["World Geodetic System 1984 (G1674)"],
        MEMBER["World Geodetic System 1984 (G1762)"],
        MEMBER["World Geodetic System 1984 (G2139)"],
        ELLIPSOID["WGS 84",6378137,298.257223563,
            LENGTHUNIT["metre",1]],
        ENSEMBLEACCURACY[2.0]],
    PRIMEM["Greenwich",0,
        ANGLEUNIT["degree",0.0174532925199433]],
    CS[ellipsoidal,2],
        AXIS["geodetic latitude (Lat)",north,
            ORDER[1],
            ANGLEUNIT["degree",0.0174532925199433]],
        AXIS["geodetic longitude (Lon)",east,
            ORDER[2],
            ANGLEUNIT["degree",0.0174532925199433]],
    USAGE[
        SCOPE["Horizontal component of 3D system."],
        AREA["World."],
        BBOX[-90,-180,90,180]],
    ID["EPSG",4326]]
```

```bash
# ファイルのCRSを確認
gdalinfo file.tif
ogrinfo -al -so file.shp
```

GDALは[GDAL Downloadページ](https://gdal.org/en/stable/download.html)から入手できます。

また、Windowsユーザの場合は、[OSGeo4W](https://trac.osgeo.org/osgeo4w/)を使ってインストールしてもよいと思います。エクスプレスインストールでGDALをチェックすれば自動的に必要なパッケージをインストールしてくれます。

# まとめ

最後に、用語を整理しておきます。

| 用語 | 意味 | 例 |
|-----|------|---|
| **地球楕円体** | 地球の形状を近似する数学モデル | WGS84楕円体、GRS80楕円体 |
| **測地系** | 楕円体を実際の地球に位置合わせしたもの | WGS84、JGD2011 |
| **投影法** | 3次元を2次元に変換する方法 | メルカトル、UTM |
| **地理座標系** | 経度・緯度で位置を表現 | 単位は度 |
| **投影座標系** | X・Y座標で位置を表現 | 単位はメートルなど |
| **座標参照系(CRS)** | 測地系＋座標系（＋投影法） | EPSG:4326 |
| **EPSGコード** | CRSを識別する番号 | 4326、3857 |
