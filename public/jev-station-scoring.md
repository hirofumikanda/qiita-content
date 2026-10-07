---
title: Jevを使った地物の重要度スコアリング
tags:
  - Jev
  - WebGIS
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

ウェブ地図において、小さいズームレベルでは重要な地物のみを表示し、ズームレベルが大きくなるにつれて表示する地物数を増やすというのが一般的です。

その際どのような基準やロジックで重要な地物を間引いて抽出するかが課題になります。

最近話題の[Jev](https://typesafe.ai/)では、段階的な`Score`を回答させることができますが、上記の課題に対しても効率的に地物の重要度をスコアリングする手段として使えるのではないかと思っています。

そこで、本記事ではJevにおける地物の重要度スコアリングの有効性を検証してみたいと思います。

# 国土数値情報の鉄道駅をスコアリングしてみる

テスト対象として、[国土数値情報-駅別乗降客数データ](https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-S12-2024.html)を使います。

路線ごとの鉄道駅の乗降客数のデータが入っており、グループコードを使って駅ごとに集計することができるため、Jevでいい感じに重要度を判定してくれるのではないかと期待できるからです。

# Stateを作成する

乗降客数のダウンロードデータに含まれるGeoJSONからJevのリクエストに必要な`State`を作成します。

以下の情報を駅（グループコード）ごとに集計します。

| 項目 | 内容 |
| --- | --- |
| station_group_code | グループコード |
| name | 駅名 |
| passengers_per_day | 1日当たりの乗降客数 |
| operators | 運営会社 |
| lines | 路線名 |
| operator_count | 運営会社数 |
| line_count | 路線数 |

[duckdb](https://duckdb.org/)を使ってGeoJSONから必要な属性を抽出して集計します。

```sql
WITH src AS (
    SELECT *
    FROM ST_Read('S12-25_GML/UTF-8/S12-25_NumberOfPassengers.geojson')
),

station AS (
    SELECT
        S12_001g AS station_group_code,
        MIN(S12_001) AS name,

        SUM(
            CASE
                WHEN S12_058 = '1'
                 AND S12_059 = '1'
                THEN S12_061
            END
        ) AS passengers_per_day,

        LIST(DISTINCT S12_002)
            FILTER (WHERE S12_002 IS NOT NULL)
            AS operators,

        LIST(DISTINCT S12_003)
            FILTER (WHERE S12_003 IS NOT NULL)
            AS lines,

        COUNT(DISTINCT S12_002) AS operator_count,
        COUNT(DISTINCT S12_003) AS line_count

    FROM src
    GROUP BY S12_001g
)

SELECT *
FROM station
WHERE passengers_per_day IS NOT NULL
ORDER BY passengers_per_day DESC
```

この結果をJSONに変換して評価対象のStateに含めます（以下は渋谷駅の例）。

```json
{
    "station_group_code": "003922",
    "name": "渋谷",
    "passengers_per_day": 2897703,
    "operators": [
        "東日本旅客鉄道",
        "京王電鉄",
        "東京地下鉄",
        "東急電鉄"
    ],
    "lines": [
        "東横線",
        "山手線",
        "井の頭線",
        "3号線銀座線",
        "田園都市線",
        "11号線半蔵門線",
        "13号線副都心線"
    ],
    "operator_count": 4,
    "line_count": 7
}
```

# Jevにリクエストを投げる

上記で作成した`State`に評価基準を記載した`questions`を加えてリクエストボディのJSONを組み立てます。

```json
{
    "model": "jev-latest",
    "state": {
        "station_group_code": "003922",
        "name": "渋谷",
        "passengers_per_day": 2897703,
        "operators": [
            "東日本旅客鉄道",
            "京王電鉄",
            "東京地下鉄",
            "東急電鉄"
        ],
        "lines": [
            "東横線",
            "山手線",
            "井の頭線",
            "3号線銀座線",
            "田園都市線",
            "11号線半蔵門線",
            "13号線副都心線"
        ],
        "operator_count": 4,
        "line_count": 7
    },
    "questions": {
        "importance": {
            "type": "score",
            "instructions": "日本全国を対象とした一般的な地図における、駅名注記としての重要度を評価してください。乗降客数だけでなく、路線数、事業者数、広域交通上の役割、駅の認知度を総合的に評価してください。",
            "criteria": [
                "非常に局所的な駅。詳細な地図でのみ表示する価値がある",
                "地域内でのみ重要な駅",
                "市区町村レベルで重要な駅",
                "都市圏レベルで重要な駅",
                "広域交通上で重要な駅",
                "全国的に重要な主要駅"
            ]
        }
    }
}
```

6段階評価にしています。
Jevにリクエストを投げると以下のようなレスポンスが返ってきます。

```json
{
    "model": "jev-1.13.0",
    "answers": {
        "importance": {
            "type": "score",
            "score": 4.76,
            "confidence": 0.85,
            "legend": {
                "0": "非常に局所的な駅。詳細な地図でのみ表示する価値がある",
                "1": "地域内でのみ重要な駅",
                "2": "市区町村レベルで重要な駅",
                "3": "都市圏レベルで重要な駅",
                "4": "広域交通上で重要な駅",
                "5": "全国的に重要な主要駅"
            },
            "probabilities": {
                "0": 0.0,
                "1": 0.0,
                "2": 0.0,
                "3": 0.03,
                "4": 0.17,
                "5": 0.8
            }
        }
    },
    "usage": {
        "input_tokens": 627,
        "output_tokens": 17
    }
}
```

いくつかscoreをピックアップしてまとめると次のようになります。

| 駅 | score | confidence |
| --- | --- | --- |
| 東京 | 4.99 | 1.0 |
| 新宿 | 4.97 | 0.99 |
| 池袋 | 4.51 | 0.68 |
| 北千住 | 3.89 | 0.72 |
| 目黒 | 3.64 | 0.63 |
| 高田馬場 | 3.42 | 0.67 |
| 中目黒 | 2.97 | 0.78 |
| 町田 | 2.89 | 0.75 |
| 国分寺 | 2.82 | 0.78 |
| 田無 | 1.94 | 0.7 |
| 石神井公園 | 1.87 | 0.63 |
| 茗荷谷 | 1.78 | 0.33 |

そんなに間違ってはいなさそうなので、このまま可視化してみます。

# MapLibre GL JSで可視化してみる

Jevのレスポンスから得たスコアをベースに最小ズームレベルを設定したうえで、MapLibre GL JSで表示してみます。

スコアと最小ズームレベルの関係は以下のように整理しました。

| スコア | 最小ズームレベル |
| --- | --- |
| 5.00-4.50 | 4 |
| 4.49-4.00 | 6 |
| 3.99-3.00 | 8 |
| 2.99-2.00 | 10 |
| 1.99-1.00 | 12 |
| 0.99- | 14 |

スコアの高い重要な駅ほど小さいズームレベルから表示するようにしています。

この基準で駅注記を間引いてMVTを作成し、MapLibre GL JSで表示すると次のようになります。
（画像をクリックするとリンクに飛びます）

<a href="https://hirofumikanda.github.io/station-map-viewer/" target="_blank">
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/1fcd94f5-5ad3-4133-a4e7-4356b269cc45.png" alt="station-map" />
</a>

# まとめ
今回は、Jevから出てきたスコアについて、評価やチューニングをせずにそのまま利用したため、全体のバランスに偏りがあったり、局所的にみても精度が十分でない部分があったかもしれません。

それでもおおよそ大きな問題のないスコア付けができているように思います。

実用に耐えるかは精査と評価が必要ですが、地物の重要度判定の効率化・品質向上の手段として、Jevの活用を考えてみるのも一つ有効なのではないかと思います。