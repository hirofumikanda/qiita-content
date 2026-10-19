---
title: MapLibre GL JSのglobal stateを使った国名ラベルの言語切り替え
tags:
  - MapLibre
  - MVT
private: false
updated_at: '2026-10-19T19:45:23+09:00'
id: c1cfc45d171d3e551afd
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
# はじめに

先週リリースされた**MapLibre GL JS v6.14.0**に、`globalState`のデフォルト設定値をMap初期化時に上書きする機能が追加されました。

globalStateは、同じStyleJSONを使いつつ、フィーチャ横断的（グローバル）に地図表示を変更する機能です。

本記事では、Natural EarthのCountriesのデータを用いて、globalStateの機能を使った国名ラベルの言語切り替えをデモしてみたいと思います。

# Natural EarthのCountriesデータをダウンロード

データは以下からダウンロードできます。

https://www.naturalearthdata.com/downloads/10m-cultural-vectors/10m-admin-0-countries/

ダウンロードできたら、解凍して属性を確認します。

```bash
unzip ne_10m_admin_0_countries.zip -d data

# プロパティ確認
ogrinfo -so -al data/ne_10m_admin_0_countries.shp
```

# GeoJSONに変換

tippecanoeを使ってMVTに変換するため、まずは、[ogr2ogr](https://gdal.org/en/stable/programs/ogr2ogr.html)使ってシェープファイルをGeoJSONに変換します。

国名は**NAME_EN**などの属性名で入っているのでそれらのみ抽出します。

```bash
ogr2ogr \
  -f GeoJSON \
  data/countries.geojson \
  data/ne_10m_admin_0_countries.shp \
  -select "ADMIN, \
  NAME_AR, \
  NAME_BN, \
  NAME_DE, \
  NAME_EN, \
  NAME_ES, \
  NAME_FA, \
  NAME_FR, \
  NAME_EL, \
  NAME_HE, \
  NAME_HI, \
  NAME_HU, \
  NAME_ID, \
  NAME_IT, \
  NAME_JA, \
  NAME_KO, \
  NAME_NL, \
  NAME_PL, \
  NAME_PT, \
  NAME_RU, \
  NAME_SV, \
  NAME_TR, \
  NAME_UK, \
  NAME_UR, \
  NAME_VI, \
  NAME_ZH, \
  NAME_ZHT"
```

# 国名ラベル用のGeoJSONを作成

国名ラベル用のポイントデータをGeoJSONで作成するため、[duckdb](https://duckdb.org/)を使います。

事前にSQLを作成します。

`ST_PointOnSurface`関数でポリゴン内部の代表点ポイントを生成できます。

```sql
INSTALL spatial;
LOAD spatial;

-- Countriesのポリゴンを読み込む
CREATE OR REPLACE TABLE countries AS
SELECT
    *
FROM ST_Read('data/countries.geojson');

-- 国名ラベル用のポイントを生成
CREATE OR REPLACE TABLE country_labels AS
SELECT
    ADMIN,
    NAME_AR,
    NAME_BN,
    NAME_DE,
    NAME_EN,
    NAME_ES,
    NAME_FA,
    NAME_FR,
    NAME_EL,
    NAME_HE,
    NAME_HI,
    NAME_HU,
    NAME_ID,
    NAME_IT,
    NAME_JA,
    NAME_KO,
    NAME_NL,
    NAME_PL,
    NAME_PT,
    NAME_RU,
    NAME_SV,
    NAME_TR,
    NAME_UK,
    NAME_UR,
    NAME_VI,
    NAME_ZH,
    NAME_ZHT,

    -- ポリゴン内部の代表点
    ST_PointOnSurface(geom) AS geom
FROM countries
WHERE geom IS NOT NULL
  AND NOT ST_IsEmpty(geom);

-- GeoJSONに出力
COPY country_labels
TO 'data/country_labels.geojson'
WITH (
    FORMAT GDAL,
    DRIVER 'GeoJSON',
    SRS 'EPSG:4326'
);
```

duckdbでSQLを実行します。

```bash
duckdb -f create_country_labels.sql
```

# MVTに変換

[tippecanoe](https://github.com/felt/tippecanoe)でGeoJSONからMVTを生成します。

```bash
tippecanoe \
  -o data/countries.mbtiles \
  -Z0 -z5 \
  -L countries:data/countries.geojson \
  -L country_labels:data/country_labels.geojson \
  -pf -pk \
  --drop-rate=0 \
  -f

# PMTilesに変換
pmtiles convert data/countries.mbtiles data/countries.pmtiles
```

# StyleJSONでglobal stateを使ってレイヤ定義

フィーチャのプロパティにはNAME_EN（英語名）やNAME_JA（日本語名）のような属性値が入っているので、アンダースコア（_）より後ろの値をglobal stateとして管理して、表示する属性を切り替えられるようにします。

StyleJSONでデフォルト値を定義しておきます。

```json
"state": {
    "language": { "default": "EN" }
}
```

その上で `global-state` 式でglobal stateのプロパティ値を取り出します。

例えば以下のようにラベルレイヤを定義できます。

```json
{
  "id": "country-labels",
  "type": "symbol",
  "source": "countries",
  "source-layer": "country_labels",
  "layout": {
      "text-field": [
          "coalesce",
          [
              "get",
              [
                  "concat",
                  "NAME_",
                  ["global-state", "language"]
              ]
          ],
          ["get", "NAME_EN"],
          ["get", "ADMIN"]
      ],
      "text-font": ["Noto Sans Regular"],
      "text-size": 12,
      "text-allow-overlap": false
  },
  "paint": {
      "text-color": "#1f2937",
      "text-halo-color": "#ffffff",
      "text-halo-width": 1.5
  }
}
```

# Map初期化でglobal stateを上書き

MapLibre GL JS v6.14.0からMapオブジェクト初期化時にglobal stateのデフォルト値を上書きできるようになりました。

StyleJSON側では英語名をデフォルトとしていましたが、それを日本語に上書きするには以下のようにします。

```javascript
const map = new Map({
    container: 'map',
    hash: true,
    style: './style.json',
    center: [139.767125, 35.681236],
    zoom: 2,
    // v6.14.0: 初回描画時に日本語を指定
    globalState: {
        language: 'JA'
    },
});
```

# プルダウンメニューで言語（global state）を切り替え

プルダウンメニューで言語の切り替え機能も追加しておきます。

HTMLでプルダウンメニューを作成します。

```html
<select id="language">
  <option value="AR">العربية (アラビア語)</option>
  <option value="BN">বাংলা (ベンガル語)</option>
  <option value="DE">Deutsch (ドイツ語)</option>
  <option value="EN">English (英語)</option>
  <option value="ES">Español (スペイン語)</option>
  <option value="FA">فارسی (ペルシア語)</option>
  <option value="FR">Français (フランス語)</option>
  <option value="EL">Ελληνικά (ギリシャ語)</option>
  <option value="HE">עברית (ヘブライ語)</option>
  <option value="HI">हिन्दी (ヒンディー語)</option>
  <option value="HU">Magyar (ハンガリー語)</option>
  <option value="ID">Bahasa Indonesia (インドネシア語)</option>
  <option value="IT">Italiano (イタリア語)</option>
  <option value="JA" selected>日本語</option>
  <option value="KO">한국어 (韓国語)</option>
  <option value="NL">Nederlands (オランダ語)</option>
  <option value="PL">Polski (ポーランド語)</option>
  <option value="PT">Português (ポルトガル語)</option>
  <option value="RU">Русский (ロシア語)</option>
  <option value="SV">Svenska (スウェーデン語)</option>
  <option value="TR">Türkçe (トルコ語)</option>
  <option value="UK">Українська (ウクライナ語)</option>
  <option value="UR">اردو (ウルドゥー語)</option>
  <option value="VI">Tiếng Việt (ベトナム語)</option>
  <option value="ZH">简体中文 (中国語・簡体字)</option>
  <option value="ZHT">繁體中文 (中国語・繁体字)</option>
</select>
```

Mapの`setGlobalStateProperty`メソッドでglobal stateを更新します。

```javascript
const select = document.getElementById('language');

select.addEventListener('change', event => {
    const language = event.target.value;
    map.setGlobalStateProperty('language', language);
});
```

これで言語の切り替えが可能になります。

以下のサイトで確認できます（クリックするとリンク先に飛びます）。

<a href="https://hirofumikanda.github.io/maplibre-globalstate-viewer/" target="_blank">
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/88d1af89-2f1b-41b3-a460-fe17fda3890a.png" alt="MapLibre Global State Viewer" />
</a>

# まとめ

- MapLibre GL JS v6.14.0からMap初期化時にglobal stateのデフォルト値の更新が可能となった。
- global state機能を使うことでフィーチャ横断でのスタイルの切り替えが容易に実現できる。
