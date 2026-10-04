---
title: Martinを用いたサーバサイドレンダリング
tags:
  - Martin
  - raster
  - MVT
private: false
updated_at: '2026-10-04T16:39:22+09:00'
id: 732fb44bde7ba499be84
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
# はじめに

[Martin](https://maplibre.org/martin/)は、地図タイルを高速に配信するためのRust製のタイル配信サーバです。

PostGISをはじめ、様々なデータソースからタイルデータを抽出して配信することが可能ですが、それに加えて[MapLibre Style Spec](https://maplibre.org/maplibre-style-spec/)に準拠したStyle JSONをレンダリングしてラスタタイルとして配信することも可能です。

2026/9/29にリリースされたv2.0のベータ版では、このサーバサイドレンダリング機能が安定版として提供されるようになりました（[Sever-side rendering is stable](https://github.com/maplibre/martin/blob/main/martin/CHANGELOG.md#server-side-rendering-is-stable)）。

そこで、本記事では、この機能の使い方を簡単に紹介したいと思います。

# Style JSONを用意

サーバサイドレンダリングのテスト用に[Protomaps](https://protomaps.com/)が提供しているオープンデータを使います。

basemap用のViewerが[maps.protomaps.com](https://maps.protomaps.com/)で提供されています。

**Get style JSON** ボタンからこのViewerに適用されているStyle JSONを入手します。

<a href="https://maps.protomaps.com/" target="_blank">
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/22ee3ec0-28ad-424f-a64f-00d2ddc92a57.png" alt="protomaps" />
</a>

# Rendering対応版のMartinを取得

Martinドキュメントの[Server-side raster tile rendering](https://maplibre.org/martin/sources-styles/rendering/)によると、レンダリング機能はデフォルトビルドには含まれていません。

renderingフィーチャを有効化してソースビルドするか、`-full`タグ付きのDockerイメージを使用する必要がありますが、今回は簡単に`-full`タグ付きのDockerイメージを使います。

GitHub Packagesの[Martin](https://github.com/maplibre/martin/pkgs/container/martin)をみると**2.0.0-beta.2-full**タグがあるので、これをpullします。

```bash
docker pull ghcr.io/maplibre/martin:2.0.0-beta.2-full
```

# config.yamlを作成

Martinの設定ファイル（`config.yaml`）を作成します。

styleソースの書き方は[Style Sources](https://maplibre.org/martin/sources-styles/)に記載があります。

また、[Server-side raster tile rendering](https://maplibre.org/martin/sources-styles/rendering/)に、レンダリングの有効化方法が書いてあります。

これらに従うと以下のようになります。

```yaml
styles:
  rendering: true
  sources:
    basemap: /styles/style.json
```

# コンテナを実行

準備が整いましたので、Martinコンテナを実行します。

```bash
docker run --rm \
  -p 3000:3000 \
  -v "$(pwd)/styles:/styles" \
  -v "$(pwd)/config.yaml:/config.yaml" \
  ghcr.io/maplibre/martin:2.0.0-beta.2-full \
  --config /config.yaml
```

以下のように出力されれば問題ありません。

```
2026-10-04T06:55:58.939506Z  INFO martin: Starting Martin v2.0.0-beta.2
2026-10-04T06:55:58.940829Z  INFO martin: Using /config.yaml
2026-10-04T06:55:58.983311Z  INFO resolve: martin::config::file::cache: Initializing PMTiles directory cache with maximum size 128 MB
2026-10-04T06:55:58.984854Z  INFO resolve: martin::config::file::cache: Initializing tile cache with maximum size 256 MB
2026-10-04T06:55:58.985175Z  INFO resolve: martin::config::file::cache: Initializing sprite cache with maximum size 64 MB
2026-10-04T06:55:58.986348Z  INFO resolve: martin::config::file::cache: Initializing font cache with maximum size 64 MB
2026-10-04T06:55:58.989658Z  INFO resolve: martin_core::resources::styles::render_pool: Started style render pool workers=4 kind="tile"
2026-10-04T06:55:58.991270Z  INFO resolve: martin_core::resources::styles::render_pool: Started style render pool workers=4 kind="static"
2026-10-04T06:55:58.992125Z  INFO resolve: martin_core::resources::styles: Configured style source source.id=basemap style.path=/styles/style.json
2026-10-04T06:55:58.993711Z  INFO martin: Use --save-config to save or print Martin configuration.
2026-10-04T06:55:59.013543Z  INFO martin::config::file::cors: CORS enabled with defaults (origin=["*"], max_age=None)
2026-10-04T06:55:59.017317Z  INFO martin: Martin server is now active at http://0.0.0.0:3000/
2026-10-04T06:55:59.017480Z  INFO martin: Web UI is only served to localhost connections. Use `--webui enable-for-all` in CLI or a config value to enable it for all connections
```

# QGISで表示

[Rendered XYZ tiles](/style/<style_id>/{z}/{x}/{y}.{filetype})には、タイルリクエストのエンドポイントは
`/style/<style_id>/{z}/{x}/{y}.{filetype}`
になると記載があるので、今回の場合は
`http://localhost:3000/style/basemap/{z}/{x}/{y}.png`
となります。

これをQGISのXYZ接続で設定します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/642e4fb8-0efa-4507-89df-6097bfcb905d.png" alt="qgis_settings" />

これをレイヤ追加すると地図が表示されます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/a47fb618-d00c-4639-b87c-9f7841f2e82b.png" alt="qgis" />

# まとめ

サーバサイドレンダリングのOSSとしては、MapTilerの[tileserver-gl](https://github.com/maptiler/tileserver-gl)などがありますが、Martinでもv2.0からStable機能としてサーバサイドレンダリングが可能になりました。

これにより様々なクライアントに対応したタイル配信サーバを、集約して構築する手段が一つ増えたといえるのではないかと思います。
