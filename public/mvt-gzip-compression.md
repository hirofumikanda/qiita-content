---
title: MVTの圧縮方式について
tags:
  - MVT
  - mapbox
  - gzip
private: false
updated_at: '2026-08-13T23:01:00+09:00'
id: e7c3ccd913047913398f
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

# Mapbox Vector Tileはgzip圧縮で配信されている

[Mapbox Vector Tiles API](https://docs.mapbox.com/api/maps/vector-tiles/)を使ってベクタタイルをリクエストすると、クライアント側の `Accept-Encoding` の設定値にかかわらずgzip圧縮でレスポンスされます。

実際に、curlでリクエストすると、以下のとおりレスポンスが返ってきます（access_tokenは伏字）。

```bash
# Accept-Encodingヘッダーなし
curl -I "https://api.mapbox.com/v4/mapbox.mapbox-streets-v8/1/0/0.mvt?access_token=###"

# gzip圧縮
curl -I -H "Accept-Encoding: gzip" "https://api.mapbox.com/v4/mapbox.mapbox-streets-v8/1/0/0.mvt?access_token=###"

# br圧縮
curl -I -H "Accept-Encoding: br" "https://api.mapbox.com/v4/mapbox.mapbox-streets-v8/1/0/0.mvt?access_token=###"

# identity
curl -I -H "Accept-Encoding: identity" "https://api.mapbox.com/v4/mapbox.mapbox-streets-v8/1/0/0.mvt?access_token=###"
```

いずれも以下のレスポンスが返ってきます。

```bash
HTTP/2 200
content-type: application/vnd.mapbox-vector-tile
content-length: 40573
content-encoding: gzip
```

一律、`content-encoding` はgzipになっており、 `accept-encoding` に合わせて圧縮方式を変えるような仕様にはなっていません。

# なぜgzip固定の圧縮方式になっているのか

通常のウェブコンテンツでは、クライアントの `accept-encoding` に合わせた圧縮方式でレスポンスされることが多いですが、なぜMVTではそのような仕組みになっていないのでしょうか。

MVTのようなクライアントから大量のリクエストが届くことが想定されるデータの場合、転送効率をよりシビアに考慮する必要があります。

そのため、圧縮方式に柔軟性をもたせて多様なクライアントに対応するよりも、gzip固定にすることによる性能上の利点が優先されたのではないかと思われます。

性能上の利点としては、以下が挙げられるかと思います。

## 性能上の利点① データソース側で事前にデータを圧縮しておくことができる

圧縮方式が固定で決まっていれば、あらかじめ静的タイルを圧縮してデプロイしておくことができます。
MVT生成ツールである `tippecanoe` でも、gzip圧縮での出力がデフォルトになっています。

圧縮してデータを作成しておくことで、以下の利点があると考えられます。

- ストレージ容量を削減できる
- 圧縮によるレイテンシーを削減できる
- （CDN層などで圧縮するわけではないので）クライアント-オリジン間で一貫してトラフィック量を削減できる

## 性能上の利点② キャッシュ効率がいい
また、圧縮方式を一通りにすることで、圧縮方式ごとにキャッシュを分ける必要がなくなるため、CDN層のキャッシュ保持を効率化できます。

キャッシュキーにヘッダーを含める必要がなく、URLパスのみをキャッシュキーとして保存できるため、ユーザ間でキャッシュを共有できます。

これにより、キャッシュヒット率が向上します。

# gzip固定で問題にならないのか

では、逆にgzip固定にして問題にはならないのでしょうか。

これについては、vector tile specのイシューに投稿された[以下のコメント](https://github.com/mapbox/vector-tile-spec/issues/27#issuecomment-66670565)が示唆を与えてくれます。

```
1. Vector tiles benefit from compression in transport and in storage.
2. We wish to offload decompression to the user agent in client side (GL rendering) scenarios.
3. We wish to avoid repetitive decompression/recompression when a VT travels through a tilelive pipeline.
4. gzip is the only reliable transport encoding for HTTP.

Together, these constraints led to the current solution: vector tiles are gzipped at rest, and served with Content-Encoding: gzip for HTTP transport.

---

1. ベクタータイルは、転送時および保存時の両方において、圧縮することでメリットがあります。
2. クライアントサイド（GLレンダリング）のシナリオでは、展開処理をユーザーエージェント（ブラウザなど）に任せたいと考えています。
3. ベクタータイル（VT）が tilelive パイプラインを通過する際に、展開と再圧縮が繰り返し行われることを避けたいと考えています。
4. HTTPにおいて、gzipは唯一、信頼して利用できる転送エンコーディングです。

これらの制約を総合した結果、現在の方式に至りました。つまり、ベクタータイルは保存時からgzip圧縮された状態にしておき、HTTPで転送する際には Content-Encoding: gzip を付けて配信するという方式です。
```

2014年のコメントですので、Mapboxでは少なくとも10年以上、gzipで圧縮された静的タイルを `content-type: gzip` で配信していることになります。

それだけ長期にわたり運用されている方式であり、現在に至るまで課題が顕在化していないところを踏まえると、大きな問題にはなっていないものと思われます。

以上から、MVTの配信で、圧縮方式について特段の要件がない場合は、ひとまずgzip固定にしておくのが妥当かと思われます。
