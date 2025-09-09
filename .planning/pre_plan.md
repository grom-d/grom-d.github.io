# User

スマートフォン対応のWebブラウザで動くアプリケーションの開発を考えています。一緒に開発計画を作ってください

<前提条件>
・開発経験ゼロなので、自分だけが確認できる習作でよい
・ひとまずiphone向け
</前提条件>

<アプリ内容>
・Youtubeの動画を再生できるアプリ
・2つの動画をMixして再生できる機能
・公開プレイリストを使った選択
・2ラインそれぞれのボリューム調整
・ターンテーブルをイメージした再生箇所の調整機能
</アプリ内容>

# AI
雷童さん、目的は「iPhoneのSafari等で動く“2本のYouTube動画を同時再生してミックス風に楽しむ”学習用Webアプリ」ですよね。まずは“完全クライアント/ブラウザのみ”で成立する最小構成から始め、学びやすく壊れにくい計画に落とし込みます。前提として、**YouTubeはダウンロードや音声抽出は禁止**、**自動再生や同時再生はユーザー操作が必須**などの制約があるため、**公式の「IFrame Player API」で2つのプレーヤーを並べて音量・位置をコントロール**する方針で進めます（音声を取り出してWebAudioで合成はTOS的にも技術的にも非推奨）。

---

# ゴールと制約（コンテキスト/前提 → 設計の意図）

* **ターゲット**：iPhone（iOSのWebKit/Safari前提）。
  → 自動再生制限：ユーザーのタップ後に再生開始。`playsinline=1` 指定。
* **API/法務**：YouTube IFrame Player APIのみ使用。音声ストリーム抽出はしない。公開プレイリストは**IDが分かっている前提**なら**Data API不要**（`listType=playlist&list=...` で読み込める）。
* **“Mix”の定義**：2プレーヤーを**同時再生**し、**個別音量**＋**クロスフェーダー**で“聴感上のミックス”にする。波形解析や完全同期はやらない（学習負荷とTOS回避）。
* **サーバ不要**：まずは静的ホスティング（GitHub Pages等）で完結。学習用のためAPIキーも不要の構成から。
* **UI**：シンプル（2デッキ＋クロスフェーダー＋再生位置ジョグ）。スマホ片手操作に最適化。

---

# アーキテクチャ（MVP）

* **フロントのみ（HTML/CSS/JS）**

  * 2つの `<div id="playerA">`, `<div id="playerB">` に IFrame Player API で埋め込み
  * **動画指定**：個別動画ID入力 or 公開プレイリストID入力（各デッキ独立）
  * **制御**：

    * 再生/停止/ロード：各デッキにボタン
    * **音量**：各デッキのスライダー（0–100）
    * **クロスフェーダー**：中央0で両方、左寄せでA重視、右寄せでB重視（曲線：等電力カーブ）
    * **ジョグ/スクラブ**：ターンテーブル風UI（円形ノブ or 円周ドラッグで `seekTo(currentTime + Δ)`）
* **同期の考え方**：完全同期は狙わず、「同時スタート」＋ジョグ微調整で“それっぽく”合わせる。
* **再生開始要件**：初回は「Arm（武装）」ボタンで**ユーザー操作**→両プレーヤーを`mute`で起動→その後`unMute`して本再生。
* **プレイリスト**：リストIDが分かっているなら `player.loadPlaylist({list: 'PLAYLIST_ID'})` を使用。
* **データ保存**：LocalStorage（最後に使った動画IDやフェーダー位置）。

---

# UIラフ（スマホ縦向き）

* 上：デッキA（タイトル/サムネ縮小、再生・停止、音量スライダー、ジョグ円）
* 中：**クロスフェーダー**（大きめの水平スライダー＋CUE/同時再生ボタン）
* 下：デッキB（同上）
* 設定：プレイリスト/動画ID入力、`playsinline`/画質自動、ヘルプ

---

# 機能分解 & マイルストーン（ゼロ経験向け・小刻み）

**M0：環境準備（半日）**

* GitHubアカウント作成 → リポジトリ作成 → GitHub Pages（静的公開）
* 最小HTML（1ページ）を公開できる状態に

**M1：単一プレーヤー再生（0.5〜1日）**

* IFrame Player API読み込み、`playerA`埋め込み
* 任意動画IDで**手動**で再生/停止できる（タップ必須）
* `playsinline=1`, `mute/unMute`, `setVolume`, `getCurrentTime`

**M2：二重化 & 同時再生（1〜2日）**

* `playerB`追加、**同時スタート**（ボタン1つでA/B再生開始）
* iOSの**ユーザー操作必須**を満たすUI（「Arm→Play」2段階でもOK）

**M3：音量/クロスフェーダー（1日）**

* A/B個別音量スライダー
* クロスフェーダー：`x∈[0,1]` として

  * 等電力カーブ例：`volA = cos(π/2 * x)`, `volB = cos(π/2 * (1 - x))` を 0–100 にスケール

**M4：ジョグ/スクラブ（1〜2日）**

* 円形UIをドラッグで `seekTo(current + k*deltaAngle)`
* **ステップ安全策**：±0.05〜0.2秒刻み、フリックで連続シーク抑止

**M5：公開プレイリスト対応（0.5〜1日）**

* プレイリストID入力→`loadPlaylist({list:'ID'})`
* 次/前トラック、`getPlaylistIndex` 表示

**M6：UX整え（1日）**

* スマホ向けタッチ領域拡大、視認性（大きめボタン）
* LocalStorageで前回状態復元
* 簡易ヘルプ（初回チュートリアル）

**任意M7：デザイン/軽微アニメ（1日）**

* CSSでターンテーブル風（回転演出は**視覚だけ**。実際の再生位置はAPIで制御）

---

# 技術メモ（落とし穴と回避）

* **自動再生**：iOSは**無音かつユーザー操作後**でないと基本不可。まず`mute=true`で`playVideo()`→後で`unMute()`が堅実。
* **2動画の“厳密同期”**：フレーム精度は難しい。狙わない。`seekTo()`で耳合わせ＋ジョグ。
* **音声処理**：YouTube iframeの音声を**WebAudioに配線しない**（TOS/クロスオリジン問題）。**`setVolume`のみ**で表現。
* **Data APIキー**：最初は**不要**。検索や詳細取得が欲しくなったら**小さなバックエンド**を後段で検討（キー秘匿）。
* **レスポンシブ**：縦向き主体、幅が狭いときは要素を縦積み。
* **エラー対応**：`onError`（動画再生不可）、`UNSTARTED/ENDED`の状態遷移でUI無効化/次曲へ。

---

# 学習タスク（順に写経でOK）

1. **最小HTML**でIFrame Player APIを読み、任意動画を1本再生
2. `setVolume`, `seekTo`, `getCurrentTime` をコンソールから叩けるように
3. 2本化して同時スタートボタンを実装
4. クロスフェーダー（等電力）を実装
5. 円形ノブで `seekTo` 微調整
6. プレイリストIDのロード & 次/前ボタン
7. 初回ヘルプとLocalStorage保存

---

# テスト計画（iPhone）

* **機種/OS**：iOS 17/18想定、Safari/Chrome（中身は同WebKit）
* **シナリオ**：

  * 初回アクセス→「Arm」→A/B読み込み→同時再生→クロスフェーダー操作
  * ジョグで±0.1秒刻みの追い込み
  * プレイリストで次曲遷移→再度同時再生
* **確認ポイント**：音ズレ/引っかかり、UI反応、縦横回転、スリープ復帰

---

# 将来拡張（学んだ後のv2）

* 軽量バックエンド（Cloudflare Workers等）で**プレイリスト検索**や**サムネ/タイトル取得**（Data API v3）
* お気に入り保存（DB不要ならKV/JSONでも）
* BPM手動入力＋自動`seekTo`補助（本格的な解析はしない）
* PWA化（ホーム画面追加・オフラインは限定的）

---

## すぐ始められる最小スターター（貼り付け用・1ファイル）

> 👉「コードは後で」で良ければスキップ可。学習開始の足がかり用です。

```html
<!doctype html>
<html lang="ja">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>YT Mini Mixer (Practice)</title>
<style>
  body { font-family: system-ui, sans-serif; margin: 10px; }
  .deck { border: 1px solid #ccc; border-radius: 12px; padding: 10px; margin-bottom: 10px; }
  .row { display:flex; gap:8px; align-items:center; margin:6px 0; }
  .col { flex:1 }
  .big { width:100%; }
  #xfader { width:100%; }
  button, input[type=range] { touch-action: manipulation; }
</style>
<body>
<h1>YT Mini Mixer</h1>

<div class="row"><button id="arm">Arm(準備)</button><button id="playBoth">Play</button><button id="pauseBoth">Pause</button></div>
<div class="row"><label class="col">Crossfader</label><input id="xfader" type="range" min="0" max="100" value="50"></div>

<div class="deck" id="deckA">
  <h3>Deck A</h3>
  <div class="row">
    <input id="vidA" class="col" placeholder="動画ID or プレイリストID">
    <select id="modeA"><option value="video">video</option><option value="playlist">playlist</option></select>
    <button id="loadA">Load</button>
  </div>
  <div id="playerA"></div>
  <div class="row">
    <button id="playA">Play</button><button id="pauseA">Pause</button>
    <input id="volA" type="range" min="0" max="100" value="80">
    <span id="timeA">0.0s</span>
  </div>
  <div class="row">
    <button data-jog="-0.2">⟲ -0.2s</button><button data-jog="0.2">⟳ +0.2s</button>
  </div>
</div>

<div class="deck" id="deckB">
  <h3>Deck B</h3>
  <div class="row">
    <input id="vidB" class="col" placeholder="動画ID or プレイリストID">
    <select id="modeB"><option value="video">video</option><option value="playlist">playlist</option></select>
    <button id="loadB">Load</button>
  </div>
  <div id="playerB"></div>
  <div class="row">
    <button id="playB">Play</button><button id="pauseB">Pause</button>
    <input id="volB" type="range" min="0" max="100" value="80">
    <span id="timeB">0.0s</span>
  </div>
  <div class="row">
    <button data-jogB="-0.2">⟲ -0.2s</button><button data-jogB="0.2">⟳ +0.2s</button>
  </div>
</div>

<script src="https://www.youtube.com/iframe_api"></script>
<script>
let playerA, playerB, armed = false;
function onYouTubeIframeAPIReady() {
  playerA = new YT.Player('playerA', { height:'180', width:'320',
    playerVars:{ playsinline:1, enablejsapi:1, origin:location.origin },
    events:{ 'onReady': onReadyA }});
  playerB = new YT.Player('playerB', { height:'180', width:'320',
    playerVars:{ playsinline:1, enablejsapi:1, origin:location.origin },
    events:{ 'onReady': onReadyB }});
}
function onReadyA(){ playerA.setVolume(80); }
function onReadyB(){ playerB.setVolume(80); }

document.getElementById('arm').onclick = () => {
  // iOS向け：ユーザー操作で一度mute再生→準備完了後に停止
  [playerA, playerB].forEach(p => { try { p.mute(); p.playVideo(); setTimeout(()=>p.pauseVideo(), 200); } catch(e){} });
  armed = true;
  alert('Armed: 以降の再生が安定します');
};

function loadDeck(idInput, modeSel, player){
  const id = idInput.value.trim();
  if(!id) return;
  if(modeSel.value === 'playlist'){
    player.loadPlaylist({listType:'playlist', list:id});
  } else {
    player.loadVideoById(id);
  }
}
document.getElementById('loadA').onclick = ()=> loadDeck(vidA, modeA, playerA);
document.getElementById('loadB').onclick = ()=> loadDeck(vidB, modeB, playerB);

document.getElementById('playBoth').onclick = ()=>{
  if(!armed) alert('先にArmを押してください');
  playerA.playVideo(); playerB.playVideo();
};
document.getElementById('pauseBoth').onclick = ()=>{ playerA.pauseVideo(); playerB.pauseVideo(); };
document.getElementById('playA').onclick = ()=> playerA.playVideo();
document.getElementById('pauseA').onclick = ()=> playerA.pauseVideo();
document.getElementById('playB').onclick = ()=> playerB.playVideo();
document.getElementById('pauseB').onclick = ()=> playerB.pauseVideo();

volA.oninput = e => playerA.setVolume(+e.target.value);
volB.oninput = e => playerB.setVolume(+e.target.value);

// クロスフェーダー（等電力的カーブ）
xfader.oninput = e => {
  const x = +e.target.value / 100; // 0..1
  const volA = Math.cos(Math.PI/2 * x);
  const volB = Math.cos(Math.PI/2 * (1-x));
  playerA.setVolume(Math.round(volA*100));
  playerB.setVolume(Math.round(volB*100));
};

// 簡易ジョグ
function jog(player, delta){ try {
  const t = player.getCurrentTime ? player.getCurrentTime() : 0;
  player.seekTo(Math.max(0, t + delta), true);
} catch(e){} }
document.querySelector('[data-jog="-0.2"]').onclick = ()=> jog(playerA, -0.2);
document.querySelector('[data-jog="0.2"]').onclick  = ()=> jog(playerA,  0.2);
document.querySelector('[data-jogB="-0.2"]').onclick= ()=> jog(playerB, -0.2);
document.querySelector('[data-jogB="0.2"]').onclick = ()=> jog(playerB,  0.2);

// 簡易タイマー表示
setInterval(()=>{
  try { timeA.textContent = playerA.getCurrentTime().toFixed(1)+'s'; }catch(e){}
  try { timeB.textContent = playerB.getCurrentTime().toFixed(1)+'s'; }catch(e){}
}, 200);
</script>
</body>
</html>
```

> これを `index.html` としてGitHub Pagesに置けば、**M1〜M3の骨格**が動きます。
> （厳密同期や高度な波形表示はあえて入れていません）

---

## リスクと対策（要点）

* **iOS自動再生制限**：Armボタンでユーザー操作を先に一度入れる。
* **TOS抵触の回避**：音声抽出・録音・ダウンロードはしない／IFrame APIのみ。
* **ラグ/同期ズレ**：完全同期は不問。ジョグで手当て。
* **公開プレイリストの探索**：当面は**ID手入力**。将来はバックエンド経由でData API検索。

---

## 次アクション（今日やること）

1. リポジトリ作成 → GitHub Pages有効化
2. 上記HTMLをそのままコミットして公開
3. 手持ちの**動画ID**と**公開プレイリストID**で読み込みテスト
4. 操作の気づきメモ（使いづらい点）を書き出す → M6で改善

必要なら、\*\*課題管理用のToDo（Issueテンプレ）\*\*や、\*\*デザイン（Figmaの簡易ワイヤー）\*\*もすぐ用意します。
この計画で走り出してみましょう。躓いたポイントが出たら、その場で“最小修正”を一緒に入れていきます。

# User
同時再生ではなく、DJのつなぎのような、ずらしたうえでそれぞのトラックで再生・停止がやりたいんですが、実現できるでしょうか。

# AI
できます。要点は「2つのIFrame Playerを**独立制御**し、各デッキで**任意のキューポイントから個別に再生/一時停止/停止**できるようにする」ことです。**同時再生は前提にせず**、A→Bへ“ずらして”つなぐ運用にします。

## できること / 制約（前提の再確認）

* 可能：各デッキで

  * 任意位置に**CUE設定**（現在位置を記録）
  * **CUEから再生**（A/Bを別タイミングで開始）
  * **一時停止/停止**（Aだけ止めてBだけ再生、など）
  * **クロスフェーダー**で音量を移行
  * **ジョグ**で微調整（±0.05〜0.2s）
* 注意：iOSは**自動再生制限**が強い
  → 最初に「Arm（準備）」で**ユーザー操作**を入れ、各プレイヤーを一瞬`mute`再生→`pause`して“プライム（再生許可）”しておく。
  → 後で\*\*時間差での`playVideo()`\*\*が通りやすくなる（ただし環境差はあるので、失敗時はUIで「タップして開始」を促すフォールバックを用意）。

---

## UXフロー（DJのつなぎを想定）

1. **Deck A** を再生（イントロ〜1番など）
2. **Deck B** をロード → 入れたい地点で**CUE**を打つ（Bのサビ頭など）
3. タイミングを見て **B: CUEから再生** → クロスフェーダーを**A→B**へ移動
4. Aを**一時停止/停止**。必要に応じてBのジョグで微調整

> 完全な拍同期（BPM解析や位相合わせ）はやりません。耳合わせ＋ジョグで十分“それっぽく”なります。

---

## 追加する最小の実装（前回スターターの拡張）

以下の概念だけ入れ替え/追加すればOKです。

### 1) CUEの状態を持つ

```js
let cueA = 0, cueB = 0;
function setCue(player, which) {
  try {
    const t = player.getCurrentTime();
    if (which==='A') cueA = t; else cueB = t;
  } catch(e){}
}
function playFromCue(player, which) {
  const t = (which==='A') ? cueA : cueB;
  try {
    player.seekTo(Math.max(0, t), true);
    player.playVideo();
  } catch(e){}
}
function stopDeck(player) {
  try { player.stopVideo(); } catch(e){}
}
```

### 2) ボタン（各デッキに追加）

```html
<!-- Deck A 内の操作列に追加 -->
<button id="cueSetA">Set CUE</button>
<button id="cuePlayA">Play CUE</button>
<button id="stopA">Stop</button>

<!-- Deck B も同様 -->
<button id="cueSetB">Set CUE</button>
<button id="cuePlayB">Play CUE</button>
<button id="stopB">Stop</button>
```

```js
document.getElementById('cueSetA').onclick = ()=> setCue(playerA, 'A');
document.getElementById('cuePlayA').onclick = ()=> playFromCue(playerA, 'A');
document.getElementById('stopA').onclick    = ()=> stopDeck(playerA);

document.getElementById('cueSetB').onclick = ()=> setCue(playerB, 'B');
document.getElementById('cuePlayB').onclick = ()=> playFromCue(playerB, 'B');
document.getElementById('stopB').onclick    = ()=> stopDeck(playerB);
```

### 3) “時間差開始”の補助（任意）

「今からn秒後にBを走らせる」簡易機能。
※ iOS制約で失敗したらUIでタップ促し。

```html
<div class="row">
  <input id="delaySec" type="number" min="0" max="10" step="0.1" value="0.5">
  <button id="startBDelayed">Start B after n sec</button>
</div>
```

```js
document.getElementById('startBDelayed').onclick = ()=>{
  const n = +document.getElementById('delaySec').value || 0;
  // 成功率を上げるため、事前に軽く再生→停止して“温める”ことも有効
  try { playerB.mute(); playerB.playVideo(); setTimeout(()=>playerB.pauseVideo(), 150); } catch(e){}
  setTimeout(()=>{
    try {
      playerB.seekTo(cueB, true);
      playerB.playVideo();
      setTimeout(()=>playerB.unMute(), 50); // 音出し
    } catch(e){
      alert('開始できませんでした。BのPlayをタップしてください。');
    }
  }, Math.max(0, n*1000));
};
```

### 4) Arm（準備）の強化

初回タップ時に**両デッキをプライム**（再生許可を得やすくする）しておきます。

```js
document.getElementById('arm').onclick = () => {
  [playerA, playerB].forEach(p => {
    try { p.mute(); p.playVideo(); setTimeout(()=>p.pauseVideo(), 200); } catch(e){}
  });
  armed = true;
  // ここで各プレイヤーのバッファが温まるまでの小休止を入れると安定度UP
  setTimeout(()=>alert('準備完了：各デッキを個別に操作できます'), 50);
};
```

---

## つなぎを“それっぽく”する操作ヒント

* **等電力クロスフェーダー**を使う（既に実装済みのcosカーブ）
* **ジョグ刻み**は最初0.2s→慣れたら0.1s/0.05sへ
* **CUE+STUTTER**（CUE再生→直後停止→再再生）で“頭出し”を身体で掴む
* プレイリスト使用時は、**B側を次曲に先送りしてCUE**→任意のタイミングで投入

---

## 既知の落とし穴と回避

* **遅延/失敗**：`playVideo()`が無反応なときは、UIで明示（ボタン点滅/トースト）→**直接タップ**を促す
* **シークの重さ**：短時間に連続`seekTo`するとカクつく。**連打をデバウンス**（例：200ms以上間隔）
* **プレイリスト切替**：曲切替直後は`getCurrentTime()`が0に戻るまで数百msかかる → CUE設定は**安定待ち**してから

---

## 次アクション（最短で検証）

1. 既存のスターターに上記の**CUE/Stop/Delayed Start**を差し込み
2. iPhoneで**A→B**の手動つなぎを試す（BはCUEから投入）
3. 失敗時のUIフォールバック（「タップして開始」）を入れる

> ここまでで「**ずらして個別に再生・停止**」は実現できます。
> 必要なら、この拡張を反映した\*\*完全版`index.html`\*\*を用意します。
