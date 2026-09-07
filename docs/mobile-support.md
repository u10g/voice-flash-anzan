# モバイル対応の根拠と設計判断

`index.html` がモバイルで採っている実装方針と、その根拠。

## 1. Web Speech API の対応状況

MDN browser-compat-data（`api/SpeechRecognition.json`）より。

| 項目 | Safari / iOS Safari | Chrome Android | Chrome desktop |
|---|---|---|---|
| `SpeechRecognition` | 14.1（`webkit` 前置）＝ iOS は 14.5 相当 | 対応（`webkit` 前置） | 139 で無前置、`webkit` は 33 |
| `continuous` | Safari 17 以降。14.1〜16 は partial | **未対応（設定はできるが効果なし）** | 33 |
| `interimResults` | 14.1 | 対応 | 33 |
| `processLocally` / `install()`（オンデバイス認識） | 未対応 | **未対応** | 139 |
| `phrases`（認識バイアス） | 未対応 | **未対応** | 142 |

### 実装への反映

- **`onend` からの再接続を前提にする。** Android Chrome では `continuous` が効かず 1 発話ごとに認識が終了する。iOS も途中で切れる。
  → `startSR()` は `onend` で再起動する。ただし無条件の即時再起動は失敗ループになるため、
  **起動から 500ms 未満で終了した場合のみ失敗とみなし、250 → 500 → … → 4000ms の指数バックオフ**をかける。
  結果を受信するか 500ms 以上継続したらバックオフをリセットし、失敗が 5 回続いたら発声検出／タップ入力へ切り替える。
- **オンデバイス認識は使えない。** モバイルでは `processLocally` が未対応のため、音声は Apple / Google のサーバへ送られる（オフライン不可）。README に明記している。
- **認識バイアス（`phrases`）も使えない。** 数字の認識精度は API 側の実装に依存するため、
  受け取った文字列側で吸収する（下記 3）。

## 2. iOS 固有の問題への対処

| 問題 | 対処 |
|---|---|
| 音の再生と認識が競合し、認識が止まる | 音は `HTMLAudioElement` ではなく `AudioContext` で鳴らす。さらに `beep()` が `audioBusyUntil` を設定し、**再生終了 + 350ms は認識の再起動を遅らせる** |
| 消音スイッチで合図音が鳴らない | `navigator.audioSession.type` を、聞き取り中は `play-and-record`、停止時は `auto` に設定（対応環境のみ） |
| ホーム画面に追加した PWA（standalone）で認識が動かない報告 | **PWA manifest を追加しない**。standalone 起動を検出したら「Safari で開く」旨のバナーを表示 |
| Safari 以外の iOS ブラウザ（WKWebView）で使えない | UA から検出してバナー表示。実際に失敗したら自動フォールバック |

## 3. 日本語の数値解釈

音声認識の返す文字列は端末によって算用数字・漢数字・かなが混ざる。`parseNums()` で次を扱う。

- 全角数字 → 半角化、カンマ・空白・読点の除去
- 算用数字（`15`）
- 漢数字（`十五`、`二十三`）
- かな読み（`じゅうご`、`にじゅうさん`）。カタカナはひらがなへ正規化
  - 「に」「し」「ご」「く」は助詞や語の一部として頻出するため、**2 文字以上に一致した並びだけ**を数値として採用する
- `マイナス` / `ひく` / `-` の**直後に現れた数値だけ**符号を反転する（文中にその語があるだけで全候補を反転させない）
- `maxAlternatives = 3` を設定し、第 1 候補以外の認識結果も照合する

中間結果（`interimResults`）で答えと一致したら即座に正解とし、不一致の判定は確定結果（`isFinal`）でのみ行う。中間結果で誤判定のブザーが鳴るのを避けるため。

## 4. レイアウトと端末操作

- 高さは `100dvh`（`100vh` をフォールバックとして先に記述）。iOS のツールバー伸縮で表示が切れるのを防ぐ
- ボタンは `touch-action: manipulation`（ダブルタップズーム抑止）、`-webkit-tap-highlight-color: transparent`、最小 44px
- 入力欄の `font-size` は 16px（iOS の自動ズーム防止）
- モバイルでは設定パネルを初期状態で閉じる
- Wake Lock は `visibilitychange` で復帰時に取り直す
- タブが非表示になったら認識を停止し、復帰時に再開する

## 5. 公開基盤（GitHub Pages）

- **Free プランでは public リポジトリのみ**が Pages の対象（private は Pro / Team / Enterprise が必要）
- `github.io` ドメインは **HTTPS で自動配信**。`getUserMedia` と `SpeechRecognition` が要求する secure context を満たす
- 制限: サイト 1 GB、帯域 100 GB/月（ソフト）、デプロイは 10 分でタイムアウト。単一 HTML のため影響なし
- デプロイは `.github/workflows/pages.yml`（公式 `actions/*` のみ、権限は `contents: read` / `pages: write` / `id-token: write` に限定）

## 6. セキュリティ上の性質

| 観点 | 状態 |
|---|---|
| サーバ側処理 | なし（静的 HTML のみ） |
| 外部通信 | なし。CSP の `connect-src 'none'` で機構的にも禁止 |
| 外部依存 | なし（CDN・Web フォント・アナリティクスを使わない） |
| 保存データ | なし（Cookie / localStorage / IndexedDB を使わない） |
| DOM への値の挿入 | すべて `textContent`。`innerHTML` / `eval` / `document.write` は使わない |
| 要求する権限 | マイクのみ。ユーザー操作起点でのみ要求 |

## 参考

- [MDN: SpeechRecognition](https://developer.mozilla.org/en-US/docs/Web/API/SpeechRecognition) / [Using the Web Speech API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Speech_API/Using_the_Web_Speech_API) / [browser-compat-data `api/SpeechRecognition.json`](https://github.com/mdn/browser-compat-data/blob/main/api/SpeechRecognition.json)
- [crbug 41297427: Android で `continuous` が効かない](https://issues.chromium.org/issues/41297427) / [Web Speech API in Android WebView](https://issues.chromium.org/issues/40417848)
- [WebAudio/web-speech-api#96: Safari の挙動](https://github.com/WebAudio/web-speech-api/issues/96) / [Apple Developer Forums: webkitSpeechRecognition on iOS](https://developer.apple.com/forums/thread/699881) / [同: interimResults in Safari iOS](https://developer.apple.com/forums/thread/775699)
- [iOS Safari で SpeechRecognition と音声再生を同時に使えない問題（zenn）](https://zenn.dev/takex5g/articles/e3c445810ea085)
- [What PWA Can Do Today: Speech Recognition](https://whatpwacando.today/speech-recognition/)
- [GitHub Pages limits](https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits) / [Securing your GitHub Pages site with HTTPS](https://docs.github.com/en/pages/getting-started-with-github-pages/securing-your-github-pages-site-with-https)
