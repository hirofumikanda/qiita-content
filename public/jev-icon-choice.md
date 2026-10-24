---
title: Jevを使った地図アイコン選択
tags:
  - Jev
  - webGIS
  - MapLibre
private: false
updated_at: '2026-10-24T19:03:28+09:00'
id: 07e2a5939734fdc96ead
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
# はじめに

Web地図で注記データをスタイリングするときに、注記のカテゴリ・分類とアイコンを関連付ける必要があります。

種類が限られているときは余り大変ではありませんが、カテゴリやアイコンが細かい粒度まで存在すると骨の折れる作業になることも少なくありません。

そこで、TypeSafeAIの[Jev](https://typesafe.ai/blog/introducing-system-one-models-and-jev)のChoiceを使ってアイコンの自動選択を実施してみました。

本記事では、その内容を紹介します。

# Maki iconをダウンロード

アイコンは、MapboxからCC0ライセンスで提供されているMakiアイコンを使います。

<a href="https://labs.mapbox.com/maki-icons/" target="_blank">
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/4736f77c-704c-4ec8-9053-e1a2747382b6.png" alt="maki icon" />
</a>

Jevへのリクエストに使うため、Makiアイコンの名称のみの一覧を以下のスクリプトで作成します。

```python
from pathlib import Path
import json

icons = sorted(
    p.stem
    for p in Path("maki/icons").glob("*.svg")
)

Path("maki_icons.json").write_text(
    json.dumps(icons, ensure_ascii=False, indent=2),
    encoding="utf-8"
)

print(f"{len(icons)} icons")
```

作成されたファイルは以下のようになります。

- **maki_icons.json**
```json
[
  "aerialway",
  "airfield",
  "airport",
  "alcohol-shop",
  "american-football",
  ...
]
```

# 国土地理院-最適化ベクトルタイルの注記カテゴリを抽出

Makiアイコンを対応づける対象として、国土地理院の最適化ベクトルタイルを使います。

https://github.com/gsi-cyberjapan/optimal_bvmap

注記分類コードとその意味は[注記分類コード・地物種別コード一覧](https://maps.gsi.go.jp/help/pdf/vector/optbv_featurecodes.xlsx)（XLSX形式）で確認できます。

エクセルのままだと使いづらいので、以下のスクリプトで注記分類コードのみの一覧CSVを作成します。

```python
#!/usr/bin/env python3
"""Extract Anno label categories from GSI optimal_bvmap feature codes XLSX."""
import csv
import sys
from pathlib import Path
from artifact_tool import Blob, SpreadsheetFile

source = Path(sys.argv[1]) if len(sys.argv) > 1 else Path('optbv_featurecodes.xlsx')
out = Path(sys.argv[2]) if len(sys.argv) > 2 else Path('categories.csv')
workbook = SpreadsheetFile.import_xlsx(Blob.load(str(source)))
sheet = next(s for s in workbook.worksheets.items if s.name == '注記分類コード')
rows = sheet.get_range('A1:Q174').values
if rows[0][:3] != ['ソースレイヤー', 'vt_code', '分類']:
    raise ValueError('Unexpected sheet format')
entries = []
current_layer = None
for row in rows[1:]:
    layer, code, category = row[:3]
    if layer:
        current_layer = str(layer).strip()
    if current_layer != 'Anno' or not isinstance(code, (int, float)) or not category:
        continue
    if int(code) != code:
        raise ValueError(f'Non-integral vt_code: {code}')
    entries.append((str(int(code)), str(category).strip()))
if len({c for c, _ in entries}) != len(entries):
    raise ValueError('Duplicate vt_code')
with out.open('w', encoding='utf-8-sig', newline='') as f:
    writer = csv.writer(f)
    writer.writerow(['vt_code', 'category'])
    writer.writerows(entries)
print(f'Created {out}: {len(entries)} Anno codes')
```

以下のようなCSVが作成されます。

- **categories.csv**

```csv
vt_code,category
110,市区町村
120,飛び地
130,郡
140,都道府県
210,公称（町字名）
220,集落名称（通称）
...
```

# JevにChoice Questionを投げる

JevにChoice Questionをリクエストして、各注記カテゴリに対する最適なアイコンを選択します。

https://docs.typesafe.ai/primitives/choice

TypeSafe Python SDKを使ってコーディングします。

https://docs.typesafe.ai/sdk/python

ここで、一つ留意点として考慮しておきたいのは、すべての注記分類に対してアイコンを表示するのが適切とは限らないことです。

例えば、役所や官公庁の建物名にアイコンを表示するのは視認性の改善につながる可能性がありますが、行政名は広域の面に対して付与される注記であるため、アイコンを表示するのは一般的ではありません。

また、道路名や鉄道路線名などの一定の長さを持つ線状地物に対する注記もアイコンを表示するのは適切とはいえません。

そのため、「アイコンの選択が適切でない」という選択肢もJevに与えておくのが妥当です。

以下のようなPythonスクリプトでリクエストを送付します。

```python

import csv
import json
import os

from typesafe_sdk import Choice, TypeSafeClient

# 実在するMakiアイコン名
with open("maki_icons.json", encoding="utf-8") as f:
    icons = json.load(f)

with open("categories.csv", encoding="utf-8-sig") as f:
    categories = list(csv.DictReader(f))

# 念のため重複排除
icons = sorted(set(icons))

criteria = {
    name: f"Maki icon: {name}"
    for name in icons
}
criteria["none"] = (
    "No suitable POI icon; better displayed "
    "as text only or requires subtype information."
)

results = []

with TypeSafeClient() as client:
    for row in categories:
        state = {
            "vt_code": row["vt_code"],
            "category": row["category"],
            "context": (
                "Japanese topographic map annotation. "
                "Choose an icon for a map label."
            ),
        }

        response = client.system_one(
            state=state,
            questions={
                "icon": Choice(
                    instructions=(
                        "Choose the most semantically "
                        "appropriate Maki POI icon for "
                        "this Japanese map label category. "
                        "Prefer none for broad, ambiguous, "
                        "linear, or area-based labels."
                    ),
                    criteria=criteria,
                )
            },
        )

        answer = response.answers["icon"]

        results.append({
            **row,
            "icon": answer.choice,
            "confidence": answer.confidence,
            "review": answer.confidence < 0.8,
        })

        print(
            row["vt_code"],
            row["category"],
            answer.choice,
            answer.confidence,
        )

with open(
    "icon_mapping.json", "w", encoding="utf-8"
) as f:
    json.dump(results, f, ensure_ascii=False, indent=2)
```

"icon": "none"以外のものをまとめると次のような結果が得られました（逆にいうとここに記載がないものはすべて「アイコンなし」が選択されました）。

"category"が最適化ベクトルタイルの注記分類名で、"icon"がJevによって選択されたMaki iconの名称です。

```json
[
  {"vt_code": "312", "category": "山、岳、峰等", "icon": "mountain", "confidence": 0.89, "review": false},
  {"vt_code": "313", "category": "尖峰、丘、塚等", "icon": "mountain", "confidence": 0.64, "review": true},
  {"vt_code": "314", "category": "山、岳、峰等（３０００ｍ以上）", "icon": "mountain", "confidence": 0.94, "review": false},
  {"vt_code": "315", "category": "山、岳、峰等（１０００ｍ以上）", "icon": "mountain", "confidence": 0.92, "review": false},
  {"vt_code": "316", "category": "山、岳、峰等（１０００ｍ未満）", "icon": "mountain", "confidence": 0.91, "review": false},
  {"vt_code": "344", "category": "海、灘", "icon": "beach", "confidence": 0.6, "review": true},
  {"vt_code": "412", "category": "道路施設（ＩＣ、PA、道の駅等）", "icon": "highway-rest-area", "confidence": 0.78, "review": true},
  {"vt_code": "413", "category": "道路構造物（橋、トンネル等）", "icon": "bridge", "confidence": 0.49, "review": true},
  {"vt_code": "422", "category": "鉄道駅名", "icon": "rail-metro", "confidence": 0.57, "review": true},
  {"vt_code": "431", "category": "港湾", "icon": "harbor", "confidence": 0.96, "review": false},
  {"vt_code": "432", "category": "港湾施設（フェリー発着所、埠頭等）", "icon": "harbor", "confidence": 0.52, "review": true},
  {"vt_code": "441", "category": "空港名", "icon": "airport", "confidence": 0.95, "review": false},
  {"vt_code": "511", "category": "構造物名称（高塔、煙突等）", "icon": "observation-tower", "confidence": 0.65, "review": true},
  {"vt_code": "521", "category": "ダム", "icon": "dam", "confidence": 0.95, "review": false},
  {"vt_code": "522", "category": "堰", "icon": "dam", "confidence": 0.62, "review": true},
  {"vt_code": "532", "category": "史跡名勝天然記念物", "icon": "historic", "confidence": 0.5, "review": true},
  {"vt_code": "533", "category": "漁港", "icon": "harbor", "confidence": 0.99, "review": false},
  {"vt_code": "534", "category": "公園", "icon": "park", "confidence": 0.88, "review": false},
  {"vt_code": "611", "category": "合同庁舎", "icon": "building", "confidence": 0.54, "review": true},
  {"vt_code": "612", "category": "国の機関（合同庁舎、矯正施設及び自衛隊を除く）", "icon": "town-hall", "confidence": 0.36, "review": true},
  {"vt_code": "613", "category": "矯正施設（刑務所、少年院等）", "icon": "prison", "confidence": 0.98, "review": false},
  {"vt_code": "621", "category": "都道府県庁、北海道の総合振興局・振興局、支庁", "icon": "town-hall", "confidence": 0.72, "review": true},
  {"vt_code": "632", "category": "短期大学", "icon": "college-JP", "confidence": 0.55, "review": true},
  {"vt_code": "634", "category": "特殊学校", "icon": "school-JP", "confidence": 0.62, "review": true},
  {"vt_code": "651", "category": "水族館・動植物園", "icon": "aquarium", "confidence": 0.81, "review": false},
  {"vt_code": "653", "category": "発電所", "icon": "industry", "confidence": 0.64, "review": true},
  {"vt_code": "661", "category": "神社", "icon": "religious-shinto", "confidence": 0.99, "review": false},
  {"vt_code": "662", "category": "寺院", "icon": "religious-buddhist", "confidence": 0.87, "review": false},
  {"vt_code": "671", "category": "商業施設", "icon": "commercial", "confidence": 0.59, "review": true},
  {"vt_code": "810", "category": "山", "icon": "mountain", "confidence": 0.92, "review": false},
  {"vt_code": "822", "category": "滝", "icon": "waterfall", "confidence": 0.97, "review": false},
  {"vt_code": "831", "category": "温泉、泉、噴気口", "icon": "hot-spring", "confidence": 0.99, "review": false},
  {"vt_code": "860", "category": "道の駅", "icon": "highway-rest-area", "confidence": 0.89, "review": false},
  {"vt_code": "870", "category": "公園", "icon": "park", "confidence": 0.79, "review": true},
  {"vt_code": "882", "category": "保健所", "icon": "doctor", "confidence": 0.19, "review": true},
  {"vt_code": "883", "category": "警察", "icon": "police-JP", "confidence": 0.59, "review": true},
  {"vt_code": "884", "category": "消防", "icon": "fire-station", "confidence": 0.43, "review": true},
  {"vt_code": "885", "category": "学校", "icon": "school-JP", "confidence": 0.61, "review": true},
  {"vt_code": "886", "category": "病院", "icon": "hospital-JP", "confidence": 0.67, "review": true},
  {"vt_code": "887", "category": "郵便局", "icon": "post-JP", "confidence": 0.95, "review": false},
  {"vt_code": "889", "category": "博物館", "icon": "museum", "confidence": 0.97, "review": false},
  {"vt_code": "890", "category": "老人ホーム", "icon": "shelter", "confidence": 0.25, "review": true},
  {"vt_code": "1301", "category": "都市記号（人口100万人以上）", "icon": "city", "confidence": 0.71, "review": true},
  {"vt_code": "1302", "category": "都市記号（人口50万～100万人未満）", "icon": "city", "confidence": 0.49, "review": true},
  {"vt_code": "1303", "category": "都市記号（人口50万人未満）", "icon": "town", "confidence": 0.46, "review": true},
  {"vt_code": "1401", "category": "都道府県所在地", "icon": "city", "confidence": 0.42, "review": true},
  {"vt_code": "1402", "category": "市役所・東京都の区役所（都道府県所在都市以外）", "icon": "town-hall", "confidence": 0.93, "review": false},
  {"vt_code": "1403", "category": "町・村役場・政令指定都市の区役所", "icon": "town-hall", "confidence": 0.9, "review": false},
  {"vt_code": "2943", "category": "サービスエリア", "icon": "highway-rest-area", "confidence": 0.74, "review": true},
  {"vt_code": "2944", "category": "パーキングエリア", "icon": "highway-rest-area", "confidence": 0.85, "review": false},
  {"vt_code": "2945", "category": "スマートインターチェンジ", "icon": "toll", "confidence": 0.46, "review": true},
  {"vt_code": "3201", "category": "官公署", "icon": "town-hall", "confidence": 0.62, "review": true},
  {"vt_code": "3204", "category": "外国公館", "icon": "embassy", "confidence": 0.98, "review": false},
  {"vt_code": "3205", "category": "市役所・東京都の区役所", "icon": "town-hall", "confidence": 0.96, "review": false},
  {"vt_code": "3206", "category": "町村役場・政令指定都市の区役所", "icon": "town-hall", "confidence": 0.93, "review": false},
  {"vt_code": "3211", "category": "交番", "icon": "police-JP", "confidence": 0.93, "review": false},
  {"vt_code": "3212", "category": "学校　高等学校・中等教育学校", "icon": "school-JP", "confidence": 0.82, "review": false},
  {"vt_code": "3213", "category": "学校　中学校", "icon": "school-JP", "confidence": 0.9, "review": false},
  {"vt_code": "3214", "category": "学校　小学校", "icon": "school-JP", "confidence": 0.93, "review": false},
  {"vt_code": "3215", "category": "老人ホーム", "icon": "shelter", "confidence": 0.24, "review": true},
  {"vt_code": "3216", "category": "博物館法の登録博物館・博物館相当施設", "icon": "museum", "confidence": 0.97, "review": false},
  {"vt_code": "3217", "category": "図書館", "icon": "library", "confidence": 0.88, "review": false},
  {"vt_code": "3218", "category": "郵便局", "icon": "post-JP", "confidence": 0.93, "review": false},
  {"vt_code": "3221", "category": "灯台", "icon": "lighthouse", "confidence": 0.7, "review": true},
  {"vt_code": "3231", "category": "神社　通常の神社", "icon": "religious-shinto", "confidence": 0.99, "review": false},
  {"vt_code": "3232", "category": "寺院　通常の寺院", "icon": "religious-buddhist", "confidence": 0.94, "review": false},
  {"vt_code": "3241", "category": "警察署", "icon": "police-JP", "confidence": 0.87, "review": false},
  {"vt_code": "3242", "category": "消防署", "icon": "fire-station-JP", "confidence": 0.58, "review": true},
  {"vt_code": "3243", "category": "病院", "icon": "hospital-JP", "confidence": 0.74, "review": true},
  {"vt_code": "3261", "category": "工場", "icon": "industry", "confidence": 0.84, "review": false},
  {"vt_code": "4101", "category": "煙突", "icon": "communications-tower", "confidence": 0.31, "review": true},
  {"vt_code": "4102", "category": "風車", "icon": "windmill", "confidence": 0.93, "review": false},
  {"vt_code": "4104", "category": "記念碑", "icon": "monument-JP", "confidence": 0.73, "review": true},
  {"vt_code": "4105", "category": "自然災害伝承碑", "icon": "monument-JP", "confidence": 0.6, "review": true},
  {"vt_code": "6301", "category": "墓地", "icon": "cemetery-JP", "confidence": 0.73, "review": true},
  {"vt_code": "6331", "category": "温泉", "icon": "hot-spring", "confidence": 1.0, "review": false},
  {"vt_code": "6332", "category": "噴火口・噴気口", "icon": "volcano", "confidence": 0.95, "review": false},
  {"vt_code": "6341", "category": "史跡・名勝・天然記念物", "icon": "historic", "confidence": 0.58, "review": true},
  {"vt_code": "6361", "category": "港湾", "icon": "harbor", "confidence": 0.93, "review": false},
  {"vt_code": "6362", "category": "漁港", "icon": "harbor", "confidence": 1.0, "review": false},
  {"vt_code": "6367", "category": "特定重要港", "icon": "harbor", "confidence": 0.94, "review": false},
  {"vt_code": "6368", "category": "主要な港", "icon": "harbor", "confidence": 0.96, "review": false},
  {"vt_code": "6371", "category": "空港", "icon": "airport", "confidence": 0.96, "review": false},
  {"vt_code": "6373", "category": "自衛隊等の飛行場", "icon": "airfield", "confidence": 0.63, "review": true},
  {"vt_code": "6375", "category": "国際空港", "icon": "airport", "confidence": 0.98, "review": false},
  {"vt_code": "6376", "category": "主要な空港", "icon": "airport", "confidence": 0.97, "review": false},
  {"vt_code": "7101", "category": "電子基準点", "icon": "marker", "confidence": 0.42, "review": true},
  {"vt_code": "7102", "category": "三角点", "icon": "triangle", "confidence": 0.76, "review": true},
  {"vt_code": "7221", "category": "標高（火山）", "icon": "volcano", "confidence": 0.72, "review": true},
  {"vt_code": "8103", "category": "発電所等", "icon": "industry", "confidence": 0.42, "review": true},
  {"vt_code": "8105", "category": "電波塔", "icon": "communications-tower", "confidence": 0.98, "review": false}
]
```

confidenceが低いものもあり、特に0.5未満のものは見直しが必要そうなものもありますが、概ね問題なさそうにみえます。

# MapLibreで表示

Jevのレスポンスから作成した注記カテゴリとアイコンの対応表を使ってMakiアイコンを注記に表示してみます。

最適化ベクトルタイルの標準地図風スタイル（std.json）ベースで、注記のアイコンだけMakiアイコンに置き換えます。

以下が実際にMapLibreで作成したサイトです（クリックするとリンク先に飛びます）。

<a href="https://hirofumikanda.github.io/gsi-maki-viewer/#15/35.68124/139.76713" target="_blank">
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/38666687-a2a5-4246-815f-6cf289e7dfa4.png" alt="maplibre" />
</a>

余り違和感なくみられるアイコンにはなっているかと思います。

# まとめ

- **Jev**の**Choick Question**を使って注記のカテゴリに対するアイコンの選択を自動化
- 注記にはアイコンを表示すべきでないものもあるため「アイコンなし」も選択肢に追加
- confidenceが低くチューニンを要するものもあったが、概ね適切な回答が得られた
- 地図スタイリングにおけるJev適用の可能性の一つとしてアイコン選択を考えられるのではないか
