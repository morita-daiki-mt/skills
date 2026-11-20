# Orca での実行

## バージョンと配置を確認する

以降の `ORCA` はこのセッションで使う CLI に置き換える。環境に `ORCA_CLI_COMMAND` の指定があればそれを使い、Orca 開発環境の `ORCA_DEV_REPO_ROOT` が設定されている場合は `orca-dev` を使う。Linux の通常のシェルでは `orca-ide` を使う。Orca 管理下のセッションや macOS ではインストールされた Orca CLI を使い、エラー時に別のランタイムへ切り替えない。

```text
ORCA skills get orca-cli
ORCA skills get orchestration
ORCA skills get orchestration --reference references/coordinator-loop.md
ORCA status --json
```

最初の引き継ぎを別セッションで行うときは、`agent launch --help` と CLI ガイドで、同じ workspace の新規 Claude セッションの作成方法を確認する。今見えているタブから推測せず、確認した workspace のセレクターを使う。利用者が既存の Claude セッションを進行役に指定した場合は、そのセッションの配置と担当を照合する。

CLI で進行役を開く例。引き継ぎ文には「このセッションが進行役である」「対象の仕様確認後、許可された作業だけを監督する」と記載し、実ファイルを指す。起動時に引き継ぎ ID、対象 workspace と checkout、タスク文書のパスを渡す。既存の引き継ぎがあれば記録されたセッションを照合する。起動結果が不明な場合は既存セッションの有無を確認してから復旧する。

```text
ORCA agent launch --agent claude --model claude-opus-5-5 --worktree <workspace-selector> --surface terminal --prompt "<引き継ぎ文>" --json
```

## 引き継ぎ確認と仕様承認

1. Claude は編集に入る前に、引き継ぎ ID・自身のセッション・workspace・checkout・読んだ規約・許可範囲を返す。引き継ぎ元は起動結果と照合し、不一致は解消してから引き継ぎを確定する。
2. Claude は変更範囲、受け入れ条件、必須チェック、QA ケースを含む仕様書を保存する。仕様書の SHA-256 と仕様の要点を示し、その版をユーザーが明示承認するまで待つ。明示的な開始操作に `/goal` を使う環境では、対象のハッシュを含む開始条件をユーザーに提示して、その操作を待つ。
3. 承認されたハッシュを実行記録に残す。起動前と再開時に仕様書を照合する。対象、許可範囲、受け入れ条件を変えた場合は新しい版の承認を得てから進める。

## checkout のロック

承認後、初回の Run や編集担当を起動する前に、対象 checkout ごとに排他ロックを取得する。対象ホスト上で `git -C <checkout> rev-parse --absolute-git-dir` を使い、そのディレクトリ内の固定名 `orca-delivery.lock` を `mkdir` で作成する。原子的にディレクトリを作成できた場合だけ取得成功とする。既存ディレクトリの有無を調べてから別の操作で所有を宣言する方式は競合を防げない。

ロックには引き継ぎ ID、進行役のセッション、対象 checkout を記録し、Run 作成後に Run ID を追記する。すでにロックがあれば所有者と Orca の実行状態を確認し、同じ Run の再開と証明できるまで新しい書き手を起動しない。所有者不明、期限切れに見える場合、ロック取得に失敗した場合も実装は保留する。複数 checkout のうち一つでも取得できなかった場合は、今回取得済みで担当を起動していないロックを返却する。取得済みロックは関連する担当がすべて決着した後に所有者が解放する。これは同じ checkout の Delivery 同士を調整する仕組みで、外部編集の検知は次の候補照合で行う。

## 進行役

以下は承認とロック取得を済ませたローカルの同一 workspace で実行する例。各 Task ファイルは絶対パスにし、担当、対象の版、編集の許可、期待する証拠を記載する。以下のモデル指定はこのスキルの既定値。指定がある場合はコマンドと起動設定を照合する。

```text
ORCA orchestration run-create --objective "<承認された目的>" --json
ORCA orchestration worker-start --run <run-id> --worktree current --agent codex --model gpt-6.2-sol --spec-file <実装タスク> --json
ORCA orchestration worker-start --run <run-id> --worktree current --agent claude --model claude-opus-5-5 --spec-file <レビュータスク> --json
ORCA orchestration worker-start --run <run-id> --worktree current --agent codex --model gpt-6.2-sol --spec-file <レビュータスク> --json
ORCA orchestration worker-start --run <run-id> --worktree current --agent codex --model gpt-6.1-terra --effort max --spec-file <QAタスク> --json
```

まず Run を作り、実装だけを起動する。実装結果と必須チェックを確認してから独立レビューの2件を起動する。QA は両方のレビューが決着し、必須チェック失敗と重大な指摘が解消した候補に対して起動する。受領情報の実行設定と placement を確認する。ソース編集の権限を役割に合わせられない場合は、Orca の low-level topology ガイドを読み、適切な権限で起動した専用端末を worker に結び付ける。端末起動時の `--model` と `--effort` は `worker-start --terminal` に重ねない。別の checkout が必要なときは、プロジェクト規約と版の同期方法を確認し、Orca の配置ガイドに従う。共有 DB やビルドディレクトリを使う検証は、相互に干渉しないよう調整する。

質問・完了報告は Orca の wake に従って処理する。通信は Dispatch 宛ての `send` や質問への `reply` を使う。担当は実行時の preamble に従って結果を返す。起動失敗時は受領情報の `failedStage` と残存リソースを確認し、`references/recovery-and-cleanup.md` を取得する。停止時は、停止の依頼と実際に作業が終わったことを区別する。今回作った端末だけを対象に後処理する。

## 候補の受け渡しと中断

各 Task は対象 workspace、候補の版、許可パス、適用規約、受け入れ条件、結果の返し方を含める。実装担当には変更ファイルとチェックの終了結果、レビュー担当には再現条件と根拠、QA 担当にはケース別の証拠を求める。QA Task には対象 URL、起動中の版、許可操作、テストデータの前提を付ける。ソース読み取り専用の担当が報告を書けない場合は会話に出させ、進行役が回収する。

進行役は、承認済み仕様の版・候補・Run ID・Task / Dispatch ID・役割・実際のモデル・期限・完了状態・証拠を実行記録に残す。記録の保存先はユーザー指定かリポジトリ外の作業用ディレクトリを使う。

1. 各段階の前後で HEAD、ブランチ、index、追跡ファイルの差分、非無視の未追跡ファイルとその内容、仕様・適用規約の版を比較する。例として `git status --porcelain=v1 -z --untracked-files=all`、`git ls-files --stage -z`、`git diff --binary --no-ext-diff --no-textconv` および対象未追跡パスの SHA-256 を組み合わせる。記録はパスとハッシュに絞り、ファイル内容や機密情報は写さない。
2. 実装担当の決着後は、変更パスを承認済みの範囲と照合する。許可されていない HEAD・ブランチ・index・規約・ソースの変更が見つかったら、差分を保全し、出所と影響が分かるまで次へ進まない。レビュー・QA の実行中に候補が変わった場合も結果を保留する。
3. 実装または修正後の版を新しい候補として記録する。必須チェックは対象候補で実行し、コマンド、終了コードと結果を残す。独立レビューには同じ仕様と候補を渡す。QA は動いているアプリがその候補だと確認してから開始する。前の候補の結果を新しい候補の実績として扱わない。

既定は4候補、累積の自動作業90分、同時 worker 2人、停止時の整理猶予120秒。人の判断待ちは自動作業時間から外し、再開時は使用済みの予算を引き継ぐ。修正を始める前に次の候補の枠を確認する。最後の候補で重大な指摘・必須チェック失敗・必須 QA の FAIL が残ったら、検証の枠がない修正は作らず、未解決事項を報告する。上限に達したら新規 Dispatch を止め、稼働中の担当に協調停止を依頼する。停止が確認できない間は別の書き手を同じ対象に起動せず、最終報告には未決着の Dispatch と作業ツリーの状態を示す。

終了報告には最新候補、必須チェック、独立レビューと QA の証拠、未解決事項を含める。状態は成功、判断待ち、上限到達、環境・権限などの理由で実行不能のいずれかを明示する。
