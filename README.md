# Interactive Globe Demo

3Dドットグローブ（自転・拠点点滅・ネットワーク線付き）のLP埋め込みデモ。

**デモURL**: https://globe-demo.vercel.app/
**ローカルプレビュー**: `index.html` をブラウザで直接開くか、`npx serve`

---

## LP への埋め込み方法

### 推奨: iframe + `loading="lazy"`

LP本体のJS／レンダリングに一切干渉せず、スクロールで近づいた時だけ初期化される。

```html
<div style="background: linear-gradient(135deg,#c9ccff,#7079e2); aspect-ratio:16/9;">
  <iframe
    src="https://globe-demo.vercel.app/"
    loading="lazy"
    style="width:100%; height:100%; border:0; display:block;"
    title="Global Network"
  ></iframe>
</div>
```

`aspect-ratio` は `16/9` 以外でも動く。レスポンシブ対応したい場合は親要素に幅100%＋高さを指定。

### インライン埋め込み（非推奨）

`<section class="yg-root">...</section>` 以下のマークアップ + `<style>` + `<script>` を直接ページに貼ることも可能。CSS は `.yg-*` 接頭辞でスコープ化してある。ただし Three.js が global に読み込まれるので、基本は iframe を推奨。

---

## コンテンツ仕様

- **大陸表現**: fibonacci 球面分布 × NASA topology 画像からの land mask で約6,000〜15,000個の白ドット
- **大陸の塗り**: 地形画像をスフィアテクスチャとして重ね、薄いパール色で陸地をうっすら塗る
- **拠点**: 世界の主要都市（東京・NY・ロンドン・シンガポール・シドニー・ドバイほか30都市）を ShaderMaterial で柔らかく点滅
- **ネットワーク線**: 拠点間をランダムに結ぶ Bezier アークが出現・消滅
- **緯線経線**: 白のワイヤーフレーム（緯度15°ごと、経度30°ごと）
- **リム光**: 外周に淡い大気光
- **地軸**: 23.4°傾けてある

---

## カスタマイズポイント

### 拠点の変更
`index.html` 内の `officesLatLng` 配列を編集。lat/lng の配列として渡すだけ。

```js
const officesLatLng = [
  { lat: 35.6762, lng: 139.6503 }, // Tokyo
  { lat: 40.7128, lng: -74.0060 }, // New York
  // ...
];
```

拠点数が大幅に変わる場合、パフォーマンスに影響する。

### 色のカスタマイズ

| 要素 | 箇所 | 変更方法 |
|---|---|---|
| 背景グラデーション | CSS `.yg-root background` | CSS で書き換え |
| 大陸ドットの色 | shader `gl_FragColor = vec4(1.0, 1.0, 1.0, a);` | buildLandPoints 内 |
| 大陸の塗り色 | shader `vec3 col = vec3(0.94, 0.96, 1.0);` | buildLandTintSphere 内 |
| 拠点の色 | shader `mix(vec3(0.11,0.17,0.55), vec3(0.28,0.38,0.95), vPulse)` | buildOffices 内 |
| ネットワーク線 | `new THREE.LineBasicMaterial({ color: 0xe8ecff, ... })` | Arc class 内 |

### 自転速度
`animate()` 内 `globeGroup.rotation.y += dt * 0.07;` の係数（現在 0.07 rad/sec ≈ 約90秒で1周）

### 初期の向き
`globeGroup.rotation.y = 2.05;` — ラジアン指定。

### 起動時に特定の拠点を中心に見せたい
```js
// 例: lat=35, lng=139 を中心に
// lng→rotation.y の換算: rotation.y = -lng * PI / 180
globeGroup.rotation.y = -139 * Math.PI / 180;
```

---

## パフォーマンス設計

### 自動最適化
- **モバイル判定**: `userAgent` + `pointer: coarse` メディアクエリで自動判別
- **モバイル時**: ドット数・球体解像度・テクスチャサイズ・ネットワーク線数などを約半減
- **MSAA**: モバイルでは無効化（iOS GPUの最大ボトルネック）
- **DPR上限**: デスクトップ2 / モバイル1.5

### 自動停止
- `IntersectionObserver`: globe が画面外に出たら描画ループ停止
- `visibilitychange`: タブが非アクティブ時も停止
- → **スクロールで過ぎ去ったあと、GPU/CPU はほぼゼロ**。LP中盤設置でも負荷が残らない

### Core Web Vitals
- **LCP**: first view でなければ影響ゼロ（`loading="lazy"` iframe 前提）
- **INP**: OrbitControls ドラッグ時のみ若干負荷、通常は 0
- **CLS**: iframe に固定 aspect-ratio を指定すれば 0

---

## 依存ライブラリ

すべて `unpkg.com` から ES module として動的読込。**ビルドツール不要**。

- `three@0.160.0` (build/three.module.js)
- `three@0.160.0/examples/jsm/controls/OrbitControls.js`
- 地形画像: `three-globe@latest/example/img/earth-topology.png`

`importmap` で解決しているので、モダンブラウザ（Safari 16.4+、Chrome 89+）のみ対応。

---

## ファイル

```
globe-demo/
├── index.html        ← 本体。単一ファイルで完結
├── README.md         ← このファイル
└── vercel.json       ← Vercel デプロイ設定（キャッシュヘッダ）
```

---

## 開発メモ

### 座標系
`latLngToVec3(lat, lng)` と fibonacci サンプルからの lat/lng 逆算（`atan2(z, -x)`）が同じ座標系になるよう統一してある。

### シェーダの "facing"
どのシェーダも `FACING_VS` スニペットで `vFacing = dot(normal, viewDir)` を計算し、裏面ドットを薄く前面を濃く描画。これで「透過する地球」感を出している。

### なぜ `depthTest: false` を多用？
透明オブジェクトが重なる箇所で、three.js の深度ソートに頼らず `renderOrder` で明示制御している（地球内部の見通しを担保するため）。
