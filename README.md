# 都市交通シミュレーター v22 開発ログ

## ファイル構成

| ファイル | 説明 |
|---|---|
| `traffic_simulation_v22.html` | メインゲーム本体 |
| `map_editor.html` | マップエディタ（斜め線対応済み） |
| `douro_asym.png` | 非対称道路テクスチャ（140×54px、上り2車線+下り1車線） |
| `douro_asym_lane1closed.png` | 非対称道路・車線封鎖バリアント（将来用） |

---

## 主要定数

```javascript
const LANE_W = 18;          // 1車線幅 (px)
const MIN_GAP = 20;         // 最小車間距離
const SLOW_GAP = 60;        // 徐行開始距離
SPEED_LIMIT: main=2.2, branch=1.1, oneway=1.5, asym=2.2
ゲーム内24時間 = 3600フレーム（06:00スタート）
```

---

## 実装済み機能

### 道路システム

#### 非対称道路（`type='asym'`）
- 上り2車線・下り1車線の非対称3車線道路
- `douro_asym.png`（140×54px）をテクスチャとして使用
- `e.asymWide`: true=2車線側、false=1車線側
- `e.closedLane`: 将来の1車線封鎖用（-1=なし）
- `e.underConstruction`: 植栽帯↔工事帯の切替フラグ

**ヘルパー関数**
```javascript
addAsymEdge(a, b, flip=false)
// flip=false: a→b=2車線、b→a=1車線
// flip=true: 逆向き
```

**植栽帯 / 工事帯（コード描画）**
- 通常時：緑地ベース＋丸い植木（54px間隔）
- 工事中（`underConstruction=true`）：砂利地肌＋カラーコーン（40px間隔）

**道路操作ボタン**
- 「↔ 非対称反転 -600pt」：広狭を入れ替え（走行中の車の車線もクランプ）
- 「🚧 工事帯切替 -400pt」：植栽⇔工事帯トグル

#### 立体交差（`isOverpass`）
- 踏切ノードをクリックで踏切↔立体交差トグル（-1200pt）
- 立体交差では踏切停止チェックをスキップ
- Box Junction制御を立体交差にも適用（重なり防止）
- yield判定から除外（ただの通過点として扱う）

---

### 交差点描画

#### 斜め道路対応
- 凸包ベースの交差点パッド（斜めエッジ検出時）
- `edgeRoadWidth(e)` グローバルヘルパーで非対称エッジの幅を正確に計算
- `maxRoadWidthAtNode(n)` でノード側の最大道路幅を取得
- 横断歩道・停止線の範囲を `rangeMin=-oppositeRoadW`、`rangeMax=+roadW` で正確に計算
  - `inEdge` の `perpX` 正方向 = 道路のある側（from→toの右側）

#### カーブノードのyield判定スキップ
- 出口方向が1つのみのノード（`outEdges.length <= 1`）はyield判定スキップ
- 60%速度でカーブ通過

---

### 固定MAP一覧

| ID | 名前 | 説明 |
|---|---|---|
| `simple_cross_y` | シンプル交差点Y | 八角形ループ＋出入口2本 |
| `simple_cross_z` | シンプル交差点Z | 斜め幹線あり |
| `asym_test` | 非対称道路テスト | 幹線×非対称縦道路、flip機能確認用 |
| `asym_test2` | 非対称道路テスト2 | シンプル交差点Zの幹線を非対称化 |
| `crossing_a` | 踏切のある交差点A | 駅・踏切・駐車場・ガソリンスタンド |
| `crossing_b` | 線路のある交差点B | 複線・立体交差 |

---

### 建物・施設

#### ガソリンスタンド（`houseType='gasStation'`）
- サイズ：72×72px（住宅2軒分）
- レイアウト：給油エリア（下半分）＋コンビニ本体（奥左2/3）＋洗車機棟（奥右1/3）
- `gsFacing`：描画向き（south/north/east/west）、回転対応
- 給油スロット4台（ポンプ島2基の両脇）
- 給油時間：300〜900フレーム（通常駐車より短め）
- 売上計算：給油量30〜60L × 単価¥165〜185/L を来客ごとに累積
- クリックで情報パネル表示（給油中台数・向かう車・累計売上・来客数・給油量・客単価）

**crossing_a での配置**
- x=882, y=268（主線横y=340の北側）
- `gsFacing='south'`（給油エリアが道路側）
- `connectParkingToNearestEdge` で接続（外部ノード接続エッジも許可に修正済み）

---

### 駅情報パネル
- 駅の屋根・ホームをクリックで表示
- 電車状態（走行中/停車中＋残り秒数）・方向・停車回数・踏切数
- 駅前駐車場セクション：現在台数・向かう車・累計利用（台数＋乗客数）・1停車あたり約N人
- 乗客数：車1台につき1〜4人でランダム計算
- 0.5秒ごとに自動更新

---

### 時間帯システム

#### 時刻変化
```
t=0    → 06:00  朝ラッシュ
t=375  → 09:00  午前
t=625  → 12:00  昼
t=750  → 13:00  午後
t=1125 → 17:00  夕方ラッシュ
t=1375 → 19:00  夜
t=1750 → 22:00  深夜
```

#### 朝ラッシュの駅前集中
- `t < 375`（06:00〜09:00）：65%の確率で駅前駐車場を目的地に選択

#### 夜間演出
- **案①：暗色オーバーレイ**：青みがかった暗色（`rgba(5,8,25,...)`）をCanvas全体に
- **案②：信号交差点の光ハロー**：`hasSignal && isRealCross` のノード周辺に暖色の放射光
- ガソリンスタンドも薄く光る（24時間営業）
- `nightAlpha`：0〜0.68 でなめらかに遷移（`nightAlpha += (target - nightAlpha) * 0.02`）

---

### 出口番号機能
- MAP生成後に `external=true` のノードをx→y座標順に採番（`n.exitNumber`）
- `drawExitNumbers()` で濃紺丸バッジに番号を常時表示
- 出口クリックで「向かっている車: N台」トースト表示

---

## バグ修正履歴（今セッション）

| 問題 | 原因 | 修正 |
|---|---|---|
| 非対称道路の横断歩道ズレ | `perpX`正方向の誤解（道路のある側が正） | `rangeMin=-oppositeRoadW`, `rangeMax=+roadW` に修正 |
| 非対称反転時クラッシュ | `lanes`減少時に車の`route.lane`がクランプされない | ルート内全stepと走行中車のlaneをクランプ |
| GSに車が来ない | `connectParkingToNearestEdge`が外部ノード接続エッジをスキップ | `e.from.external && e.to.external`（両端が外部のみ）に緩和 |
| 立体交差で車が重なる | Box Junction制御が`!isCrossing`で立体交差を除外 | `!isCrossing || isOverpass`に修正、yield判定からも除外 |

---

## 将来の候補（未実装）

- **案②テンプレート組み合わせ方式**：固定MAPパーツの接続
- **案④プリセットシナリオ方式**：市街地型・郊外型など
- **案①OSMベース**：長期的な拡張
- **4車線→1車線封鎖**：`e.closedLane`フラグ準備済み
- **夜間演出案③**：走行中の車のヘッドライト
- **夜間演出案④**：建物の窓灯り

---

## 技術メモ

- `perpX`正方向 = `inEdge`（`to=n`）の道路側（from→toの右側 = `y=-roadW〜0`の描画側）
- asymエッジの描画: `y=-roadW〜0`（片側基準、onewayは中心基準）
- 工事帯のカラーコーンは`timeCounter`なし（静的描画）
- `connectParkingToNearestEdge`：外部ノード接続エッジも中間点接続を許可（`e.from.external && e.to.external`のみスキップ）
- ゲーム内時間: 1フレーム=ゲーム内約24分、3600f=1日

---

## 【重要】道路サインバグ詳解

> 「発生箇所に規則性がない」「マップ・道路タイプを問わず散発する」という症状が長期間続いた。
> 根本原因は `connectParkingToNearestEdge` によるエッジ分割と、`drawDirectionSigns` の遡りロジックの相互作用にある。

### 背景：connectParkingToNearestEdge のエッジ分割

建物を道路に接続するとき、最寄りのエッジ `A→B` の中間点に `accessNode`（`isParkingAccess=true`）を挿入して分割する。

```
分割前:  A ──────────────── B
分割後:  A ── acc ── B        （accはisParkingAccess=true）
```

複数の建物が同じエッジを分割すると多重になる：

```
A ── acc1 ── acc2 ── acc3 ── B
```

対向エッジも同時に分割されるため、両方向のエッジチェーンが存在する。
accNodeに入るエッジが常に2本（正方向・逆方向）存在することがバグの温床。

### 修正した問題（全5箇所）

#### 修正1: connectParkingToNearestEdge — 新エッジのlanesコピー漏れ

```javascript
// 修正前: new Edge() で lanes がデフォルト（mainでも1）になる
edges.push(new Edge(accessNode, origTo, e.type));

// 修正後: 元エッジの lanes・asymWide を明示コピー
const newEdgeAB = new Edge(accessNode, origTo, e.type);
newEdgeAB.lanes    = e.lanes;
newEdgeAB.asymWide = e.asymWide;
newEdgeAB.laneCars = Array.from({length: e.lanes}, () => []);
```

main（2車線）エッジが分割されると acc→B の新エッジが1車線になり、サインが1車線分しか描かれなかった。対向エッジ（origFrom→accessNode）も同様に修正。

#### 修正2: getEffectiveFrom — 内積による方向判定（最重要）

`drawDirectionSigns` でサインを描く際、`e.from` が `isParkingAccess` の場合は「実質的な始点」まで遡る必要がある。

**問題の構造**:
```
n4(504) → acc(270) → n6(216)  ← n6に向かうエッジ acc→n6 を処理中
n6(216) → acc(270)             ← 逆向きエッジも存在
```

`edges.find(ex => ex.to === cur)` は2本ヒットするが、最初に見つかった逆向きの `n6→acc` を採用してしまい `effectiveFrom=n6=toNode` → スキップ。

**修正**: 内積で「進行方向と同じ向きのエッジ」のみを採用：

```javascript
function getEffectiveFrom(e, toNode){
  if(!e.from.isParkingAccess) return e.from;
  const dirX = toNode.x - e.from.x;  // e の進行方向ベクトル
  const dirY = toNode.y - e.from.y;
  let cur = e.from;
  for(let _i = 0; _i < 20; _i++){
    const prevEdge = edges.find(ex => {
      if(ex.to !== cur) return false;
      if(ex.from.type === 'parking') return false;
      const dx = cur.x - ex.from.x;
      const dy = cur.y - ex.from.y;
      return (dx * dirX + dy * dirY) > 0;  // 内積が正 = 同じ向き
    });
    if(!prevEdge) break;
    if(!prevEdge.from.isParkingAccess) return prevEdge.from;
    cur = prevEdge.from;
  }
  return null;
}
```

**重要**: 呼び出しは必ず `getEffectiveFrom(e, toNode)` と `toNode` を渡すこと。

#### 修正3: outEdges のUターン除外を e.from ベースに変更

```javascript
// 修正前: effectiveFrom（遠い元ノード）で除外 → 本来の出口エッジまで除外してしまう
ex.to !== effectiveFrom

// 修正後: e.from（直前ノード）で除外
ex.to !== e.from
```

#### 修正4: outEdges の駐車場フィルタを緩和

```javascript
// 修正前: accessNodeへの出口を全除外 → acc→cross のエッジが出口から消える
!ex.to.isParkingAccess

// 修正後: parking本体のみ除外（accessNodeの先には正規交差点がある）
ex.to.type !== 'parking'
```

#### 修正5: autoDirs の方向計算に fakeInEdge を使用

```javascript
// 修正前: e（fromがaccessNode）をそのまま渡す → 方向ベクトルがズレる
getDirectionType(e, ex)

// 修正後: effectiveFrom を使った仮エッジで正しい方向を計算
const fakeInEdge = { from: effectiveFrom, to: toNode };
getDirectionType(fakeInEdge, ex)
```

openSignPanel の getAutoDir 関数も同様の修正が必要（同じロジックを使っている）。

### デバッグ方法

サインが消えていると思ったら `drawDirectionSigns` 内に以下を仕掛けてコンソールで確認：

```javascript
// スキップ理由をカウント
const _skip = {};
// 各条件の後: if(_skip[...]||0)+1 でカウント
// 描画成功時: console.log('[DRAWN]', effectiveFrom, '→', toNode, key)
```

主なスキップ理由と対処：

| ログ | 原因 | 対処 |
|---|---|---|
| `outEdges=0` | toNodeからの出口が全除外されている | `e.from` / `parking` フィルタ確認 |
| `effectiveFrom=null` | accessNode遡りが途中で止まる | 内積の符号・toNode引数を確認 |
| `effectiveFrom===toNode` | 遡り先が交差点自身 | getEffectiveFrom の方向判定を確認 |
| `signKey=null` | autoDirs が空 | fakeInEdge と outEdges 内容を確認 |
| `alongOffset<2` | エッジが短すぎる | 正常（短いエッジは描かない） |
