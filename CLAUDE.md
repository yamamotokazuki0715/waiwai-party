# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

「ワイワイパーティー」— ブラウザで遊ぶオンライン同時プレイのパーティーゲーム。**`index.html` 1ファイルで完結**させるのが前提(ビルド工程・依存パッケージなし)。GitHub Pages で公開する。

- 外部依存は PeerJS(WebRTC)のみ。`cdn.jsdelivr.net` から `<script>` で読み込む
- シグナリングは PeerJS の公開クラウドサーバー。独自サーバーは持たない
- UI テキストは日本語

## 開発・動作確認

- 公開: `main` に push すると GitHub Pages(https://yamamotokazuki0715.github.io/waiwai-party/)に自動で反映される
- ビルド・lint・テストは無い。`index.html` をブラウザで開けば動く
- 複数タブで接続テストをするときは HTTP で配信する(`file://` だとタブ間テストがうまくいかないことがある)。例: `python -m http.server 8123`
- 1人でも「ルームをつくる」→「スタート」でプレイできる
- `requestAnimationFrame` は非表示タブで止まるため、自動テストでは入力が遅れて反映される。ゲームロジックの検証は `MODES.<id>.create/step` に合成入力を渡して直接回すと確実

## アーキテクチャ

### 通信モデル(ホスト権威 + 移動のみクライアント権威)

- ホストの Peer ID は `PEER_PREFIX + 4文字ルームコード`。クライアントはその ID に接続する。URL の `#CODE` で招待できる
- **ホスト**が `setInterval` 33ms でゲーム状態を進め(`step`)、スナップショットを全員に配信する(`st`)
- **各プレイヤー**は自キャラの移動・衝突をローカルで計算し、位置+ボタン押下回数を `in` メッセージで送る。ホストは位置をそのまま採用し、アイテム操作などのゲーム判定だけを行う
- ボタンは「押下回数カウンタ」(`g`, `a`)で送り、ホストが前回値との差分で押下を検出する(パケットをまたいでも取りこぼさない)
- ホスト自身の入力も `app.inputs[myId]` に入れ、クライアントと同じ経路で処理する
- `close` イベントが来ないケースに備え、2秒間隔の `ping` ハートビートで、10秒無通信なら切断扱いにする
- メッセージ種別は `index.html` の「通信」セクション冒頭のコメントに一覧がある

### モードの追加方法

`MODES[id]` にオブジェクトを登録し、`MODE_LIST` にメタ情報(名前・絵文字・説明・操作ヘルプ)を追加する。`MODE_LIST` にあって `MODES` に実装が無いものは、ロビーで「準備中」と表示される。

必要なメソッド:
- ホスト側: `create(players)`, `addPlayer(S,p)`, `removePlayer(S,id)`, `initData(S)`(開始時に1回だけ送る静的データ), `step(S,dt,inputs)`, `snapshot(S)`, `isOver(S)`, `result(S)` → `{stars:0-3, stats:[[ラベル,値],...]}`
- 全員: `clientFrame(app,inp,dt)`(ローカル移動・効果音・演出), `inputMsg(app,inp)`, `render(app,dt)`

効果音・ふきだしは、ホスト状態の `ev` 配列(連番 `id` 付き)に積む。各クライアントは `app.lastEv` 以降のイベントを再生する。

### ドタバタキッチン(`cooking`)

- マップは文字列グリッド(`#` カウンター、`T/L/O` 食材箱、`C` まな板、`S` コンロ、`P` お皿、`W` 提出口、`X` ゴミ箱)。座標はタイル単位で、キーは `"x,y"`
- 操作対象は、プレイヤーのいるマスから向き(4方向に丸める)の隣のマス(`target()`)
- アイテム: 食材 `{k, ch}`(`ch`=切った)、お皿 `{k:'plate', c:[...]}`。スープは `'<食材>_soup'` としてお皿の `c` に入る
- 注文判定は材料をソートして連結したキー(`recipeKey`)で比較する
