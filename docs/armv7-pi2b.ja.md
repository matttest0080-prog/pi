# Raspberry Pi 2 Model B（ARMv7）で pi を動かす

> 言語／Language：[繁體中文](armv7-pi2b.md) ｜ [English](armv7-pi2b.en.md) ｜ [简体中文](armv7-pi2b.zh-CN.md) ｜ **日本語** ｜ [한국어](armv7-pi2b.ko.md) ｜ [Español](armv7-pi2b.es.md)

**結論：動きます。公式 npm パッケージ v1.5.2 を実機 Pi2B で検証しました。** 公開済み npm パッケージの実行に agent のコード変更は不要です。このブランチの小さな変更 2 か所は、「armv7 でソースからビルドする」場合と「将来 armv7 用ネイティブ helper を作る」場合がアーキテクチャで止まらないようにするものです（下記「このブランチの変更」）。v1.5.0（2026-11-04）と今回の v1.5.2（2026-11-16）の隔離インストーラー受け入れ検査は、どちらも 11/11 合格です。

- 対象ボード：Raspberry Pi 2 Model B（armv7l、4×900 MHz、921 MiB RAM）、Raspbian GNU/Linux 13 (trixie)
- 最新の実機テスト：**2026-11-16、公開済み `@earendil-works/pi-coding-agent@1.5.2`**。隔離インストーラー検査 11/11 合格、加えてローカルモデル要求をテスト。
- 同期状態：fork の `main` は upstream 安定版 **v1.5.2** と公開後の Kimi K3 テスト修正を含みます。未公開の upstream `main` の追加コミットは含みません。
- v1.5.2 レポート：[`armv7-pi2b-verification-v1.5.2.md`](armv7-pi2b-verification-v1.5.2.md)。旧ログ [`armv7-pi2b-verification.md`](armv7-pi2b-verification.md) は v1.5.0（2026-11-04、11/11 合格）の記録です。
- v1.5.2 のモデル検証範囲は CLI 起動とローカルモデル要求で、**ツール呼び出しとソースビルドは未検証**。

---

## 1. 決定的な 3 つの事実

| 事実 | 根拠 |
| --- | --- |
| **Node 22 には公式 armv7l ビルドがあり、Node 23 以降には無い** | `nodejs.org/dist/index.json`：v22 は **35/35** すべてのリリースが `linux-armv7l` を含む（22.19 以上の 11 版を含む、最新 v22.23.2）。直近 12 リリース（v23 以降）には無い。このリポジトリは `engines.node >= 22.19.0` を要求 → **v22 系でのみ満たせる** |
| **公式のプリビルドバイナリに armv7 は無い** | リリース資産は `linux-x64` / `linux-arm64` / `darwin-x64|arm64` / `windows-x64|arm64` のみで、`build-binaries.yml` のプラットフォーム判定も同じ 4 つだけ → 公式 tarball は**使えない** |
| **npm パッケージ経路にネイティブの障害は無い** | `@earendil-works/pi-coding-agent@1.5.2`：**`os`/`cpu` 制限なし**、ビルド済み CLI 同梱（`dist/bundle/*.js`）、依存は純 JS か wasm（`photon-node` が `photon_rs_bg.wasm` を内包）、`canvas` は `devDependencies` のみ、セッション backend は Node 内蔵の `node:sqlite` |

唯一の機能低下：TUI のネイティブ X11／クリップボード helper（`linux-platform-x11.node`）は
x64/arm64 のプリビルドしか無く、ローダー `packages/tui/src/native-platform.ts` はそれ以外の
アーキテクチャで `undefined` を返します。`packages/tui/native/linux/README.md` 自身が
"Coding-agent falls back to command-line tools when native reads are unavailable" と書いており、
armv7 ではこれが**設計上の許容された縮退**です（クラッシュはせず、クリップボード／画像連携だけが失われます）。

導入済み 1.5.0 を使った実機計測：

```text
process.arch = arm, process.platform = linux
getNativeClipboard()      -> undefined
getNativePlatformHelper() -> undefined
```

## 2. v1.5.0 の過去の受け入れ検査結果（2026-11-04）

| 検査項目 | 実測値 |
| --- | --- |
| アーキテクチャ | `armv7l`（Raspbian 13 trixie） |
| 実行ユーザー | uid 1000、**sudo は一切不使用**（`$HOME` にインストール） |
| Node | **v22.23.2**（公式 `linux-armv7l` tarball、26,338,176 bytes） |
| `node:sqlite` | **利用可** → セッション backend にネイティブモジュール不要 |
| CLI | **`pi --version` → 1.5.0**、`pi --help` も正常出力 |
| npm インストール | `changed 123 packages in 4m`（npm 10.9.8、1.4.0 → 1.5.0 の再インストール検証。初回更新は `added 2 packages, and changed 121 packages in 3m`） |
| 同梱プリビルド | `darwin-arm64|x64`、`linux-arm64|x64`、`win32-arm64|x64` のみ —— **`linux-arm`（armv7）は無い** |
| 起動時間 | `pi --version` **5.35 秒**（検証時）、再測定 4.69–6.06 秒 |
| メモリ | `node` の基準 RSS **39 MiB**、`pi --version` 子プロセスのピーク 77 MB（空き 587 MiB、OOM なし） |
| ディスク | Node 187 MB ＋ pi パッケージ 158 MB（`$HOME` の空き 1.2 GB） |

> 実際のモデル 1 ターン：**2026-10-01 に板の上で実機検証済み、2026-10-27 に 1.4.0、2026-11-04 に 1.5.0 で再検証済み**です（板自身が `llama-server` を
> 動かし、PC も外部 API も不要）。手順と実測値は第 8 節。pi 0.99.2 の内蔵ローカル経路は
> `LLAMA_BASE_URL` で指す llama.cpp router のままです（LM Studio の OpenAI 互換 API は使いません）。

## 3. インストールと使い方

Pi2B 上で（**sudo 不要**、すべて `$HOME` に入ります）：

```sh
# 1) Node 22 armv7l（公式 tarball。NodeSource や v23 以降は使わない）
curl -fsSLO https://nodejs.org/dist/v22.23.2/node-v22.23.2-linux-armv7l.tar.xz
mkdir -p ~/opt && tar -xJf node-v22.23.2-linux-armv7l.tar.xz -C ~/opt
ln -sfn ~/opt/node-v22.23.2-linux-armv7l ~/opt/node22

# 2) CLI
export PATH="$HOME/opt/node22/bin:$HOME/.local/bin:$PATH"
node -v                      # v22.23.2
npm install -g --prefix ~/.local @earendil-works/pi-coding-agent
pi --version                 # 1.5.2

# 3) 永続化（~/.bashrc に追記）
echo 'export PATH="$HOME/opt/node22/bin:$HOME/.local/bin:$PATH"' >> ~/.bashrc
```

**ソースから**ビルドする場合（このリポジトリ自身）：

```sh
git clone https://github.com/matttest0080-prog/pi.git && cd pi
npm ci && npm run build      # armv7 では下記の build.sh 変更が必要
```

> このソースビルド経路は、ボード上で端から端まで**検証していません**。実機で計測したのは
> アーキテクチャ判定の部分そのものです：上流の `packages/tui/native/linux/build.sh` は **1** で終了
> （`Unsupported Linux architecture: armv7l`）、この fork の版はスキップのメッセージを出して **0** で終了。
> その先（921 MiB のボードで TypeScript モノレポ全体をビルド）は未検証です。

このブランチには実機で検証した 3 本のスクリプトと自己テストが付属します（`scripts/pi2-armv7/`）：

```sh
scripts/pi2-armv7/install-pi-on-pi2.sh              # Pi 側：ゲート → インストール → 検査 → 計測
scripts/pi2-armv7/install-pi-on-pi2.sh --dry-run    # 手順を表示するだけで何も変更しない
scripts/pi2-armv7/install-pi-on-pi2.sh --check-only # 現状を調べるだけでインストールしない
scripts/pi2-armv7/pi2-pi-agent.sh probe|install|verify|all   # PC から SSH 経由で操作
scripts/pi2-armv7/selftest.sh                       # ツール自身の自己テスト（負のパスも含む）
```

これらは意図的に **fail-closed** です：`armv7l` でない、root で実行、Node が v22 でない、
armv7 専用プリビルドが現れた、ログに v22 以外の Node の痕跡がある、いずれかの手順が失敗——の
いずれでも明確に拒否し、非 0 で終了します。

## 4. このブランチの変更点

**CLI を動かすのにコード変更は一切不要です**（実機の結果がその証明です）。以下の 2 か所は
「armv7 でソースからビルドする」「将来本当に armv7 ネイティブ helper を作る」場合が止まらないように
するための**任意の有効化変更**で、いずれも理由をコメントで残しています：

| ファイル | 変更 | 理由 |
| --- | --- | --- |
| `packages/tui/native/linux/build.sh` | `armv7l`／`armv6l` では明示メッセージを出して **exit 0**（コンパイルしない） | 元の `case` は `x86_64`／`aarch64` しか知らず、他は `exit 1` → Pi2 では `npm run build` が失敗する。armv7 にプリビルドは無くローダーも要求しないので、「明示的にスキップ」が「ワークスペース全体のビルド失敗」より正しい |
| `packages/tui/src/native-platform.ts` | ローダーのアーキテクチャ判定が `arm` も受け入れる | **自分でビルドした** armv7 helper を持つ人が実際に使えるようにするため。helper が無ければ挙動は従来と同一（`require` が失敗し `undefined` を返す）。注意：npm パッケージはビルド済み JS なので、この変更は**再ビルド後**にのみ効きます |

どちらも x64／arm64 の挙動には影響しません。

## 5. 元に戻す

```sh
# pi の部分だけを削除（~/.local は同じボード上の hermes など他ツールと共有している可能性がある）
rm -rf ~/.local/lib/node_modules/@earendil-works ~/.local/bin/pi
rm -rf ~/opt/node22 ~/opt/node-v22.23.2-linux-armv7l
```

## 6. この検証の再現

```sh
scripts/pi2-armv7/selftest.sh                                   # ツールの自己テスト（PC 上、ボードは触らない）
scripts/pi2-armv7/pi2-pi-agent.sh all                            # probe → install → verify
scripts/pi2-armv7/pi2-pi-agent.sh verify docs/armv7-pi2b-verification.md
```

## 7. 注意点と、この作業で踏んだ落とし穴

- **インストール中に見つかった 2 つの実バグは、いずれもゲートが止めたもので事後発見ではなかった**：
  公式 Node tarball は展開後のディレクトリ名に `v` が残る（`node-v22.23.2-linux-armv7l`。最初は `v` を
  落として失敗）。また受け入れ条件を最初「`node_modules` に `.node` を一切含めない」と書いたのは誤りで、
  パッケージは他プラットフォームのプリビルドを正当に同梱します。正しい不変条件は
  「**armv7 専用プリビルドが無い** *かつ* CLI が起動する」です。
- **1.4.0 のパッケージ メタデータも再確認済み**：`os`/`cpu` 制限なし、`pi-tui` のプリビルドは依然
  x64/arm64 の 6 つのみ（`linux-arm` なし）。
- **上流への同期は ARMv7 の作業に影響しません**：上流 `fbad9bea7`（157 コミット、v1.3.0 → 1.4.0）を
  マージした後も、ARMv7 の delta ファイル（ドキュメント 12 本、スクリプト 4 本、README の章、
  コード変更 2 箇所）はバイト単位で同一です。今回上流が変更したのは `.gitignore` だけで、
  fork の `scripts/pi2-armv7/cache/` 行はそのまま残っています。

## 8. 板の上だけで完結する実モデル 1 ターン（実機検証済み、2026-10-01）

pi 0.99.2 は Pi 2B 上で、**モデルもサーバーも板自身で動かして**実際の 1 ターンを完了しました（PC も
外部 API も不要）：

- **`llama-server` のクロスビルド**：板に cmake がなく、ホストにクロスツールチェーンがない場合は、
  Ubuntu の armhf-cross `.deb` を `apt-get download` ＋ `dpkg -x` で `$HOME` に展開します（sudo
  不要）。`as` には `LD_LIBRARY_PATH=<prefix>/usr/lib/x86_64-linux-gnu` が必要で、リンクには
  `--sysroot=<prefix>` と `-static-libstdc++ -static-libgcc` を付けます。実測 13.5 MB、板上の
  `llama-server --version` は `0.3.0-dev (build 10734)`。
- **router モードで起動**：`llama-server --models-preset <ini>`。`--models-preset` が必須です
  （`source=preset` になる）。`--models-dir` は `source=models_dir` を返し、pi の
  `modelIsSelectable()` に除外されます（症状：`/models` には出るのに pi は Unknown provider）。
- **`~/.pi/agent/models.json` の `contextWindow` は「プロンプト + 最大出力 + 4096」より大きく**：
  `clampMaxTokensToContext()` が `contextWindow − プロンプト − 4096` を計算し下限 1 で頭打ちに
  するため、小さく宣言すると pi は `max_completion_tokens: 1` を送り、1 トークンで
  `finish_reason=length` になります。本実測は 8192 宣言／出力上限 512／router `ctx-size` も 8192。
- **他の 2 つの落とし穴**：`auth.json` のサーバー URL は `LLAMA_BASE_URL` より**優先**されます。
  pi は stdin が TTY でないと stdin を読み取るため、リモートでは `< /dev/null` を付けてください。
- **実測値**（`-t 4`、8K ctx、q8_0 KV）：270M Q4 はプロンプト 10.95 tok/s、生成 2.76 tok/s、1 ターン
  46 秒。0.5B Q4 は初回 77 秒（`stopReason: stop`）、ウォーム後 16 秒。llama-server の RSS 530 MiB、
  板の空き 636 MiB。
- **v1.5.2 のボード内モデル要求（2026-11-16）**：一時 prefix/HOME にインストールし、既存の全域 1.5.0 は変更していません。ボード上の loopback llama.cpp router、Qwen2.5-0.5B Q4、8K context、`maxTokens: 64`、隔離した `models.json` を使い、`--no-tools --no-session` で実行。コールド要求は 52 秒で `OK.`、ウォーム要求は 10 秒で `no reply.`。両方 `rc=0`、`stopReason: stop`。終了後の空き RAM は約 675 MiB。静的な OpenAI-compatible ローカル endpoint 設定を使っており、llama.cpp の動的モデル探索ではありません。要求／応答経路の検証であり、ツール使用や回答品質の検証ではありません。テスト後に一時ファイルと server を削除しました。
- **1.5.0 の板内ターン（2026-11-04）**：`-nt`＋短い system prompt＋`maxTokens: 64` で、コールド 95.5 秒
  （モデル呼び出し 84.8 秒）、ウォーム 8.36 秒（1.83 秒）。出力 `OK`、`stopReason: stop`。既定のツール
  セット（プロンプト 1,326 token、出力上限 512）は 330 秒で未完了（1.4.0 のコールドは 638 秒で、
  prefill 律速）。Pi 2B では `-nt`／ツール許可リストと小さな `maxTokens` を使ってください。
- 注意：検証できたのは**経路**であり、小型モデルの事実精度ではありません（0.5B の回答は誤りでした）。
