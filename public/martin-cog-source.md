---
title: Martinを使ったCOGのタイル配信
tags:
  - cog
  - Martin
private: false
updated_at: '2026-10-30T21:49:48+09:00'
id: 76c22925efff6de93d43
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
# はじめに

[Martin 2.0.0-beta.4](https://maplibre.org/martin/changelog/#200-beta4-2026-10-22)からCOG sourcesがstableになり、デフォルトビルドに搭載されました。

COGは、Cloud-Optimized GeoTiffのことで、クラウド環境でGeoTIFF画像の一部を容易に取り出せるように内部のデータ配置が最適化されたGeoTiffです。

Martinでは、COGからタイルデータを取得してXYZタイルを配信することができます。

本記事では、その方法をご紹介したいと思います。

# Sentinel-2 衛星画像のダウンロード

データサンプルとして、Sentinel-2の衛星画像を使います。

AWSのOpen Data Registryで提供されています。

https://registry.opendata.aws/sentinel-2-l2a-cogs/

画像は大量にあるので、Earth SearchというSTAC APIでアーカイブを検索します。

東京周辺の雲量10%未満のvisual画像（RGB合成済み）を以下のPythonスクリプトでダウンロードします。

```python
from pystac_client import Client
import requests

catalog = Client.open(
    "https://earth-search.aws.element84.com/v1"
)

search = catalog.search(
    collections=["sentinel-2-c1-l2a"],
    bbox=[139.65, 35.55, 139.90, 35.80],
    datetime="2026-09-01/2026-10-20",
    query={"eo:cloud_cover": {"lt": 10}},
    max_items=1
)

items = list(search.items())

if not items:
    raise RuntimeError("画像が見つかりません")

item = items[0]

print("ID:", item.id)
print("Date:", item.datetime)
print("Cloud:", item.properties.get("eo:cloud_cover"))
print("Assets:", list(item.assets.keys()))
print()

asset = item.assets.get("visual")
if asset is None:
    raise RuntimeError("visualアセットがありません")

url = asset.href
print("Downloading:", url)

with requests.get(url, stream=True, timeout=60) as response:
    response.raise_for_status()
    with open("sentinel2.tif", "wb") as f:
        for chunk in response.iter_content(
            chunk_size=1024 * 1024
        ):
            if chunk:
                f.write(chunk)
```

# WebMercatorに投影変換

ダウンロードした画像を[gdalwarp](https://gdal.org/en/stable/programs/gdalwarp.html)でWebMercatorに投影変換します。

黒のNodataは透過にしておきたいので、`-dstalpha` オプションを設定しておきます。

```bash
gdalwarp \
  -t_srs EPSG:3857 \
  -r bilinear \
  -dstalpha \
  -of GTiff \
  -co TILED=YES \
  -co COMPRESS=DEFLATE \
  sentinel2.tif \
  sentinel2_rgba.tif
```

# COGに変換

Martinが求めるCOGの要件は[こちら](https://maplibre.org/martin/sources-cog-files/#file-requirements)に記載があります。

画像はWebMercator投影になっており、かつフル解像度画像及びオーバービュー画像はすべてXYZタイルグリッドでタイル化されている必要があります。

そのため、[gdal_translate](https://gdal.org/en/stable/programs/gdal_translate.html)で、COGに変換しつつ、`TILING_SCHEME` と `ALIGNED_LEVELS` オプションでタイル境界をXYZタイルグリッドに整合させます（これをしないとMartinに配信対象から除外されてしまいます）。

```bash
gdal_translate \
  -of COG \
  -co TILING_SCHEME=GoogleMapsCompatible \
  -co ALIGNED_LEVELS=7 \
  -co BLOCKSIZE=256 \
  -co COMPRESS=DEFLATE \
  -co OVERVIEWS=IGNORE_EXISTING \
  sentinel2_rgba.tif \
  cog/sentinel2_aligned_rgba.tif
```

# Martinの設定ファイル作成

配信に使うCOG画像ができたので、あとはMartinで配信するだけです。

Martinの設定ファイルを作成します。

- config.yaml

```yaml
cog:
  sources:
    sentinel2: /cog/sentinel2_martin.tif
```

# Martinを起動

martin:2.0.0-beta.4イメージを使ってDockerでMartinを起動します。

```bash
docker run --rm \
  -p 3000:3000 \
  -v "$(pwd)/cog:/cog:ro" \
  -v "$(pwd)/config.yaml:/config.yaml:ro" \
  ghcr.io/maplibre/martin:2.0.0-beta.4 \
  --config /config.yaml
```

# TileJSONを確認

意図どおり配信設定されているか、以下のURLでTileJSONを確認します。

```bash
curl -s http://localhost:3000/sentinel2 | jq .
```

```json
{
  "tilejson": "3.0.0",
  "tiles": [
    "http://localhost:3000/sentinel2/{z}/{x}/{y}"
  ],
  "bounds": [
    137.81250000000003,
    34.307143856288036,
    140.62500000000003,
    36.5978891330702
  ],
  "center": [
    139.21875000000003,
    35.4606699514953,
    10
  ],
  "maxzoom": 14,
  "minzoom": 7,
  "format": "png",
  "tileSize": 256
}
```

# MapLibreで表示

最後にMapLibre GL JSで表示してみます。

OpenStreetMapを背景地図として、重ね合わせて表示してみます。

![スクリーンショット 2026-10-30 213655.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/6d8f5622-4f3a-47ff-8051-34d8b2d1c04f.png)

低ズームレベルのタイル含め問題なく表示されることが確認できました。

# まとめ

- Martin2.0では標準機能としてCOG配信することが可能
- ただし、COGはあらかじめXYZタイルグリッドにあわせてタイル化されている必要がある
