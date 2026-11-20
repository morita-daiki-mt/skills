---
name: orca-qa
description: Orca IDE の組み込みブラウザを orca CLI で実際に操作し、修正確認・不具合の再現・モバイル幅の QA を証拠付きで判定する。専用プロファイルとページ ID を使い、結果をケースごとに PASS / FAIL / BLOCKED / SKIP で返す。
---

# Orca ブラウザ QA

[Orca IDE](https://orca.dev/) の組み込みブラウザを Orca CLI で操作する。QA のタブは対象の workspace に作り、既存のタブやログイン状態から分離する。ブラウザ操作は Orca CLI に統一する。Orca の接続や対象の確認ができないケースは BLOCKED として理由を報告する。

## 1. CLI と workspace を確かめる

以降の `ORCA` はこのセッションで使う CLI に置き換える。`ORCA_CLI_COMMAND` が設定されていればその実行ファイルを使う。Orca の開発 checkout で `ORCA_DEV_REPO_ROOT` が設定されていれば `orca-dev`、Orca 管理下にない Linux の通常のシェルでは `orca-ide`、それ以外では `orca` を使う。

```text
ORCA skills get orca-cli
ORCA skills get orca-cli --reference references/browser.md
ORCA status --json
ORCA worktree current --json
```

- 同じ実行ファイルとランタイムを通して操作する。返却された workspace のパスと実行ホストを対象の checkout と照合し、workspace ID を控える。候補が複数ある場合は `worktree show --worktree <selector> --json` で特定する。UI でいま選ばれている workspace を推測で使わない。
- `runtime_access_denied` は Orca への接続が sandbox で拒否された状態。許可された昇格経路で同じコマンドを試し、許可されない場合は BLOCKED とする。このエラーで `orca open` や再起動は行わない。
- SSH やペアリング先を使う場合は、CLI のランタイムとブラウザのホストを確認する。`127.0.0.1` が指すホストはブラウザの配置で変わる。必要な環境セレクターはスキルの全コマンドで揃える。

## 2. ケースと動いている版を揃える

依頼と受け入れ条件から、ケース ID、画面、権限、前提データ、操作、期待結果、必須かどうかを決める。実行前に対象 URL、アプリを起動している checkout・ビルド、テストデータを照合する。不具合修正では修正前の再現証拠を利用し、手元にない場合は未取得と記録する。

既定の対象はローカルの検証 URL。別環境は依頼とリポジトリ規約で対象だと確認できたときに使う。開始前に許可するオリジン（スキーム・ホスト・ポート）を記録する。外部への決済・メール・通知・計測を起こし得る操作は、テスト向けの設定や許可された宛先を確認してから行う。ブラウザのモバイル表示、WebView が表示するページ、実際のアプリ内認証やブリッジは観測できる範囲が異なる。画面やプラットフォームごとに判定する。

## 3. 隔離プロファイルを作り、ページ ID を固定する

```text
ORCA tab profile create --label <qa-run-label> --scope isolated --json
ORCA tab create --worktree <workspace-id> --profile <profile-id> --url <test-url> --json
ORCA tab show --page <page-id> --json
ORCA snapshot --page <page-id> --interactive --json
```

- `tab profile create` の `result.profile.id` と `tab create` の `result.browserPageId` を記録する。`tab show` で許可オリジンを確認してから snapshot に進む。個々のタブを操作するコマンドには毎回 `--page <page-id>` を付ける。
- 複数の実行があるときも、今回のプロファイル ID とページ ID を使い続ける。作成結果が返らなければ、対象 workspace の `tab list --worktree <workspace-id> --show-profile --json` と `tab profile list --json` で作成済みかを照合してから再試行する。既存ユーザーのプロファイルは使わない。
- Orca Delivery からプロファイルとページを割り当てられた場合は、今回の作業専用の ID と URL であることを確認し、そのページを使う。作成・後処理を担当するエージェントを開始前に決める。
- 通信の証拠が必要なケースは、操作前に `ORCA capture start --page <page-id> --json` を行う。認証やモバイル表示の手順は [ブラウザ操作の詳細](references/browser.md) を参照する。

## 4. Orca の snapshot から操作し、結果を観測する

次は架空のプロフィール画面で表示名を更新する例。ref は実際の snapshot からロールと名前が一致するものを取る。操作やリロードで遷移した場合は、その都度 URL を照合してから次のコマンドへ進む。

```text
ORCA snapshot --page <page-id> --interactive --json
ORCA fill --page <page-id> --expect-url <current-url> --element <textbox-ref> --value "QA User" --json
ORCA snapshot --page <page-id> --interactive --json
ORCA click --page <page-id> --expect-url <current-url> --element <save-button-ref> --json
ORCA wait --page <page-id> --text "Saved successfully" --timeout 5000 --json
ORCA snapshot --page <page-id> --json
ORCA screenshot --page <page-id> --json
ORCA reload --page <page-id> --json
ORCA tab show --page <page-id> --json
ORCA wait --page <page-id> --text "Stored name: QA User" --timeout 5000 --json
ORCA screenshot --page <page-id> --json
```

- `--interactive` は操作対象を探すときに使う。トースト、エラー、状態文言の観測には通常の snapshot を使う。JSON の `result.refs` でロールと名前を照合する。たとえば `"e2": {"role": "textbox", "name": "Display name"}` なら `--element @e2` を使う。
- 操作の JSON に出る `result.effects` で遷移・ページ差分・ポップアップ・読み込み状態を確認する。初回のページ読み込み、遷移やリダイレクト、ポップアップ後は実際の URL を確認し、許可オリジンから外れたページでの操作・入力・証拠採取は止める。遷移後、待機後、`browser_stale_ref`、差分省略時は同じページで snapshot を取り直す。コミットや保存のような書き込みでは正しい絶対 URL を `--expect-url` に渡し、画面上の操作対象も確認する。
- 保存操作の成功は、トーストに加えてリロードや一覧・詳細からの再表示で確かめる。タイムアウトや無反応時に、更新が既に行われた可能性を確認してから再試行する。
- 結果が期待と違う場合は、ページ ID、操作した ref、非同期の待ち、権限、実行環境、起動中の版を照合する。Orca 固有のエラーと確認手順は [ブラウザ操作の詳細](references/browser.md) にある。

## 5. 証拠を残し、今回作ったものを閉じる

`screenshot --json` が返した実ファイルを確認し、証拠ディレクトリへケース名を付けて保存する。保存結果の再表示や操作前後が比較できるようにする。画像と snapshot も保存・共有前に確認し、資格情報、個人情報、内部 URL が写る場合は見せてよい範囲の証拠を取り直すか、伏せ字や切り出しを施した証拠に置き換える。公開リポジトリや共有先に元のデータを置かない。通信を確認した場合は `network --page <page-id>`、`console --page <page-id>` から関係するステータス、伏せ字にした経路、エラーの要約を抜粋する。

```text
ORCA tab close --page <page-id> --json
ORCA tab profile delete --profile <profile-id> --json
```

閉じるのは今回作成したタブだけ。プロファイルも同じ実行で作成した ID だけを削除する。別の担当が作成したものは、取り決めた担当に実行記録と ID を渡して後処理を確認する。証拠の保存先と後処理の完了を確認してから報告する。

| 判定 | 条件 |
|---|---|
| PASS | 対象の画面で操作し、期待結果を証拠で確認した |
| FAIL | 前提を満たした状態で期待との差を観測した |
| BLOCKED | Orca、環境、権限、データ等の不足で判定できない |
| SKIP | 依頼の範囲外などの理由で実行しなかった |

対象の版と URL、ケースごとの操作・期待・観測・判定・証拠を示す。必須ケースすべての PASS を総合 PASS の条件にする。失敗の根拠、ユーザーや運用への影響、残したテストデータ、未確認範囲も該当する場合に添える。QA で見つけた修正は別途依頼されたときに行う。
