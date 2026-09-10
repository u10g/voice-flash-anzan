# テスト手順

## 1. ローカル確認（PC）

```
python3 -m http.server 8000
```

`http://localhost:8000/` を開く（`localhost` は secure context 扱いのため音声認識も動く）。

| # | 確認内容 | 期待 |
|---|---|---|
| L1 | コンソール | エラー・CSP 違反が出ない |
| L2 | 診断パネル | 端末 / 接続 / 起動方法 / 音声認識 / マイク が表示される |
| L3 | タップ入力モード | 正解の数字を入力すると答えが表示され、`出題` `正答` `平均` が更新される |
| L4 | 幅 390px / 高さ 844px | ヘッダ・数字・操作バーが画面内に収まり、横スクロールが出ない |
| L5 | 「診断をコピー」 | テキストエリアに診断内容が出る |

## 2. 実機確認

公開 URL（`https://u10g.github.io/voice-flash-anzan/`）を実機で開く。

### iPhone / iPad（Safari）

| # | 手順 | 期待 |
|---|---|---|
| I1 | 診断パネルを開く | 「保護された接続: はい」「起動方法: ブラウザ」 |
| I2 | 「音声認識を試す」→ 何か話す | 「使える」または「起動は成功（声なし）」 |
| I3 | 「開始」→ マイク許可 | マイクの点が点灯し「聞き取り中」 |
| I4 | 出題に対し「じゅうご」等と発声 | 正解時に答えが表示され次の問題へ進む |
| I5 | 「正誤の音」を鳴らす設定で連続 5 問 | 音が鳴った後も認識が継続する（`audioBusyUntil` の効果） |
| I6 | 1 分以上連続で使う | 途中で切れても自動再接続し、聞き取りが続く |
| I7 | ホーム画面に追加して起動 | 「Safari で開く」バナーが出る |
| I8 | 設定 → Safari → マイク を拒否にして再試行 | 発声検出またはタップ入力へ自動で切り替わる |

### Android（Chrome）

| # | 手順 | 期待 |
|---|---|---|
| A1 | 「開始」→ マイク許可 | 「聞き取り中」 |
| A2 | 出題に対し発声 | 正解判定される |
| A3 | 1 発話ごとに認識が終了する | 「再接続」表示の後すぐ「聞き取り中」に戻り、体感で途切れない |
| A4 | 機内モードにして発声 | 「ネットワークエラーが続いている」と表示される |
| A5 | 画面を切り替えて戻る | 認識が再開し、画面消灯抑止も戻る |

### 共通

| # | 手順 | 期待 |
|---|---|---|
| C1 | 「タップして発話」に切り替え → 画面をタップして発声 | タップ後 5 秒だけ聞き取り、正解判定される |
| C2 | 「発声検出」に切り替え → 声を出す | 答えが表示される |
| C3 | 「タップ入力」に切り替え | 数字パッドが出て入力で解答できる |

## 3. 公開前のセキュリティ検証

`docs/mobile-support.md` の「6. セキュリティ上の性質」を、以下のコマンドで確認する。

```sh
# S1 機密情報
grep -rniE 'api[_-]?key|secret|token|password|bearer|AKIA|ghp_|github_pat|-----BEGIN' . --exclude-dir=.git
# S3 外部通信・外部リソース
grep -rniE 'fetch\(|XMLHttpRequest|WebSocket|sendBeacon|https?://' index.html
# S4 永続化
grep -rniE 'localStorage|sessionStorage|indexedDB|document\.cookie' index.html
# S5 XSS 経路
grep -rniE 'innerHTML|outerHTML|insertAdjacentHTML|eval\(|new Function|document\.write' index.html
# S6 権限
grep -n 'getUserMedia' index.html
```

`index.html` については S1・S3・S4・S5 がすべてヒットゼロ、S6 は `audio` のみであること。
