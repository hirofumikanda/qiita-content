---
title: MapLibre GL JSのfeature-stateで実現する動的スタイリング
tags:
  - GIS
  - MVT
  - MapLibre
private: false
updated_at: '2026-01-03T14:20:17+09:00'
id: bfe9ace67c201d29d728
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

[MapLibre GL JS](https://maplibre.org/maplibre-gl-js/docs/)でインタラクティブな地図を作る際、ユーザーの操作に応じて地物（フィーチャ）のスタイルを動的に変更したいケースがしばしばあります。例えば、マウスホバー時にフィーチャを強調表示したり、選択状態を視覚的に示したりする場合などです。

このような動的スタイリングを実現する仕組みとして、MapLibre GL JSには[feature-state](https://maplibre.org/maplibre-style-spec/expressions/#feature-state)という機能があります。本記事では、feature-stateの使い方と、利用する際に必要不可欠な**フィーチャidの設定**について紹介したいと思います。

## feature-stateとは

feature-stateは、個々のフィーチャに対して一時的な状態（state）を割り当て、その状態に応じて、対象のフィーチャに限定してスタイルを変更する機能です。

feature-stateを使用すると、**データソースを変更せず**、**個別のフィーチャのみ**にスタイルを効率的に適用することができます。フィルタを使って別途レイヤーを追加する必要はありません。

## feature-stateを使用するにはフィーチャidが必要

**feature-stateを使用するには、各フィーチャを識別するidが必要になります。** これはfeature-stateの最も重要な前提条件です。

idを設定する方法は2つあります。

### 方法1: タイル生成時にフィーチャidを埋め込む

タイル生成ツール（[tippecanoe](https://github.com/felt/tippecanoe)など）で、**フィーチャid**を設定する機能が提供されています。フィーチャidは、プロパティとは別にフィーチャを識別するために付与できる識別番号です。

[MVT仕様](https://github.com/mapbox/vector-tile-spec/tree/master/2.1#42-features)をみると、以下の要件が記載されています。

- フィーチャidは任意のフィールドである（必須ではない）
- 同一レイヤ内で一意であるべき

また、[vector_tile.proto](https://github.com/mapbox/vector-tile-spec/blob/master/2.1/vector_tile.proto)には、idのデータ型は`uint64`となっていますが、JavaScriptのNumber型で安全に扱えるのは53ビットまで（[Number.MAX_SAFE_INTEGER](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Number/MAX_SAFE_INTEGER)を参照）のため、`2^53 - 1`までの値を付与しておくのが安全かと思います。

#### tippecanoeで埋め込む場合

tippecanoeでMVTを作成する場合、--use-attribute-for-idオプションを使うことでフィーチャidを付与できます。

```bash
# プロパティ'id'をfeature idとして使用
tippecanoe -o output.mbtiles \
  --use-attribute-for-id=id \
  input.geojson
```

このとき、idは整数である必要があります。ただし、フィーチャidとして使用したいプロパティが整数にキャスト可能な文字列である場合は、--convert-stringified-ids-to-numbersオプションを追加することで、整数に変換した上でフィーチャidとして設定してくれます。

```bash
# プロパティ'id'を文字列から整数に変換した上でfeature idとして使用
tippecanoe -o output.mbtiles \
  --use-attribute-for-id=id --convert-stringified-ids-to-numbers \  
  input.geojson
```

詳細は、tippecanoeの[GitHubのREADME](https://github.com/felt/tippecanoe?tab=readme-ov-file#modifying-feature-attributes)で確認できます。

#### PostGISのST_AsMVTで埋め込む場合

PostGISのST_AsMVTでもフィーチャidを設定できます。

```sql
-- 'id'カラムをfeature idに設定
SELECT ST_AsMVT(tile, 'layer_name', 4096, 'geom', 'id')  
FROM (
  SELECT
    id,  -- この列がfeature idになる
    name,
    population,
    ST_AsMVTGeom(
      geom,
      ST_TileEnvelope(14, 14550, 6451),
      4096,
      256,
      true
    ) AS geom
  FROM your_table
  WHERE geom && ST_TileEnvelope(14, 14550, 6451)
) AS tile;
```

詳細は[リファレンス](https://postgis.net/docs/ja/ST_AsMVT.html)を参照ください。

### 方法2: promoteIdを使用してプロパティをIDに昇格

タイルにフィーチャidが含まれていない場合、MapLibre GL JSの[promoteId](https://maplibre.org/maplibre-style-spec/sources/#promoteid)オプションを使用して、プロパティをフィーチャidとして扱うことができます。

```javascript
map.addSource('vector-tiles', {
  type: 'vector',
  url: 'pmtiles://https://example.com/tiles.pmtiles',
  promoteId: {
    'layer-name': 'feature_id_property'  // ソースレイヤがもつ特定のプロパティをidとして昇格
  }
});
```

フィーチャidとは異なり、`promoteId`は、数値や文字列など任意のプリミティブ型の値で使用できます。

以上から、既にフィーチャプロパティの中にフィーチャを一意に特定する情報がある場合は、`promoteId`を使用し、そのようなプロパティがない場合は、タイル作成時に別途フィーチャidを埋め込むことになります。

## feature-stateを使ってみる

それでは前提条件がわかったところで、実際にfeature-stateを使ってみたいと思います。

### 1. データの準備

テストデータとして、Natural Earthの[Admin o - Countries](https://www.naturalearthdata.com/downloads/10m-cultural-vectors/10m-admin-0-countries/)を使用します。

国名属性は標準名（英語名）と日本語名のみにし、あらかじめ連番でid属性を付加しておきます。
ジオメトリの属性を編集する方法はいろいろあると思いますが、ここでは[duckdb](https://duckdb.org/)を使います。

```bash
$ duckdb
D install spatial; # インストール済みの場合は不要
D load spatial;
D create table country as select name, name_ja, geom from 'ne_10m_admin_0_countries.shp';
D create table country_id as select row_number() over () as id, * from country;
D copy (select * from country_id) to 'country_id.geojson' (format gdal, driver 'GeoJSON');
```

その上で、tippecanoeを使ってMVTを作成します。

```bash
tippecanoe -f -Q -o ./country.pmtiles \
-l country ./country_id.geojson \
--use-attribute-for-id=id \
-Z0 -z8 -pf -pk
```

これでテスト用のタイルデータを準備できました。

### 2. 状態プロパティを考える

データができたところで、今度はフィーチャに付与する状態プロパティを検討します。
今回は、マウスホバー時と左クリック選択時の状態を定義し、それぞれの状態にしたがってスタイルを変更することを考えます。

それぞれを以下の状態プロパティで管理したいと思います。なお、プロパティ名は任意です（mousehoverにしてもchosenにしても問題ありません）。

- マウスホバー時(hover): boolean型でホバー時はtrue、それ以外はfalse
- 左クリック選択時(selected): boolean型で選択時はtrue、それ以外はfalse

### 3. スタイリング

MapLibre GL JSで、style.jsonを作成します。
通常のスタイリングに加えて、上記で定義した状態についてもcase式でスタイリングを設定します。

```json
{
  "version": 8,
  "sources": {
    "country": {
      "type": "vector",
      "url": "pmtiles:////data/country.pmtiles",
      "attribution": "<a href=\"https://www.naturalearthdata.com/downloads/10m-cultural-vectors/10m-admin-0-countries/\" target=\"_blank\">Made with Natural Earth</a>",
      "minzoom": 0,
      "maxzoom": 8
    }
  },
  "glyphs": "font/{fontstack}/{range}.pbf",
  "layers": [
    {
      "id": "background",
      "type": "background",
      "paint": {
        "background-color": "rgb(138, 216, 236)"
      }
    },
    {
      "id": "country",
      "type": "fill",
      "source": "country",
      "source-layer": "country",
      "paint": {
        "fill-color": [
          "case",
          ["boolean", ["feature-state", "selected"], false],
          "#ffaa00",
          ["boolean", ["feature-state", "hover"], false],
          "#0080ff",
          "#ffffff"
        ],
        "fill-opacity": [
          "case",
          ["boolean", ["feature-state", "selected"], false],
          0.8,
          ["boolean", ["feature-state", "hover"], false],
          0.6,
          1
        ],
        "fill-outline-color": "rgb(0, 0, 0)"
      },
      "minzoom": 0
    }
  ]
}

```

### 4. feature-stateの取得と削除

JavaScriptで、マウスホバー時とクリック時について、イベントハンドラーを定義し、その中でfeature-stateを設定します。
feature-stateの設定は、MapLibre GL JSの[setFeatureState](https://maplibre.org/maplibre-gl-js/docs/API/classes/Map/#setfeaturestate)メソッドを使います。

#### マウスホバー時の処理

```javascript
let hoveredFeatureId = null;

map.on("mousemove", "country", (e) => {
  if (e.features.length > 0) {
    // 前回のホバー状態をクリア
    if (hoveredFeatureId !== null) {
      map.setFeatureState(
        {
          source: "country",
          sourceLayer: "country",
          id: hoveredFeatureId,
        },
        { hover: false }
      );
    }

    // 新しいフィーチャにホバー状態を設定
    hoveredFeatureId = e.features[0].id;

    if (hoveredFeatureId !== undefined) {
      map.setFeatureState(
        {
          source: "country",
          sourceLayer: "country",
          id: hoveredFeatureId,
        },
        { hover: true }
      );
    }
  }
});

map.on("mouseleave", "country", () => {
  if (hoveredFeatureId !== null) {
    map.setFeatureState(
      {
        source: "country",
        sourceLayer: "country",
        id: hoveredFeatureId,
      },
      { hover: false }
    );
  }
  hoveredFeatureId = null;
});
```

#### クリック選択時の処理

選択時にフィーチャのプロパティをポップアップで表示するようにしてみます。

```javascript
let selectedFeatureId = null;
let popup = null;

map.on("click", "country", (e) => {
  if (e.features.length > 0) {
    const clickedFeature = e.features[0];
    const clickedId = clickedFeature.id;

    // IDが存在しない場合は処理しない
    if (clickedId === undefined) {
      return;
    }

    // 同じフィーチャをクリックした場合は選択解除
    if (selectedFeatureId === clickedId) {
      // 前の選択を解除
      map.setFeatureState(
        {
          source: "country",
          sourceLayer: "country",
          id: selectedFeatureId,
        },
        { selected: false }
      );
      selectedFeatureId = null;

      // ポップアップを閉じる
      if (popup) {
        popup.remove();
        popup = null;
      }
    } else {
      // 前回の選択を解除
      if (selectedFeatureId !== null) {
        map.setFeatureState(
          {
            source: "country",
            sourceLayer: "country",
            id: selectedFeatureId,
          },
          { selected: false }
        );
      }

      // 既存のポップアップがあれば削除
      if (popup) {
        popup.remove();
        popup = null;
      }

      // 新しいフィーチャを選択
      selectedFeatureId = clickedId;
      map.setFeatureState(
        {
          source: "country",
          sourceLayer: "country",
          id: selectedFeatureId,
        },
        { selected: true }
      );

      // フィーチャ情報を表示
      const props = clickedFeature.properties;
      const coordinates = e.lngLat;
      
      const popupContent = `
        <div style="font-family: Arial, sans-serif;">
          <p style="margin: 0;">${props.name}</p>
          <p style="margin: 0;">${props.name_ja}</p>
        </div>
      `;

      // 新しいポップアップを作成
      popup = new maplibregl.Popup({ closeOnClick: false })
        .setLngLat(coordinates)
        .setHTML(popupContent)
        .addTo(map);

      // ポップアップが閉じられたときの処理
      popup.on("close", () => {
        if (selectedFeatureId !== null) {
          map.setFeatureState(
            {
              source: "country",
              sourceLayer: "country",
              id: selectedFeatureId,
            },
            { selected: false }
          );
          selectedFeatureId = null;
        }
        popup = null;
      });
    }
  }
});
```

上記の実装をしたウェブサイトが以下になります（画像をクリックするとサイトに飛びます）。

<a href="https://hirofumikanda.github.io/test-maplibre-feature-state/" target="_blank">
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/d8d7ed7e-d05f-43ac-8905-3dc3e4bddb68.png" width="400px"></a>

ソースコードは以下に置いています。

https://github.com/hirofumikanda/test-maplibre-feature-state

## まとめ

MapLibre GL JSのfeature-stateを使用する際のポイントは以下のとおりです。

- feature-stateを使用することで、レイヤーを追加することなく、フィーチャの状態に応じたインタラクティブなスタイリングを実現できる。
- ただし、feature-stateを使用するには、データソースにフィーチャidを付与する必要がある。
- フィーチャidを付与する方法は、タイル生成時に埋め込むか、`promoteId`で特定のプロパティをidとして代用するかである。

ユーザーアクションでスタイルに変化をつけたいときに、試してみてはいかがでしょうか。
