# 8bitme 設計書 v1

- リポジトリ: github.com/Saber5656/8bitme（MIT License 仮定）
- 作成日: 2026-07-05
- ステータス: 実装前設計（コード未着手）

---

## 1. コンセプト

8bitme は、家族写真や人物写真をブラウザ上でドット絵・レトロゲーム風アバターに変換する完全ローカル処理の OSS ツールである。
写真は一切サーバーに送信されず、すべての変換処理がユーザーのデバイス内（ブラウザの Canvas 上）で完結する。
技術知識ゼロの家族層でも「ページを開く → 写真を選ぶ → プリセットを押す → 保存」の 4 ステップで、
ゲームボーイ風・ファミコン風などの懐かしいアバターを作れる体験を提供する。
v1 は生成 AI を使わないアルゴリズミック変換（ダウンサンプリング + パレット量子化 + ディザリング）に絞り、
「速い・軽い・どこでも動く・写真が外に出ない」を最大の価値とする。

---

## 2. v1 スコープ

### 入れる機能

| 機能 | 内容 | 理由 |
|---|---|---|
| 画像入力 | ファイル選択 / ドラッグ&ドロップ（JPEG / PNG / WebP） | 最小限の入口 |
| 正方形クロップ UI | ドラッグで顔の位置に枠を合わせる簡易クロップ | アバター用途の中心機能。顔検出の代替を人力で担う |
| レトロ機プリセット | GB 風 4 色緑 / 8bit コンソール風 / PICO-8 16 色 / 1-bit 白黒 / CGA 風 / 自動パレット | ワンタップで「らしさ」が出る v1 の面白さの核 |
| ピクセルサイズ調整 | 出力解像度（32 / 48 / 64 / 96 / 128 px）のスライダー | 「粗さ」の調整は体験の中心 |
| ディザリング切替 | なし / Bayer（ordered）/ Floyd–Steinberg | 写真→ドット絵の品質を大きく左右する |
| 前処理 | 彩度・コントラスト自動ブースト（プリセット内蔵、上級者向けに微調整可） | ドット絵らしい発色に必須 |
| Before/After 比較スライダー | 元写真と変換結果を重ねてドラッグ比較 | 「面白い」と感じさせる仕掛け |
| PNG 出力 | nearest-neighbor 整数倍拡大、サイズプリセット（512 / 1024 / SNS アイコン用） | 主目的の成果物 |
| EXIF 除去 | 出力 PNG に位置情報・撮影情報を一切含めない（Canvas 再描画で構造的に除去） | プライバシー必須要件 |
| PWA / オフライン動作 | Service Worker でキャッシュし、機内モードでも動作 | 「ローカル完結」の体験的証明 |
| 多言語 | 英語 + 日本語の 2 言語 | OSS として英語必須、作者圏向けに日本語 |

### 入れない機能（v2 以降）

| 機能 | 振り分け | 理由 |
|---|---|---|
| 生成 AI スタイル変換（SD 系） | v2 検討（採否含め再判断） | 後述 §2.1 |
| 顔検出による自動クロップ（MediaPipe） | v2 | WASM ~6MB のロードが初回体験を重くする。v1 は手動クロップで十分代替可能。MediaPipe Tasks（tasks-vision）はブラウザ内オンデバイス推論が実証済みで、v2 での追加は技術的に低リスク |
| GIF 出力（パレットサイクル・ドット出現アニメ） | v2 | 「面白い」強化枠だが v1 の出荷速度を優先 |
| フレーム装飾（ゲーム画面風 UI 枠、ステータスバー風） | v2 | アセット制作コストが高い。v1 はシンプルな枠 1 種のみ検討 |
| バッチ変換（複数枚一括） | v2 | 家族全員分を作る需要はあるが UI が複雑化する |
| HEIC 入力対応 | v1.x | Chrome/Firefox は HEIC を非対応。libheif の WASM 追加はサイズ増となるため、v1 は非対応形式検出時に「iPhone の設定または共有時に JPEG に変換する方法」を案内する |
| CLI 版 / npm パッケージ化 | v2 | コアパイプラインを純粋関数として分離しておき、将来の切り出しを容易にする（§6） |

### 2.1 判断: v1 はアルゴリズミック変換に絞る（AI 変換は v2 検討）

**結論: 妥当。v1 は AI を使わない。**

| 観点 | アルゴリズミック（v1 採用） | 生成 AI（SD + LoRA 等） |
|---|---|---|
| ローカル実行の敷居 | Canvas API のみ。全ブラウザで即動作、追加 DL 0 MB | モデル 1〜4 GB 級の DL、WebGPU 必須、非対応端末多数。家族層のスマホでは実質動かない |
| 処理時間 | 数十 ms〜1 秒未満 | 端末性能次第で数十秒〜数分、または不能 |
| 決定性・調整可能性 | パラメータで完全制御、同じ入力→同じ出力 | ガチャ性があり「顔が別人になる」失敗が起こる。家族写真では致命的 |
| プライバシー説明 | 「JS だけで完結」と一目で説明可能 | ローカルでも「AI に写真を渡す」ことへの心理的抵抗が残る |
| 実装・保守コスト | 個人 OSS で十分維持可能 | モデル配布・ライセンス・更新の負担が大きい |
| 品質 | 「本物のドット絵師」には及ばないが、レトロ機パレット + ディザで十分「らしく」なる（pyxelate 等の先行 OSS で実証済み） | 高いが上記コストに見合うかは v1 の反応を見てから判断すべき |

先行 OSS の pyxelate（Python）は、ダウンサンプリング + パレット学習 + ディザリングの構成で
HN 等でも好意的に受容された実績があり、アルゴリズミック変換だけでもプロダクトとして成立することを示している。
v1 で「面白い」を担保する工夫は §7 に記載。AI 変換は v1 の利用実態（Issue / Star / 要望）を見て v2 で採否を再判断する。

---

## 3. 配布形態の判断

| 観点 | 静的 Web アプリ（採用） | Tauri デスクトップ | CLI |
|---|---|---|---|
| 非開発者の導入コスト | ◎ URL を開くだけ。インストール不要、スマホ OK | △ DL + インストール。macOS は署名/公証（年 $99 の Developer Program）が無いと Gatekeeper 警告で家族層は脱落 | × 対象外レベル |
| ローカル完結の担保 | ○ 全処理クライアントサイド。オフライン動作・DevTools・CSP で検証可能（§8） | ◎ ネイティブでオフライン | ◎ |
| 「本当に送ってない？」の説明しやすさ | ○ 「機内モードでも動く」を体験で示せる | ○ | ○ |
| マルチプラットフォーム | ◎ macOS / Windows / iOS / Android すべてブラウザで | △ ビルド・配布・署名をプラットフォーム毎に維持 | △ |
| 個人 OSS の維持コスト | ◎ GitHub Pages で無料・CI 1 本 | × リリース作業が重い | ○ |
| 処理性能 | ○ 本用途（〜128px 出力）には Canvas で十分 | ◎ | ◎ |
| スマホ対応 | ◎ 家族層の主端末はスマホ。ここが決定打 | × | × |

**結論: 完全クライアントサイドの静的 Web アプリ（GitHub Pages 配信、PWA 対応）を v1 の唯一の配布形態とする。**

- 決定打は 2 点: (1) 家族層の主端末はスマホであり Web 以外は届かない、(2) 個人 OSS で macOS 署名・公証や マルチ OS バイナリ配布を維持するのは非現実的。
- 「Web = アップロードされそう」という直感的不安は、§8 のプライバシー設計（オフライン実証 + CSP + 検証手順の明示）で技術と説明の両面から打ち消す。
- コア変換ロジックは UI 非依存の純粋 TypeScript モジュールとして分離し、v2 での CLI / npm / Tauri 展開の余地を残す。

---

## 4. 対応プラットフォーム

| 区分 | 対象 | 備考 |
|---|---|---|
| デスクトップブラウザ | Chrome / Edge / Firefox / Safari の各最新 2 メジャー | 開発環境は macOS だが成果物はブラウザ依存のみ |
| モバイルブラウザ | iOS Safari 16+ / Android Chrome | 家族層の一次ターゲット。タッチ操作前提で UI 設計 |
| オフライン | PWA（ホーム画面追加可、Service Worker キャッシュ） | 初回アクセス後は完全オフラインで動作 |
| 非対応と明記 | IE、旧 Android WebView | サポート表明しない |

必要 API: Canvas 2D / `createImageBitmap` / File API / Web Workers / Service Worker。いずれも上記ブラウザで安定。WebGPU / WASM は v1 では不要。

---

## 5. 技術選定

| 領域 | 選定 | 理由 |
|---|---|---|
| 言語 | TypeScript | 型でパイプラインの各段の入出力（ImageData 契約）を固定。OSS コントリビュートも受けやすい |
| ビルド | Vite | 静的サイト生成が軽量・高速。GitHub Pages 向け出力が容易 |
| UI | Preact（+ 素の CSS） | React 互換 API で ~4KB。バンドルを小さく保ち初回ロードを速くする。状態は単純なので状態管理ライブラリ不要 |
| 画像処理 | 自前実装（Canvas 2D + `ImageData` を Web Worker 内で操作） | ダウンサンプル・median-cut 量子化・k-means・Bayer / Floyd–Steinberg ディザはいずれも 100〜200 行級の古典アルゴリズム。外部ライブラリ（image-js 等）を入れるよりバンドルが小さく、教育的価値（OSS としての読みやすさ）も高い |
| EXIF | 読み取りのみ軽量自前実装（Orientation タグのみ）または exifr の tree-shake 利用 | 必要なのは回転補正だけ。出力側は Canvas 再描画により構造的に全メタデータが落ちる |
| PWA | vite-plugin-pwa | Service Worker 生成の定番 |
| テスト | Vitest + 基準画像スナップショット比較 | パイプラインが純関数なので画素単位の回帰テストが可能 |
| CI/CD | GitHub Actions → GitHub Pages | push 時に build + deploy |
| （v2 予約）顔検出 | MediaPipe Tasks（@mediapipe/tasks-vision, WASM） | ブラウザ内オンデバイス推論の実績があり外部送信なしの原則を維持できる。WASM ~6MB のため遅延ロード必須 |

### パレットの権利について（調査結果）

- カラーパレット（RGB 値の集合）はハードウェア仕様に由来する事実・数値データであり、それ自体は著作権保護の対象と考えるのが一般的。Wikipedia「List of video game console palettes」をはじめ、Lospec 等のコミュニティや多数の OSS ツールが GB / NES / PICO-8 パレットを公開・利用しており、パレット値の使用は実務上問題ないと判断する。PICO-8 は作者（Lexaloffle）がパレット利用を明示的に許容している。
- ただし **「Game Boy」「Nintendo」「Famicom」等は商標**。UI・README では商標名を製品名として使わず、記述的な表現にする:
  - `GB Green` → **"Handheld Green (4 colors)"**
  - `NES` → **"8-bit Console (retro TV palette)"**
  - `PICO-8` → 作者が名称利用に寛容なため "PICO-8 16" と表記可（要 README でのクレジット）
  - 「〇〇風」であること、任天堂等と無関係であることを README に免責として 1 行記載

---

## 6. アーキテクチャ

### 全体構成

```
[UI (Preact)] ── postMessage ──> [Web Worker: 変換パイプライン (純TS)]
     │                                    │
     └── <canvas> プレビュー <── ImageData ┘
core/  … パイプライン純関数群（DOM 非依存。将来 CLI / npm に切り出し可能）
app/   … UI・Worker 接続・PWA
```

- 変換は必ず Web Worker で実行し、スマホでも UI をブロックしない。
- `core/` は `(ImageData, PipelineOptions) => ImageData` の純関数合成のみで構成し、単体テスト・回帰テストを画素比較で行う。

### 変換パイプライン

```
入力ファイル
  │ 1. デコード + EXIF Orientation 補正（createImageBitmap / 自前 Orientation 読取）
  ▼
クロップ（ユーザー操作: 正方形 or 1:1 以外のプリセット比率）
  │ 2. 作業解像度へ縮小（長辺 512px 程度。以降の処理コスト一定化）
  ▼
前処理
  │ 3. 彩度ブースト(+15〜30%) + コントラスト強調(S字トーンカーブ)
  │    ※ドット絵は原色寄りの発色が「らしさ」の核。プリセット毎に係数を持つ
  │ 4.（任意）軽いアンシャープマスクで輪郭を立てる（後段の縮小で線が残る）
  ▼
ダウンサンプル
  │ 5. 目標ピクセル数（例 64x64）へブロック平均で縮小
  │    ※単純 nearest だとノイズを拾う。ブロック平均が安定。
  │    v1.x で「エッジ優先サンプリング」（pyxelate 系の勾配考慮）を検討
  ▼
パレット量子化
  │ 6a. 固定パレット系プリセット: 各画素を知覚距離（重み付き RGB or Oklab）で最近色へ
  │ 6b. 「自動パレット」プリセット: median-cut（v1）で画像から 8/16 色を生成
  │    ※k-means は品質向上余地として v1.x。GB 系は輝度→4 階調マップ後に緑階調へ
  ▼
ディザリング
  │ 7. なし / Bayer 4x4（均一で「レトロ機らしい」規則パターン）
  │    / Floyd–Steinberg（写真の肌・グラデに強い。誤差拡散）
  │    ※プリセット毎に既定を持つ（例: GB風=Bayer, 自動16色=Floyd–Steinberg）
  ▼
拡大・出力
  │ 8. nearest-neighbor で整数倍拡大（64px → 512px なら 8x）
  │    プレビューは CSS image-rendering: pixelated、保存時は Canvas で物理拡大
  │ 9. PNG エンコード（canvas.toBlob）→ メタデータなしのクリーンな PNG
  ▼
ダウンロード（例: 8bitme_handheld-green_512.png）
```

### 出力仕様

| プリセット | 内部解像度 | 出力サイズ | 用途表記（UI 上） |
|---|---|---|---|
| Small | 選択値（例 64） | 512x512 | 「SNS アイコンに」 |
| Large | 同上 | 1024x1024 | 「印刷・壁紙に」 |
| Dot-perfect | 同上 | 等倍（64x64 等） | 「ドット絵素材として」 |

- 拡大は常に整数倍。端数が出る場合は内部解像度側を調整して整数倍を維持（ドットの滲み防止）。
- 透過 PNG は v1 では扱わない（背景切り抜きは v2 の顔検出とセットで検討）。

---

## 7. UI/UX

### 画面構成（1 画面完結）

```
┌─────────────────────────────┐
│  8bitme  ドットの中に、家族を。      │
│  [写真はこの端末の外に出ません 🔒]   │← 常時表示バッジ。タップで §8 の説明へ
├─────────────────────────────┤
│   ①「しゃしんをえらぶ」(大ボタン)     │← D&D も可
│   ② クロップ: 「顔に枠を合わせてね」   │
│   ③ プレビュー（Before/After スライダー│
│      をドラッグで比較）              │
│   ④ スタイル: [🟩4色] [🕹8bit] [🎨16色]│← 横スクロールのカード。結果サムネ付き
│      こまかさ: ●──── (あらい↔こまかい) │
│      しあがり: [なめらか][レトロ点々]    │← ディザの言い換え
│   ⑤ [保存する] → サイズ選択          │
└─────────────────────────────┘
```

### 非開発者向けの言葉選び

| 技術用語 | UI 表記（日本語） | UI 表記（英語） |
|---|---|---|
| 解像度 / ピクセル数 | こまかさ | Detail |
| ディザリング | しあがり（なめらか / レトロ点々） | Texture (Smooth / Retro dots) |
| パレット量子化 | 色のスタイル | Color style |
| EXIF 除去 | 位置情報などは保存されません | Location data is never saved |
| ローカル処理 | 写真はこの端末の外に出ません | Your photo never leaves your device |

### 初回体験（勝負は 30 秒）

1. ページを開くと**サンプル写真で変換済みのデモ**が最初から表示されている（自分の写真を選ぶ前に「何ができるか」が分かる）
2. 写真選択 → 即座に既定プリセット（PICO-8 16 色 + Floyd–Steinberg）で変換表示。**設定を触らなくても 2 タップで結果が出る**
3. プリセットカードは全て**その人の写真で生成したサムネイル**を表示（選ぶ楽しさ = v1 の「面白い」の中心）
4. Before/After スライダーで見せ合い・SNS 共有の動機を作る
5. 保存後に「他のスタイルも試す」「べつの写真でつくる」を提示

### 「面白い」を作る v1 の工夫（アルゴリズミック変換のみで）

- レトロ機プリセットの**キャラ立ち**（4 色緑は最強の記号性。1-bit はモノクロ写真として意外に映える）
- プリセットサムネの一覧表示 = 「自分の顔のバリエーションが並ぶ」体験そのものが面白い
- Before/After スライダー
- 前処理（彩度・コントラスト強調）で「くすんだ写真がドット絵らしい発色になる」驚き
- ファイル名にスタイル名を含めコレクション性を出す

---

## 8. プライバシー設計

### 原則

1. **画像データを外部に送る通信経路をそもそも作らない**（アップロード API が存在しない）
2. **構造的に証明可能にする**（設定で OFF ではなく、能力として不可能に）
3. **非開発者に体験で伝え、開発者に技術で証明する**

### 担保策

| 層 | 施策 |
|---|---|
| アーキテクチャ | 全処理をブラウザ内 Canvas/Worker で実行。バックエンド・API・外部 SaaS を一切持たない |
| CSP | `connect-src 'self'`（+ 実質 Service Worker キャッシュのみ）、`img-src 'self' blob: data:` を meta タグで宣言。画像を外部ドメインへ送る fetch はブラウザレベルでブロックされる |
| オフライン | PWA として初回ロード後は機内モードで全機能が動作。**「機内モードにして試してください」を UI とREADME に明記**（非開発者向けの最強の証明） |
| 計測ゼロ | Google Analytics 等のトラッカー・外部フォント・CDN を一切使わない（CSP とも整合）。GitHub Pages のアクセスログ以上の情報を持たない |
| OSS | 全コード公開 + ビルドが GitHub Actions 上で行われることで、配信物とソースの対応を検証可能 |
| 検証手順の公開 | README に「DevTools → Network タブを開いて変換・保存を実行 → リクエストが飛ばないことを確認」の手順とスクリーンショットを掲載 |

### EXIF の扱い

- **入力**: EXIF は Orientation タグのみ読み取り（回転補正のため）。GPS 等は読み取らない・保持しない。
- **出力**: `canvas.toBlob()` による PNG 再生成のため、EXIF/GPS/撮影日時等のメタデータは構造的に一切含まれない。これを仕様として README・UI に明記する（「変換後の画像に位置情報は含まれません」）。
- メモリ上の画像データはページを閉じれば消える。localStorage 等への画像保存は v1 では行わない（行う場合も端末内のみと明記）。

---

## 9. 配布方法

| 項目 | 内容 |
|---|---|
| ホスティング | GitHub Pages（`https://saber5656.github.io/8bitme/`）。独自ドメインは任意・後回し |
| デプロイ | GitHub Actions: main への push → Vite build → Pages deploy。手作業ゼロ |
| バージョニング | タグ + GitHub Releases（CHANGELOG）。静的アプリのため「更新 = 再訪で最新」 |
| PWA 配布 | 「ホーム画面に追加」で疑似アプリ化（iOS/Android）。ストア申請はしない |
| ライセンス | MIT。パレット出典（Lospec / PICO-8）と商標免責を NOTICE 節に記載 |
| 将来 | `core/` を npm パッケージ `@8bitme/core` として公開（v2）。Tauri 化は需要が出た場合のみ |

---

## 10. README 構成案（英語）

```markdown
# 8bitme 🕹
Turn family photos into pixel-art, retro-game-style avatars —
100% in your browser. Your photos never leave your device.

[hero: before/after 画像を横並び 1 枚（家族写真 → GB風/16色 の 2 変換例）]
[demo GIF: 写真選択 → プリセット切替 → 保存 の 15 秒画面録画]

**👉 Try it now: https://saber5656.github.io/8bitme/** — no install, no upload, no sign-up.

## Features
- 🎨 Retro presets: Handheld Green (4 colors), 8-bit Console, PICO-8 16, 1-bit, CGA
- 🔍 Before/after comparison slider
- 📱 Works on phones, tablets, desktops — even offline (PWA)
- 🔒 Privacy by design: all processing happens locally in your browser

## Privacy — "How do I know it's local?"
1. Load the page once, then turn on Airplane Mode. Everything still works.
2. Open DevTools → Network tab. Convert and save — no requests are sent.
3. CSP blocks all external connections. Read the source — it's all here.
4. Exported PNGs contain no EXIF/GPS metadata.

## How it works（パイプライン図 1 枚 + 3 行説明）
## Development（clone / npm i / npm run dev の 3 行）
## Roadmap（v2: face detection, GIF export, AI styles — see issues）
## License & Credits
MIT. Palette data via Lospec & community documentation.
Not affiliated with Nintendo or Lexaloffle. PICO-8 palette used with
the author's public permission for community use.
```

ポイント: 冒頭 3 行 + hero 画像 + 「Try it now」の 1 行導線までで意思決定が完結する構成。デモ GIF は README 用最優先アセット。

---

## 11. リスクと実装前検証項目

| 優先度 | リスク / 検証項目 | 内容と検証方法 |
|---|---|---|
| **P0** | **アルゴリズミック変換の品質が期待に届くか** | 最重要。UI を作る前に `core/` パイプラインだけを Node スクリプトで組み、**多様な家族写真 10〜20 枚 ×（5 プリセット × 解像度 32/64/128 × ディザ 3 種）のコンタクトシート**を生成して目視評価する。評価基準: (a) 本人と分かるか (b) 「ドット絵」に見えるか（単なるモザイクに見えないか）(c) 肌トーンの破綻がないか。特に「引きの集合写真の小さい顔」は潰れる想定 → クロップ前提の UX（顔アップ推奨のガイド文言）で回避できるかを確認。基準未達なら前処理（輪郭強調・トーンカーブ）とエッジ考慮ダウンサンプルを先に強化する |
| **P0** | 肌色 × 少色数パレットの破綻 | GB 4 色・1-bit で顔が識別可能か。ディザ既定値（Bayer vs Floyd–Steinberg）をプリセット毎に決めるのはこの検証の出力とする |
| **P1** | iPhone 由来 HEIC が開けない | Chrome/Firefox は HEIC 非対応。ファイル判定して案内メッセージを出す（v1）。libheif-wasm 追加は v1.x で判断。**家族層の写真の過半が HEIC の可能性があり、案内文言の質が離脱率を左右する** |
| **P1** | スマホのメモリ・巨大画像 | 48MP 級の写真で iOS Safari の Canvas 制限に当たらないか。デコード直後に長辺 512px へ縮小する設計で回避 → 実機（古めの iPhone/Android）で確認 |
| **P1** | 「Web = アップロードされる」不安による離脱 | §8 の施策が非開発者に伝わるかを家族 2〜3 名でユーザーテスト（バッジ文言の A/B: 「外に出ません」vs「オフラインでも動く」） |
| **P2** | 商標（Game Boy 等）の名称使用 | §5 の記述的名称ポリシーを README/UI 実装時にレビュー項目化 |
| **P2** | ブラウザ間の Canvas 色差 | カラープロファイル（Display P3 写真）で色がズレる可能性 → sRGB 変換の要否を主要ブラウザで確認 |
| **P2** | PWA キャッシュの更新不全 | 旧バージョンが残り続ける問題 → vite-plugin-pwa の autoUpdate + 「新しいバージョンがあります」トースト |

---

## 12. v1 Issue 分割案（8 個）

- **#1 Validate conversion quality with a headless pipeline spike** — `labels: spike, P0, core`
  UI 実装前に、ダウンサンプル → 量子化 → ディザの最小パイプラインを Node + TS で実装し、サンプル写真からコンタクトシートを自動生成する。§11 P0 の品質判断を行い、プリセット毎の既定パラメータ（前処理係数・既定ディザ）を決定する。
  受け入れ条件: 10 枚以上の多様な人物写真で全プリセットのコンタクトシートが生成され、品質評価メモと既定パラメータ表が Issue に記録されている。

- **#2 Implement core pixelation pipeline as a pure TypeScript module** — `labels: core, v1`
  `core/` に EXIF Orientation 補正・前処理・ブロック平均ダウンサンプル・固定/median-cut パレット量子化・Bayer / Floyd–Steinberg ディザ・整数倍 NN 拡大を純関数として実装し、Web Worker から呼べる形にする。#1 の決定パラメータを反映する。
  受け入れ条件: DOM 非依存で Vitest の画素スナップショットテストが通る。64x64 変換が実機スマホ相当で 1 秒未満。

- **#3 Add retro palette presets with trademark-safe naming** — `labels: core, v1`
  Handheld Green (4色) / 8-bit Console / PICO-8 16 / 1-bit / CGA / Auto-16 の各プリセット（パレット値 + 前処理係数 + 既定ディザ）を定義する。名称は §5 の商標ポリシーに従い、出典クレジットを NOTICE に記載する。
  受け入れ条件: 全プリセットがパイプラインで選択・適用でき、README 用の出典・免責文が用意されている。

- **#4 Build image input with crop UI and format guidance** — `labels: ui, v1`
  ファイル選択 / D&D で JPEG・PNG・WebP を受け付け、正方形クロップ UI（タッチ対応）を提供する。HEIC 等の非対応形式は検出して JPEG 変換の案内を表示する。読み込み時に長辺 512px へ縮小する。
  受け入れ条件: iOS Safari / Android Chrome で写真選択→クロップ→変換まで操作できる。HEIC 選択時に日英の案内が表示される。

- **#5 Build main screen with preset thumbnails and before/after slider** — `labels: ui, v1`
  1 画面 UI（§7）を実装する。プリセットカードにユーザー写真での変換サムネイルを表示し、Before/After 比較スライダー、こまかさスライダー、しあがり切替を提供する。文言は非開発者向け表記（日英）とする。
  受け入れ条件: 写真選択から 2 タップで変換結果が表示される。全変換が Worker 実行で UI が固まらない。

- **#6 Implement PNG export with size presets and metadata-free output** — `labels: core, ui, v1`
  nearest-neighbor 整数倍拡大による 512 / 1024 / 等倍の PNG 出力を実装する。`canvas.toBlob` 再生成により EXIF・GPS を含まないことをテストで保証し、ファイル名にスタイル名を含める。
  受け入れ条件: 出力 PNG にメタデータが存在しないことを自動テストで検証。ドットの滲み（非整数倍拡大）が発生しない。

- **#7 Ship as offline-capable PWA with privacy-proof CSP and Pages deploy** — `labels: infra, privacy, v1`
  vite-plugin-pwa によるオフライン対応、`connect-src 'self'` の CSP、トラッカーゼロ構成、GitHub Actions → GitHub Pages の自動デプロイを設定する。更新トーストも実装する。
  受け入れ条件: 初回ロード後に機内モードで全機能が動作する。DevTools Network で変換・保存時に外部リクエストが 0 件である。main への push で自動デプロイされる。

- **#8 Write English README with hero image and demo GIF** — `labels: docs, v1`
  §10 の構成で README を作成する。before/after hero 画像・15 秒デモ GIF・「Try it now」1 行導線・プライバシー検証手順（機内モード / Network タブ）・商標免責を含める。
  受け入れ条件: hero 画像とデモ GIF が表示され、初見の非開発者がリンク 1 クリックで利用開始できる。プライバシー検証手順が再現可能である。

---

## 参考資料（設計時の調査ソース）

- pyxelate（先行 OSS の品質・手法）: https://github.com/sedthh/pyxelate / HN 反応: https://news.ycombinator.com/item?id=29443721
- Pixelated Image Abstraction（Gerstner 2012, エッジ考慮の学術手法）: https://gfx.cs.princeton.edu/pubs/Gerstner_2012_PIA/Gerstner_2012_PIA_small.pdf
- ディザリング手法比較（Floyd–Steinberg / Bayer / Atkinson）: https://www.ascii-magic.com/blog/complete-guide-to-dithering
- nearest-neighbor と整数倍拡大: https://image-scaler.com/blog/nearest-neighbor-interpolation/
- コンソールパレット一覧: https://en.wikipedia.org/wiki/List_of_video_game_console_palettes / PICO-8 パレット: https://lospec.com/palette-list/pico-8 / https://pico-8.fandom.com/wiki/Palette
- MediaPipe Face Detector for Web（v2 顔検出の実現性）: https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector/web_js
- GitHub Pages の PWA 化: https://christianheilmann.com/2022/01/13/turning-a-github-page-into-a-progressive-web-app/
