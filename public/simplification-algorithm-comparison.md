---
title: tippecanoeにおけるジオメトリ簡略化アルゴリズムの比較
tags:
  - tippecanoe
  - mvt
  - Douglas-Peucker
  - Visvalingam
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

[tippecanoe](https://github.com/felt/tippecanoe)で、MVT（Mapbox Vector Tile）を生成するとき、`--visvalingam` オプションを選択することができます。

これは、ジオメトリを簡略化するときに用いるアルゴリズムを**Douglas-Peucker**から**Visvalingam**に変更するオプションです。

本記事では、このオプションの効果について掘り下げてみたいと思います。

# Douglas-Peucker（ダグラス・ピーカー法）とは

デフォルトのDouglas-Peucker法は、以下の方法により「距離」を基準にして頂点数を減らします。

1. 折れ線の「始点」と「終点」を直線で結びます。
2. その直線から最も離れている中間点を探します。
3. 最も離れた点と直線の距離が、指定した許容誤差より大きい場合はその点を残し、線分を2つに分割して同じ処理を繰り返します。
4. 距離が許容誤差以下の場合は、中間の点をすべて削除します。

特徴的な角（鋭いカーブや頂点）を優先的に残すため、全体の見た目や形状が崩れにくいという特長があります。
その反面、緩やかなカーブを処理すると不自然なカクカクした直線が生じることがあります。

# Visvalingam（ヴィスヴァリンガム法）とは

他方、Visvalingam法では、「面積」を基準にして頂点数を減らします。

1. 折れ線上の各頂点と、その「直前の頂点」「直後の頂点」の3点で三角形を作ります。
2. その三角形の面積を計算します。
3. 面積が最も小さい頂点から順に削除していきます。
4. 許容する面積のしきい値に達するか、あらかじめ指定した頂点数になるまで繰り返します。

なめらかな曲線や自然な地形を保つことができる一方で、鋭い角や突起が削除されやすいという特徴があります。

# 両者の比較

OvertureMapsの河川データを使って違いを比較してみます。

釧路湿原周辺の河川データをDuckDBでダウンロードします。

```sql
LOAD spatial;

COPY (
    SELECT
        id,
        subtype,
        class,
        names.primary AS name,
        geometry
    FROM read_parquet(
        's3://overturemaps-us-west-2/release/2026-09-23.1/theme=base/type=water/*'
    )
    WHERE subtype = 'river'
      AND bbox.xmax >= 144.26186
      AND bbox.xmin <= 144.52796
      AND bbox.ymax >= 42.98414
      AND bbox.ymin <= 43.16728
) TO 'river.geojson'
WITH (
    FORMAT GDAL,
    DRIVER 'GeoJSONSeq',
    SRS 'EPSG:4326'
);
```

それぞれのアルゴリズムで簡略化したMVTを作成します。
比較のため、簡略化しない場合も作成しておきます。

```bash
# Douglas-Peucker
tippecanoe -o river_dp_s5.mbtiles -l river-dp -f -Z5 -z14 -S5 river.geojson

# Visvalingam
tippecanoe -o river_vw_s5.mbtiles -l river-vw -f -Z5 -z14 -av -S5 river.geojson

# 簡略化なし
tippecanoe -o river_no_simplification.mbtiles -l river-ps -f -Z5 -z14 -ps river.geojson

# 結合
tile-join river_dp_s5.mbtiles river_vw_s5.mbtiles river_no_simplification.mbtiles -o river.mbtiles --no-tile-size-limit

# PMTilesに変換
pmtiles convert river.mbtiles river.pmtiles
```

これをMapLibre GL JSで表示したのが以下のサイトです。

https://hirofumikanda.github.io/simplification-comparison-viewer/

### z14
Douglas-Peucker
![Douglas-Peucker-z14](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/5d19bcc7-dfe5-4b95-bf91-6ed611f24925.png)

Visvalingam
![Visvalingam-z14](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/15dd5523-3ec7-4dae-b428-767539b28e3a.png)

### z18
Douglas-Peucker
![Douglas-Peucker-z18](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/e24993fe-ae09-42ee-9133-ccca4d0bfe33.png)

Visvalingam
![Visvalingam-z18](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/ca018bb3-e2af-4b3d-ae35-ea4cc71516cc.png)

今回、z14までしかタイルを作成していないので、z15以上はz14タイルのオーバーズームですが、それでも違いが目視で明確にわかるようになるのはz16以上からです。

z18でみると、Douglas-Peuckerの方が形状の特徴を維持できており、Visvalingamはカーブや角をなだらか（直線的）にする効果があることがわかります。

なお、MBTilesのサイズは

| 簡略化 | データ容量 |
| --- | --- |
| なし | 368 KB |
| Douglas-Peucker | 268 KB |
| Visvalingam | 264 KB |

であり、両者でほとんど変わりません。

# まとめ

全体的にDouglas-Peuckerの方が元の形状を維持できており、単純に頂点を減らしたいだけあれば、デフォルトのDouglas-Peuckerのままでよいように思います。

他方、細かなカーブをなくして直線的にしたいなど、形状を滑らかにしつつ簡略化したい場合にはVisvalingamが有効になる可能性があります。

これらの違いを理解した上で試行錯誤しつつ最適な簡略化アルゴリズムを選択できるとよさそうです。