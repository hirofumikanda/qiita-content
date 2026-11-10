---
title: MapLibre で日本語フォントを glyph サーバから供給する
tags:
  - MapLibre
  - Glyph
  - font
private: false
updated_at: '2026-11-10T22:43:35+09:00'
id: b32190a1170da789b822
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
# はじめに

MapLibre GL JS では、日本語などの CJK 文字はデフォルトではサンセリフ体のローカルフォントが使われます。

styleJSON にグリフサーバーが設定されていてもそちらは参照されません。

本記事では、日本語テキストもグリフサーバーを参照するようにする方法とその際の留意点について説明します。

# localIdeographFontFamily オプション

マップの `localIdeographFontFamily` オプションで CJK 文字で参照するフォントの挙動を変更することができます。

https://maplibre.org/maplibre-gl-js/docs/API/type-aliases/MapOptions/#localideographfontfamily

デフォルト値は `sans-serif` となっており、ローカルフォントのサンセリフ体を利用します。

グリフサーバーを参照するようにするには、このオプションを `false` にする必要があります。

```javascript
localIdeographFontFamily: false,
```

これで日本語もグリフサーバーを参照するようになりますが、これ以外にも日本語を扱う上で気を付けなければならないポイントがあります。

# BMP 外のコードポイントの文字を使う場合

ユニコードは、複数の面（プレーン）から構成されています。

1つのプレーンには、65,536 個のコードポイントが含まれます。

https://ja.wikipedia.org/wiki/Unicode%E4%B8%80%E8%A6%A7%E8%A1%A8

その中でも最初の第 0 面を基本多言語面（BMP / Basic Multilingual Plane）と呼び、主要な日常言語の文字が収録されています。

[maplibre / font-maker](https://github.com/maplibre/font-maker) などで、グリフを作成すると、BMP 範囲のコードポイントのグリフしか出力されません。

そのため、BMP 範囲外の文字もグリフサーバーから供給するには、BMP の範囲を超えてグリフを用意しておく必要があります。

実際に一部の地名や人名で使われている漢字（𠮷（つちよし）など）は、BMP 範囲外に該当します。

# BMP 外の文字も含めてグリフデータを作成

font-maker のソースを少し改変すれば、BMP 範囲に限定せずにグリフデータを作成することができます。

ここでは、Noto Sans JP Regualr で試します。

Google Fonts から TTF をダウンロードします。

https://fonts.google.com/selection?query=Noto+Sans

まず、以下の python スクリプトでフォントデータに収録されているユニコード範囲を抽出します。

```python
from fontTools.ttLib import TTFont

font = TTFont("NotoSansJP-Regular.ttf")
cmap = font.getBestCmap()

# BMP: 全256範囲
bmp_ranges = set(range(0, 0x10000, 256))

# BMP外: フォントに収録されている範囲のみ
non_bmp_ranges = {
    (cp // 256) * 256
    for cp in cmap
    if cp > 0xFFFF
}

# 統合
ranges = sorted(bmp_ranges | non_bmp_ranges)

with open("ranges.txt", "w", encoding="utf-8") as f:
    for start in ranges:
        f.write(f"{start}\n")

print(f"BMP ranges: {len(bmp_ranges)}")
print(f"Non-BMP ranges: {len(non_bmp_ranges)}")
print(f"Total ranges: {len(ranges)}")
```

`ranges.txt` が出力されます。

```
0
256
512
768
...
65024
65280
127232
127488
131072
...
200704
```

# font maker で BMP に限定せずにグリフデータを出力

さきほど作成した `ranges.txt` にしたがって、グリフデータを出力するように font-maker のソースを変更します。

git clone します。

```bash
git clone --recursive \
  https://github.com/maplibre/font-maker.git
```

**main.cpp** に

```c++
#include <fstream>
#include <cstdint>
```

を追加した上で、

```c++
for (int i = 0; i < 65536; i += 256) {
```

を以下に変更します。

```c++
std::ifstream range_file("ranges.txt");

if (!range_file) {
    std::cerr << "Cannot open ranges.txt" << std::endl;
    return 1;
}

uint32_t i;

while (range_file >> i) {
```

その上でビルドします。

```bash
sudo apt update

sudo apt install -y \
  cmake \
  build-essential \
  libfreetype-dev \
  libboost-dev

cmake -S . -B build
cmake --build build -j$(nproc)
```

ビルドが成功したら、それを使ってグリフデータを生成します。

```bash
font-maker/build/font-maker \
  --name "Noto Sans JP Regular" \
  fonts \
  NotoSansJP-Regular.ttf
```

fonts/Noto Sans JP Regular 配下にグリフデータが出力されます。

# MapLibre で表示

MapLibre でBMP外の文字も含めて表示されるか試してみます。

小字（こあざ）の名称には、BMP の範囲外の文字が使われているケースがいくつかあるため、町字のデータでテストします。

[Geolonia japanese address v2 API](https://github.com/geolonia/japanese-addresses-v2) で、青森県東北町のデータを取得し、「字仏沢」を正式名称の「字𠏹沢」に変更します。

「𠏹」はユニコード U+203F9 の文字で、10進数にすると 132,089 であるため、生成したグリフの中の 131840-132095.pbf に含まれます。

実際に表示してみたのが以下のサイトです（クリックするとリンク先に飛びます）。

<a href="https://hirofumikanda.github.io/maplibre-non-bmp-glyphs/" target="_blank">
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/0a40066c-5823-4417-83d1-c42779b19d47.png" alt="maplibre" />
</a>

冒頭ご紹介したとおり、map オブジェクト生成時に

```javascript
localIdeographFontFamily: false,
```

としています。

ブラウザの開発者ツールのネットワークタブを確認すると、131840-132095.pbf がグリフサーバにリクエストされていることがわかります。

![networkタブ](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3924659/4a03a057-53da-4ced-a375-0b20b5fdd38b.png)

# まとめ

- MapLibre で CJK フォントを使うとき、グリフサーバを利用するには map オブジェクトのオプションで `localIdeographFontFamily: false` とする必要がある。
- また、一部の小字地名などでユニコードの BMP 範囲外の文字が使われていることがあるため、利用するグリフデータにその範囲の文字が提供されているか確認しておく必要がある。
- 自前でグリフデータを用意する場合は、必要に応じて BMP 外のグリフも作成する。fonts-maker では、そのままだと BMP 範囲のグリフしか作成されないが、一部ソースを変更することで任意の範囲のグリフを作成することが可能である。
