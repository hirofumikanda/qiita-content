---
title: 地図タイルAPIがクエリ文字列でアクセストークンを使う技術的理由
tags:
  - API
  - Security
  - CORS
  - パフォーマンス
  - 地図タイル
private: false
updated_at: '2025-11-23T15:46:47+09:00'
id: 9e4e301740c1f8f5ac0f
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに

Mapbox Vector Tile APIやGoogle Map Tile APIなど、主要な地図タイルAPIの仕様を見ると、APIキーやアクセストークンをクエリ文字列（URLパラメータ）に設定する方式が採用されています。

```
https://api.mapbox.com/v4/mapbox.mapbox-streets-v8/1/0/0.mvt?access_token=YOUR_TOKEN
```

一般的に、APIキーやアクセストークンをクエリ文字列に含めることは、セキュリティのベストプラクティスに反するとされています。通常は`Authorization`ヘッダーや署名付きURLなどを使用すべきとされる中、なぜ地図タイルAPIはクエリ文字列方式を採用しているのでしょうか？

本記事では、地図タイルAPIの特殊な要件を考慮しつつ、その技術的要因を調査・考察してみたいと思います。
**なお、あくまで私見であり、いかなる地図APIプロバイダーの公式見解ではありません。**

# 地図タイルAPIの特性

まず、地図タイルAPIの特性を理解することが重要と考えます。

## 大量のリクエストが発生する

地図表示では、ズームレベルや表示範囲、パン・ズーム操作に応じて**数十〜数百のタイルリクエスト**が同時に発生します。例えば：

- 画面全体を表示する場合、少なくとも20タイル前後が必要
- ユーザーがパンやズームする度に新しいタイルをリクエスト
- スムーズな地図操作には、これらのリクエストが**低レイテンシ**で処理する必要がある

## `<img>`タグや`<canvas>`からの直接リクエスト

多くの地図ライブラリは、パフォーマンス最適化のため、タイル画像を`<img>`要素や`fetch()`で直接取得します。この際、カスタムヘッダーを付与すると、クロスオリジンの場合、後述するCORSプリフライトリクエストが発生します。

# クエリ文字列を使う技術的理由

## 1. CORSプリフライトリクエストの回避

### プリフライトリクエストとは

[CORS（Cross-Origin Resource Sharing）](https://developer.mozilla.org/ja/docs/Web/HTTP/CORS)において、特定の条件を満たさないリクエストは、ブラウザが本リクエストの前に**OPTIONSメソッドのプリフライトリクエスト**を自動的に送信し、そのリクエストがサーバーで許可されているかを確認します。地図タイルをリクエストする場合、以下のようなリクエストが発生します。

1. クライアント → サーバー: OPTIONS /tile/1/0/0.mvt (プリフライト)
2. サーバー → クライアント: 204 No Content (Access-Control-Allow-*ヘッダー)
3. クライアント → サーバー: GET /tile/1/0/0.mvt (本リクエスト)
4. サーバー → クライアント: 200 OK (タイルデータ)

つまり、リクエスト数が通常の2倍になります。各プリフライトリクエストはネットワークラウンドトリップを追加するため、**レイテンシが累積**し、ユーザー体験を著しく悪化させます。

### Simple Request（単純リクエスト）の条件

プリフライトリクエストを回避するには、[Simple Request](https://developer.mozilla.org/ja/docs/Web/HTTP/CORS#%E5%8D%98%E7%B4%94%E3%83%AA%E3%82%AF%E3%82%A8%E3%82%B9%E3%83%88)の条件を満たす必要があります。

- メソッドが`GET`, `HEAD`, `POST`のいずれか
- カスタムヘッダーを使用しない（`Accept`, `Accept-Language`, `Content-Language`, `Content-Type`のみ許可）
- `Content-Type`が`application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain`のいずれか

**クエリ文字列にトークンを含める方式は、この条件を満たすことができます。**

なお、プリフライトレスポンスは`Access-Control-Max-Age`ヘッダーでキャッシュできます。

```http
Access-Control-Max-Age: 86400  # 24時間
```

しかし、ブラウザごとに最大値の制限があり、また初回アクセス時のオーバーヘッドは避けられません。地図アプリケーションでは、初回表示速度が重要なため、この遅延は許容しづらいと考えられます。

## 2. クライアント実装の簡素化

### ネイティブAPIとの互換性

クエリ文字列方式を採用することで、以下のような簡単な実装が可能になります。

```javascript
// シンプルなimg要素での読み込み
const img = new Image();
img.src = `https://api.example.com/tiles/1/0/0.png?token=${accessToken}`;

// Authorizationヘッダーを使う場合（プリフライトが発生）
const response = await fetch('https://api.example.com/tiles/1/0/0.png', {
  headers: {
    'Authorization': `Bearer ${accessToken}`
  }
});
```

`<img>`要素では`Authorization`ヘッダーを設定できないため、クエリ文字列方式が必須となります。

### ライブラリとの統合

Leaflet、OpenLayers、MapLibre GL JSなどの地図ライブラリは、タイルURLテンプレートを使用します。

```javascript
// MapLibre GL JSの例
const map = new maplibregl.Map({
  container: 'map',
  style: {
    version: 8,
    sources: {
      'raster-tiles': {
        type: 'raster',
        tiles: ['https://api.example.com/tiles/{z}/{x}/{y}.png?token=YOUR_TOKEN'],
        tileSize: 256
      }
    },
    layers: [{
      id: 'simple-tiles',
      type: 'raster',
      source: 'raster-tiles'
    }]
  }
});
```

この方式により、ライブラリ側でカスタムヘッダー処理の実装が不要になり、開発者の学習コストも下がります。

# セキュリティ上の懸念と対策

クエリ文字列方式には確かにセキュリティリスクが存在します。

## 主なリスク

1. **URLログへの記録**: サーバーログ、プロキシログ、ブラウザ履歴にトークンが残る
2. **ソースコードへの埋め込み**: フロントエンドコードからトークンが容易に確認できる

## 地図タイルAPIでのリスク軽減策

そのため、実際に提供されている地図タイルAPIでは、以下のようなリスク軽減策がとられています。

### 1. トークンの権限スコープ制限

トークンに以下のような **スコープ（権限）** を設定できるようにしています。

```
パブリックトークン（読み取り専用）:
- styles:read
- fonts:read
- tiles:read

シークレットトークン（書き込み権限）:
- uploads:write
- styles:write
```

これにより、**地図表示には読み取り専用トークンのみを使用**し、データアップロードなどの機密操作はサーバーサイドで処理することなどが可能になります。

### 2. URL制限

トークンに対して、使用可能なドメインを制限できるようにしています。

```
許可URL: https://example.com/*
```

これにより、トークンが漏洩しても、他のドメインでは使用できないようすることも可能です。

### 3. トークンのローテーション

トークンに有効期限を設定し、定期的にトークンを更新することで、漏洩リスクを時間的に制限しています。

### 4. レート制限とモニタリング

異常なアクセスパターンを検出し、不正使用を防止する制限が設けられています。

```
# 例: 1分間に100,000リクエストの制限（Mapboxのデフォルト）
```

また、ダッシュボードなどで、 **トークン別の使用統計**を確認できるようにすることで、予期しない使用を検出することを可能にしています。

# 地図タイルAPIに求められるセキュリティの特殊性

そのほか、セキュリティを検討する上で、地図タイルの特性を踏まえる必要があります。地図タイルは**基本的にパブリックデータ**であり、トークンの目的は以下のようなものが挙げられます。

- **課金管理**: どのユーザーが使用したかの追跡
- **レート制限**: 過度な使用の防止
- **利用規約の適用**: URL制限などのポリシー実施

機密データの保護が目的ではないことがほとんどであるため、一般的なAPIとはセキュリティ要件が異なると考えられます。

# 他の認証方法との比較

## 1. Authorizationヘッダーを使用する場合

```javascript
// MapLibre GL JSでカスタムヘッダーを使用する場合
const map = new maplibregl.Map({
  container: 'map',
  style: {
    version: 8,
    sources: {
      'raster-tiles': {
        type: 'raster',
        tiles: ['https://api.example.com/tiles/{z}/{x}/{y}.png'],
        tileSize: 256
      }
    },
    layers: [{
      id: 'simple-tiles',
      type: 'raster',
      source: 'raster-tiles'
    }]
  },
  transformRequest: (url, resourceType) => {
    if (resourceType === 'Tile') {
      return {
        url: url,
        headers: { 'Authorization': `Bearer ${token}` }
      };
    }
  }
});
```

**メリット**:
- セキュリティベストプラクティスに準拠
- URLログに記録されない

**デメリット**:
- CORSプリフライトリクエストが発生（2倍のリクエスト数）
- `<img>`要素で使用不可
- 実装が複雑化（上記の例ではtransformRequestの実装が必要）

そのため、地図タイルのようなパフォーマンス重視のユースケースには適合しないと考えられます。

## 2. 署名付きURL（Signed URL）を使用する場合

**メリット**:
- 時間制限付きアクセス
- プリフライトリクエストなし
- トークンの代わりに使い捨て署名

**デメリット**:
- サーバーでの署名生成が必要
- CDNキャッシュ効率が低下（URL毎に異なる）
- 実装の複雑性

高セキュリティが必要なケースには有効ですが、一般的な地図表示にはやや過剰と考えられますし、URL仕様によってはキャッシュ効率を低下させる可能性があります。

# まとめ

地図タイルAPIがクエリ文字列でアクセストークンを使用する理由は、以下の技術的要件によるものと考えられます。

## 1. パフォーマンスの最適化
- **CORSプリフライトリクエストの回避**が最大の理由
- 数百のタイルリクエストで、プリフライトがあるとリクエスト数が2倍になり、レイテンシが著しく増加
- Simple Requestの条件を満たすことで、ブラウザの追加オーバーヘッドを排除

## 2. 実装の簡素化
- `<img>`要素やネイティブAPIとの互換性
- 既存の地図ライブラリとの統合が容易
- 開発者の学習コストが低い

## 3. セキュリティとのバランス

地図タイルAPIは、一般的なREST APIとは異なる特性を持ちます。

- **パブリックデータ**が前提（機密情報ではない）
- トークンの目的は**課金管理とレート制限**
- **読み取り専用スコープ**により被害を限定
- **URL制限**と**モニタリング**で不正使用を防止

これらの対策により、クエリ文字列方式の利便性を享受しつつ、セキュリティリスクを許容可能なレベルに低減しているものと考えられます。

# 参考資料

- [Mapbox Vector Tiles API](https://docs.mapbox.com/api/maps/vector-tiles/)
- [How to use Mapbox securely](https://docs.mapbox.com/help/troubleshooting/how-to-use-mapbox-securely/)
- [MDN - Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/ja/docs/Web/HTTP/CORS)
- [MDN - Preflight request](https://developer.mozilla.org/ja/docs/Glossary/Preflight_request)
- [Google Maps Platform - API Key Best Practices](https://developers.google.com/maps/api-security-best-practices)
