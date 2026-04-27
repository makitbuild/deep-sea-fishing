# CLAUDE.md

## プロジェクト概要

単一HTMLファイルで動作する深海釣りゲーム。19世紀の航海日誌風の見た目（ジュール・ヴェルヌ調）。プレイヤーは11,000m（マリアナ海溝の深さ）の水柱に仕掛けを投下し、当たりを待ち、生物を釣り上げる。図鑑には種ごとに星表示のレア度とインラインSVGの絵が記録される。

成果物は `deep_sea_fishing.html` 一枚。ビルド工程なし、外部依存はGoogle Fontsの読み込みのみ。任意のモダンブラウザで開けば動く。

## なぜ一枚のHTMLか

元々がチャットで作成されたアーティファクト。設計上の制約：
- バンドラなし、import文なし
- 生物の絵は全てJS配列内のインラインSVG（外部画像なし）
- セーブはlocalStorageの素のJSON
- 外部通信はGoogle Fontsへの一回のみ

新規追加もこの方針を守る。画像ファイルよりインラインSVG、ライブラリより素のJS、ビルド工程は導入しない。

## アーキテクチャ

スクリプトは一枚のJSとして書かれており、コメント区切り（`// =================== DATA ===================` など）でセクション分けされている。モジュールがないので順序が意味を持つ — 後ろのセクションは前のセクションの定義に依存する。

```
DATA          ZONES, CREATURES, RARITY_* 定数
STATE         ゲーム状態オブジェクト、保存・読込
DOM           要素参照のキャッシュ
SCENE         buildScene() — 11,000m分の縦長シーンを描画
CAMERA        updateCamera() — 深度に追従する単純な平行移動
RENDER STATE  HUD更新（深度表示、ステータステキスト）
GAME LOGIC    フェーズマシン、当たり予約、生物抽選
INPUT         アクションボタンと海面タップのハンドラ
```

### 座標系

世界全体はひとつの絶対配置 `<div id="scene">` で表現され、その高さは `WORLD_PX = 11000 * 0.8 = 8800px`。カメラは `transform: translateY(-depth * 0.8)` するだけ。

`1メートル = 0.8ピクセル`。この係数（`METERS_PER_PX`）はパーティクル位置・深度マーカ・周囲生物の配置・海底ビネット等、随所にハードコードされている。`0.8` の乗算や `1/0.8` の除算を全箇所監査せずに変更しないこと。

### フェーズマシン

`state.phase` で進行を管理：

```
idle → casting → waiting → biting → reeling → (caught | escaped) → reeling-home → idle
```

遷移：
- `startCast()`：idle → casting（ボタン押下）
- MAX_DEPTH到達かボタン押下で `startWaiting()`
- `scheduleBite()` がsetTimeoutで `triggerBite()` を予約 → biting
- biting中の最初のタップで `startReeling()`（同じタップが最初の巻き上げ入力にもカウントされる — 後述）
- 巻き上げゲージ100% → `catchIt()`、時間バーが0 → `escape()`
- `reelHome()` で素早い戻し演出を再生し、idleに戻る

`gameTick()` はrAFループ。連続的な深度移動（`casting` と `reeling-home`）のみを担当する。離散イベントは状態オブジェクトに格納したsetTimeout/setIntervalで管理し、フェーズ離脱時にクリアする。

### 当たり時の最初のタップは二役

biting中の最初のタップは、針掛け（biting終了）と最初の巻き上げ入力を兼ねる。意図的な設計 — 「アタリ！」と「巻け！」を別タップに分けると操作が冗長で気持ち悪かったため。INPUTセクションのクリックハンドラ参照。

関連：巻き上げバーと時間バーは、巻き上げ開始時ではなく当たり時に予表示している（`triggerBite()` 内）。これがないとバーが現れた瞬間にアクションボタンが下方向にずれて、連打のリズムが崩れる。

### 永続化

```js
const STORAGE_KEY = 'deepSeaFishing_v1';

// 保存される形式：
{
  casts: number,
  maxDepth: number,
  collection: { [creatureId]: count }
}
```

`saveState()` は3か所で呼ばれる：投下時、最深記録更新時、捕獲成功時。`_v1` というサフィックスは将来のスキーマ移行時に新キーへ切り替えるための余地。

図鑑下部のエクスポート/インポート機能は同じ形式に `version` と `exportedAt` を加えてラップしている。インポートは構造を検証し、現在の `CREATURES` 配列に存在しないIDを黙って除外する — 一部の生物がリネームされた古いセーブも、リネームされた種を失うだけで他は復元できる。

## 生物を追加する手順

1. `deep_sea_fishing.html` を開き、`const CREATURES = [` のブロック（934行目付近）を見つける。
2. 該当ゾーンのグループに追加する（ゾーンは `// SURFACE`、`// TWILIGHT` などのコメントで区切られている）。図鑑は各ゾーン内でレア度昇順に並ぶ。同じレア度内では配列順を保つ。
3. 必須フィールド：

```js
{
  id: 'unique_snake_case',
  name: '日本語名',
  sci: 'Scientific name',  // 捕獲ポップアップに表示。創作でも可
  zone: 'surface' | 'twilight' | 'midnight' | 'abyss' | 'hadal',
  rarity: 'common' | 'uncommon' | 'rare' | 'legendary' | 'mythical',
  minD: 0,    // 出現深度の下限（メートル）
  maxD: 200,  // 出現深度の上限（メートル）
  desc: '一行程度の説明文。',
  svg: `<svg viewBox="..." width="..." height="...">...</svg>`
}
```

4. IDは配列全体で一意である必要がある：

```bash
grep -c "id:'YOUR_ID'" deep_sea_fishing.html  # 1 が出ればOK
```

5. SVGサイズの目安：60px（オキアミ、ハチェットフィッシュ）〜200px（ジンベエザメ、マッコウクジラ）。`viewBox` の縦横比は実物の体型に合わせる。
6. 深度範囲が周囲生物の配置と捕獲判定を兼ねる。例えば800m地点で釣れるようにするには `minD ≤ 800 ≤ maxD` を満たす必要がある。

レア度の重みは `RARITY_WEIGHT = {common:10, uncommon:4, rare:1.5, legendary:.5, mythical:.15}`。同じゾーンに common を大量に追加すると他レア度の出現率が下がる。mythical を追加しても平均捕獲率はほぼ変わらないが、到達可能なトロフィーが増える。

## 世界の深度を変える

`MAX_DEPTH = 11000` に紐づく箇所が複数ある：

- `WORLD_PX = 11000 * 0.8` 定数
- `ZONES` 配列の最終要素の `max`
- `buildScene()` の深度マーカ生成ループ（`for(let d=500; d<=11000; ...)`）
- 海底ビネット `top: ${9500*0.8}px` と海底岩肌 `top: ${11000*0.8 - 60}px`
- パーティクル分布 `Math.random()*Math.random()*10500`（バイアス項）
- ハダル層生物の `maxD` 値（一部は11000）
- タイトル文言（"マリアナ海溝の航海"）

`11000`、`10500`、`9500` を全文検索して網羅すること。

## 既存のID対応表

セーブデータと和名の対応を確認したい場合の参考：

```bash
# 全ID・全和名を一覧
grep -oE "\\{id:'[^']+',name:'[^']+'" deep_sea_fishing.html

# 特定の和名からIDを引く
grep -oE "\\{id:'[^']+',name:'マイワシ'" deep_sea_fishing.html
```

## 注意点

- **SVG断片はドキュメントネームスペースを共有する。** 2つの生物が同じ `<defs><radialGradient id="X">` を使うと衝突する。新規追加時はユニークなID（`glowA`、`glowB`、`pyroG`、`crysG`…）を使う。
- **既存のIDをリネームすると、古いセーブからその種が消える。** インポートのバリデーションで黙って除外される。リネームが必要な場合は `loadState()` に移行処理を追加する。
- **localStorage はオリジン単位。** `file://` で開いた場合と `http://localhost:8000` で開いた場合は別ストレージ。開発中に「セーブが消えた？」と思ったら出所を疑う。
- **モバイルの縦横切り替え対策はない。** 720px以下でグリッドが縦積みになる（`@media (max-width:720px)` 参照）。回転動作を都度確認。
- **アクションボタンの位置は当たり中に変わってはいけない。** ボタンとバーの間に何かUIを挟むと連打体験が壊れる。当たり時にバーを予表示する現在の実装がこれを防いでいる。
- **テンプレートリテラル内のバッククォート。** SVG文字列はテンプレートリテラル `` ` `` で囲まれている。SVG内に `` ` `` を含めると即座に壊れる。属性値は必ずダブルクォートで。

## 開発フロー

ビルドもテストランナーもpackage.jsonもない。

```bash
# 直接ブラウザで開く
open deep_sea_fishing.html

# または http で配信して localStorage を独立させる
python3 -m http.server 8000
# → http://localhost:8000/deep_sea_fishing.html
```

埋め込みJavaScriptの構文チェックだけ走らせたいときは：

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('deep_sea_fishing.html', 'utf8');
const m = html.match(/<script>([\s\S]*?)<\/script>/);
fs.writeFileSync('/tmp/check.js', m[1]);
require('child_process').execSync('node --check /tmp/check.js', {stdio:'inherit'});
"
```

ブラウザを開かなくても、SVG文字列内のバッククォート未エスケープやエントリ間のカンマ抜けなど、よくある破損はこれで捕まる。

種数や重複IDの確認：

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('deep_sea_fishing.html', 'utf8');
const ids = [...html.matchAll(/\\{id:'([^']+)'/g)].map(m=>m[1]);
console.log('Total entries:', ids.length);
console.log('Unique IDs:', new Set(ids).size);
const dupes = ids.filter((id,i) => ids.indexOf(id) !== i);
if(dupes.length) console.log('DUPLICATES:', [...new Set(dupes)]);
"
```

`Total entries` には `ZONES` 配列の5件も含まれるので、生物数はそれを引いた値。

## 検討中の機能

会話に出たが未実装：
- 達成バッジ / 称号（初の伝説級捕獲、特定ゾーン全捕獲など）
- 時間帯や天候による捕獲率の変動
- 複数の竿・餌のメタ進行レイヤ
- エクスポートしたログの共有機能（現状はJSONファイルのみ、オンライン要素なし）

コードベースは約2,000行で、いずれもリファクタなしで追加可能。フレームワークを導入したくなる衝動には抗うこと。
