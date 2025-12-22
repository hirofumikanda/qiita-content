---
title: 'MVT（Mapbox Vector Tile）のバッファーに関する考察'
tags:
  - GIS
  - vectortile
  - MapLibre
private: false
updated_at: '2025-12-22T00:00:00+09:00'
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

**MVT（Mapbox Vector Tile）** は、ベクトル地図データを効率的に配信するためのタイル形式として、現代のWeb地図アプリケーションで広く使用されています。

GeoJSONなどのベクトル形式のデータをMVTに変換する際に、[tippecanoe](https://github.com/felt/tippecanoe)がしばしば使われますが、オプションで、**バッファー値**を設定することができることをご存知でしょうか。

デフォルトでもバッファー=5が設定されますが、その役割について説明しているドキュメントは多くないため、なんとなく隣接タイルとの整合性・連続性を担保してくれているのかなくらいの認識の方もいらっしゃるかもしれません。

そこで、本記事では、MVTのバッファーがどのような役割を果たしているのかを、実際に地図ライブラリ（MapLibre GL）での表示を通して、考察してみたいと思います。

なお、地図ライブラリのソースコードの確認までは実施していないため、検証結果の考察部分に関しては、あくまで私見であることをご承知おきください。

## MVTにおけるバッファーの概念

### バッファーとは何か

MVTにおける「バッファー」とは、**タイルの境界を超えて拡張された領域**のことを指します。具体的には、タイルの物理的な範囲（MVTでは通常4,096×4,096のタイル座標に正規化される）を越えて、周囲に余分にフィーチャデータを持つことを意味します。以下は[Overture Maps](https://docs.overturemaps.org/attribution/)のtransportationデータを異なるbufferサイズでMVTに変換して、(z/x/y)=(14/14552/6451)のタイルデータのみを表示した例です。

- buffer=0

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/46986b0c-f943-4972-ae87-b829f6da258e.png" width="400px">

- buffer=4

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/acae6fc5-a18c-42ae-8c64-ee933a29e0e6.png" width="400px">

- buffer=64

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/349dd933-893d-4361-8332-5cddd6ab8f16.png" width="400px">

タイル境界をはみ出してフィーチャを保有している様子がよくわかるかと思います。なお、いずれもfeltの[tippecanoe](https://github.com/felt/tippecanoe)でMVTに変換しています（以下同様）。例えば、以下のようなコマンドで作成できます。

```bash
tippecanoe -f -Q -o ./overturemaps_transportation_buffer0.mbtiles \
   -l transportation --simplification=5 --simplify-only-low-zooms \
   ./overturemaps_transportation.geojson \
   --buffer=0 -Z9 -z14 -pf -pk
```

バッファーの単位は画面ピクセルで、1タイル幅の1/256になります（以下同様）。

## バッファーがないとどうなるのか

バッファーがどのような役割を果たしているのかを検証するため、あえてタイルデータをバッファーなしで作成して、どのような表示になるのかを見てみたいと思います。

検証は、ジオメトリタイプがライン、ポリゴン、ポイントのそれぞれの場合で実施し、レンダリングには[MapLibre GL JS](https://maplibre.org/maplibre-gl-js/docs/)と[TileServer GL](https://tileserver.readthedocs.io/)を使用しました。

MapLibre GL JSはクライアントサイド（ブラウザ）でベクタータイルをレンダリングする地図ライブラリです。また、TileServer GLはサーバサイドでベクタータイルをレンダリングしてラスタタイルを配信する地図タイルサーバーになります。TileServer GLはレンダリングエンジンにMapLibre GL Nativeを使用しています。

それぞれの利用場面でバッファーがどのような影響を与えるか確認していきたいと思います。

なお、検証ではいずれも[Overture Maps](https://docs.overturemaps.org/getting-data/)のデータを使用しています。データは© OpenStreetMap contributors, Overture Maps Foundationに帰属しています。詳しくは[リンク](https://docs.overturemaps.org/attribution/)を参照ください。

### ライン（LineString）の場合

ラインの検証対象として、道路ネットワークデータに着目したいと思います。なお、線幅は、line-width=3で設定しています。バッファー=0のタイルで表示すると、以下のように描画されます。

- MapLibre GL JS(buffer=0)

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/7b4df292-da7c-4453-8ad0-2588402f717c.png" width="400px">

- TileServer GL(buffer=0)

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/978a8778-db1d-4a9d-bac1-12b6afbb2015.png" width="400px">

ちょっとわかりにくいかもしれませんが、タイル境界付近をよく見ていただくとわかるとおり、タイル境界に対して90度近い角度で入射しているラインは問題ありませんが、鋭角になっているラインでは形状の一部が欠落しているように（あるいは歪んでいるように）見えます。

これは、タイルごとに描画した後に、タイル境界で切り落とす（クリッピングする）ことによりみられる事象です。欠落しているのは、タイル境界の外側にあるフィーチャにより描画される部分であるため、バッファーがないと描画されません。

これは、線幅が大きくなるほど顕著になり、かつタイル境界に対して並行に近い（鋭角になる）ジオメトリほど欠落が大きくなります。限りなく並行に近くなると、最大で線幅の半分程度になります。

したがって、このような欠落を防止するためには線幅の半分以上のバッファーを入れる必要があります。

今回の場合、line-width=3で設定しているため、buffer=2あればこのような事象は防げることになります。

実際にbuffer=2でタイルを作成すると以下のようになります。

- MapLibre GL JS(buffer=2)

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/2a4f683c-a51b-4a62-8366-fc210d0b093d.png" width="400px">

- TileServer GL(buffer=2)

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/76824184-d3d5-4827-ac35-f9e9fb306913.png" width="400px">

きれいに描画されるようになりました！

このようにきれいに描画するには、想定の線幅に応じてその半分以上の大きさのバッファーを設定しておく必要があります。

### ポリゴン（Polygon）の場合

次に、ポリゴンについてみてみたいと思います。ポリゴンは、建物データで試してみます。

ラインの場合は、線幅に比例して欠落が発生しますが、ポリゴンの場合は、少なくともMapLibre GLのスタイリングでは、外周線の線幅は設定できません。

そのため、MapLibre GLを使用している限りにおいては、ほとんどbufferが問題になることはありません。

実際にbuffer=0のタイルで表示してみた例は以下のとおりです（グレーで塗られているのが建物で、外周線アリで描画しています。）。

- MapLibre GL JS(buffer=0)

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/1772652b-beed-4016-b2b9-40808023542f.png" width="400px">

- TileServer GL(buffer=0)

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/3680bb05-420d-4013-8208-0815c43972f0.png" width="400px">

肉眼で確認する限り、特におかしな点は見られません。

試しにbuffer=2で作成したタイルも表示してみます。

- MapLibre GL JS(buffer=2)

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/f139e0ed-157b-4c50-8f55-946b37e13beb.png" width="400px">

- TileServer GL(buffer=2)

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/e38fd0bd-be0f-4f66-bf65-7ee1e705a1b7.png" width="400px">

特に見た目に変化はありません。

通常、ポリゴンの外周線を自由度高くスタイリングできるようにする場合は、別途ラインを用意する場合が多いと思われるので、ポリゴンについてはbufferの設定値は最低限で問題ないと思われます。

### ポイント（Point）の場合

最後に、ポイント（注記ラベル）の場合を見てみたいと思います。

これはラインやポリゴン形状の描画とは考慮すべきポイントが異なります。

実際にbuffer=0のタイルを表示させてみます。使用しているのはOverture MapsのPlace labelです。

- MapLibre GL JS(buffer=0)

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/6e8966fb-0828-4965-9ea5-e5543b17c92c.png" width="400px">

- TileServer GL(buffer=0)

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/659a8055-5b79-4aa1-85c3-6b3d92e00751.png" width="400px">

MapLibre GL JSでは、すべてのラベルが問題なく表示されていますが、TileServer GLではタイル境界でラベルが見切れてしまっています。

これは、注記のオーバーラップを防ぐための処理が原因と思われます（MapLibre GLのスタイリングでは注記のオーバーラップを許容する設定もできますが、デフォルトではfalseになっている（かつ許容しないのが通常である）ため、許容しない場合で考えます）。

MapLibre GL JSでは、クライアント側でレンダリングするため、画面の表示範囲や向きによって、動的に表示できる注記を変えればよいので、タイル横断で注記配置を最適化することができます。

一方で、TileServer GLでは、サーバー側でレンダリングして、クライアントには静的なラスタタイルとして配信するため、常に同じ注記配置でレンダリングしておく必要があります。

そのため、各タイルは自身のタイルデータに含まれている注記のみで描画し、上記のラインやポリゴンで見てきた挙動と同様にタイル境界で切り落とされているものが使われていると推察されます。

したがって、タイル境界で注記が欠けてしまう事象を防ぐためには、最大の文字列長とスタイリングで設定している文字の大きさを考慮して、適切なバッファーを設定する必要があります。

tippecanoeで設定できる最大のbuffer値（=127）で作成したタイルでは、以下のようになります。

- MapLibre GL JS(buffer=127)

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/43a6bc9d-dbea-4fa3-9509-4ce0ba871941.png" width="400px">

- TileServer GL(buffer=127)

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/1154b1eb-0913-41ee-9a02-aab20136fc11.png" width="400px">

TileServer GLでも、注記が欠けることなく表示されるようになりました！

このように、バッファーを設定していないと、TileServer GLなどの個々のタイルが自身タイルに含まれる注記データのみで描画されるレンダリングエンジンでは、有用性が大きく損なわれます。

以上から、データ容量に問題がなければ、含まれる注記の文字列長に応じて、64-127くらいの大きめのbufferで作成しておくのが無難と思われます。

## 本家のMapboxはどのように設定しているか

ここで、MVTの生みの親であるMapboxがどのようにバッファーを設定しているかを見てみたいと思います。

Mapboxが提供しているベクタータイルのタイルセットリファレンスに、各レイヤで設定されているバッファー値の記載があります。例えばStreets v8タイルセットについては、以下のリンクから参照できます。

- [Mapbox Streets v8 - Layer reference](https://docs.mapbox.com/ja/data/tilesets/reference/mapbox-streets-v8/#layer-reference)

本記事執筆時点（2025/12/22）のバッファーの設定値は以下のようになっています。

| レイヤ名 | ジオメトリタイプ | バッファー値 |
|---------|----------------|-------------|
| admin | ライン | 4 |
| aeroway | ライン | 4 |
| road | ポイント, ライン, ポリゴン | 4 |
| waterway | ライン | 4 |
| building | ポリゴン | 2 |
| structure | ポイント, ライン, ポリゴン | 4 |
| landuse_overlay | ポリゴン | 8 |
| landuse | ポリゴン | 4 |
| water | ポリゴン | 8 |
| motorway_junction | ポイント | 8 |
| airport_label | ポイント | 64 |
| housenum_label | ポイント | 64 |
| natural_label | ポイント, ライン | 64 |
| place_label | ポイント | 128 |
| poi_label | ポイント | 64 |
| transit_stop_label | ポイント | 64 |

ラベルレイヤ（注記）には**64-128という大きめのバッファー**が設定されており、ジオメトリレイヤ（ライン・ポリゴン）には**2-8という比較的小さめのバッファー**が設定されていることがわかります。

細かな設定値の差異については、議論の余地があるかと思いますが、おおよその傾向として、本稿の考察結果と矛盾はしていないかと思います（と信じています）。

## まとめ

ジオメトリタイプごとに、バッファーの主な役割と設定すべきバッファーの値をまとめると以下のようになります。

1. **ライン**
   - 役割：クリッピングによるタイル境界付近でのジオメトリの欠落を防ぐ。
   - 設定値：欠落幅の最大値は線幅の半分になるため、**線幅の半分以上**の値を設定しておくのが理想（例：**10ピクセルで描画するのであれば5以上**）。

2. **ポリゴン**
   - 役割：クリッピングによるタイル境界付近でのジオメトリ（外周線）の欠落を防ぐ。
   - 設定値：外周線を太く描画できるレンダリングエンジンを使う場合は、ラインと同様の留意が必要だが、自由なスタイリングを許容数場合、通常は別途ラインで提供することになると思われるため、基本的には**最低限のバッファー（2とか）** で問題ないと思われる。

3. **ポイント（注記）**
   - 役割：タイル個別にレンダリングする描画エンジン（例：TileServer GL）を使う場合に、タイル境界でラベルが見切れるのを防ぐ。
   - 設定値：文字列長を踏まえつつ、**大きめに設定（64-127くらい）** しておくのが適当。

このほか、バッファー値を設定する際は、UXを損なわないようにタイル容量も考慮する必要があります。当然ですが、バッファーの設定値が大きくなるほどタイル容量が大きくなります。

ベクタータイルの場合、一般的に**平均で50KB未満、最大でも500KBまで**に抑えるのが適切とされています。興味のある方は、少々古い記事ですが、以下を参照ください。

- [A data-driven journey through Vector Tile optimization](https://medium.com/@ibesora/a-data-driven-journey-through-vector-tile-optimization-4a1dbd4f3a27)

以上になります。ここまで読んでくださりありがとうございました！
少しでも皆さんがMVTを作成する際の参考になっていれば幸いです。