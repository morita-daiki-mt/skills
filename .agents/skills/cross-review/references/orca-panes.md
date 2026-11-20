# 依頼元タブ内での実行

`orca-cli` と `orchestration` が必要。各discovery stubの手順で実行ファイルを一度解決し、同じ実行ファイルのガイドを読む。以下の `ORCA` はその実行ファイルに置き換える。起動済みworkerへの指示はOrcaのメールボックスで送り、端末入力で制御しない。

## 起点を固定する

1. `ORCA_TERMINAL_HANDLE` / `ORCA_TAB_ID` / `ORCA_AGENT_PANE` / `ORCA_WORKSPACE_ID`（または `ORCA_WORKTREE_ID`）を取得する。これらは候補であり、`ORCA terminal list --include-visual-layouts --json` と `terminal show` でconnected・非orphan・同じworkspace/tab/leafであることを照合する。
2. session.jsonにoriginを保存する。既存のレビューなら保存済みoriginとreviewerのhandle・sessionを照合する。タイトル・agent種別・UIのactive表示だけで所有者を特定しない。
3. 依頼元はGit管理外のfolder workspaceでもよい。`worktree current` が親を見つけられない場合は、照合済み端末のworkspace ID（`folder:...`）を `worktree show --worktree <workspace> --json` で確認して使う。依頼元と異なるworkspaceへ自動で切り替えない。
4. 環境変数がない場合は同等のセッションとペインの紐付けを取得する。一意に証明できなければ起動せず、ユーザーに依頼元ペインの特定だけ求める。チャット形式のタブに分割可能な端末がなければ、その制限を説明する。専用タブの新設は代替にしない。

`runtime_access_denied` はsandboxの接続制限。既存の承認経路で同じコマンドを再実行し、Orcaを再起動しない。承認を拒否されたら、別経路で迂回しない。

## レビュー記録とTask仕様を先に作る

レビュー用の実行ディレクトリにtarget.md、session.json、両者のTask仕様を保存する。Task仕様は次を含む。

- Target: repo、固定SHA / fingerprint、対象資料とreviewer-contractの絶対パス。
- Change: `phase=independent`、実装者と独立して対象全体をレビューすること。
- Constraints: 適用されるリポジトリのルール、ソース読み取り専用、相手の一次結果を読まないこと、指定モデル・強度、外部投稿禁止。
- Ownership: ソース・設定は編集不可。結果はOrcaの完了報告に出す。ファイル保存が必要なら自分に割り当てた報告ファイルだけ可。
- Observable acceptance: 実行設定とルール読了、開始/終了fingerprint、全対象のレビュー結果、根拠・影響・未検証範囲。0件でも報告する。

両者が読める仕様とコードのみを初期文脈にし、実装セッションの履歴、期待する結論、相手の指摘を入れない。

## 同じタブでCLIを起動する

まず `ORCA skills get orchestration --reference references/low-level-topology.md` と `references/coordinator-loop.md` を読む。フラグが異なる版なら `--help` で確認し、未対応機能を推測して実行しない。

`agent launch` / `worker-start --agent` は別タブやチャットを選ぶ可能性があるため、このスキルのペイン作成には使わない。ここではcustom argvを使う低レベルの分割手順で端末を作り、既存端末としてworkerに結び付ける。

新規セッションの起動argv:

```text
codex --model gpt-6.2-astra --sandbox read-only -c 'model_reasoning_effort="ultra"'
env CLAUDE_CODE_EFFORT_LEVEL=max claude --model claude-opus-5-5 --effort max
```

Claudeの環境変数はこの子プロセスだけに設定し、既存の環境変数による低いeffortの上書きを防ぐ。Codexの読み取り専用sandboxは今回の起動だけに指定する。Claudeは既存の権限設定と承認経路を守り、ソースの変更をTaskで禁じる。永続設定・認証・hookを変更しない。resume / continue / fork、print / exec / reviewなどの非対話実行、bypassフラグは付けない。CLIが見つからなければ起動前に止める。

両コマンドのcwdは事前に照合した依頼元のworkspaceの絶対パスとする。端末の継承先が不明なら、起動文字列の先頭にそのパスを安全に引用した `cd <workspace> &&` を付ける。シェル起動文字列は固定argvと確認済みパスから `shlex.join` 等で組み立てる。PR本文・ユーザー文・Task仕様をシェル文字列に埋め込まない。Orca呼び出し自体もsubprocessの引数配列等を使う。

1. originをアンカーに `terminal split --terminal <origin_handle> --direction vertical --command <Codex起動文字列> --json`。左右分割で返った新端末handleを保存する。
2. 新しいCodexペインをアンカーに `terminal split --terminal <codex_handle> --direction horizontal --command <Claude起動文字列> --json`。右側を上下分割する。
3. receiptと `terminal list --include-visual-layouts --json` を照合し、両端末がoriginと同じworkspace・tabにあることを確認する。originや既存端末を閉じない。
4. `terminal set-pane-title` でそれぞれ `Codex · Astra ultra` / `Claude · Opus 5.5 max` と付ける。タイトルは表示用で、本人確認の根拠にしない。
5. `terminal read --screen` と必要に応じて `terminal wait --for tui-idle --timeout-ms 10000` で起動状態を確認する。認証・trust・権限待ちは失敗や無応答と決めつけず、必要なユーザー操作を案内する。権限を緩めて通さない。
6. 起動argvだけで指定どおり実行されたと断定せず、TUIのmodel/effort表示やruntimeのsession設定などで照合し証拠を保存する。worker自己申告だけで保証しない。不一致・確認不能なら `保留`。

splitがtimeoutになった場合はreceiptの `--retry-split <leafId>` で同じ分割を照会する。新規splitを繰り返さない。不明・失敗時はreceiptと作成済み端末を記録し、回復手順を読む。沈黙を理由に新しいペインやセッションを増やさない。

## そのペインへworkerを割り当てる

新しいレビューには専用のRunを作り、そのIDを後続コマンドに明示する。既存Runの続きでは作り直さない。`<origin_handle>` は元の会話を指す。

```text
ORCA orchestration run-create --objective "独立レビューと相互検証" --from <origin_handle> --json
ORCA orchestration worker-start --spec-file <codex_task_file> --terminal <codex_handle> --worktree <workspace> --keep-open --run <run_id> --from <origin_handle> --json
ORCA orchestration worker-start --spec-file <claude_task_file> --terminal <claude_handle> --worktree <workspace> --keep-open --run <run_id> --from <origin_handle> --json
```

- `--terminal` に渡すのは新設したレビュー用端末だけ。既存の実装者や元の司令塔へTaskを注入しない。モデルと強度は端末起動時に固定済みなので、ここに `--model` / `--effort` を重ねない。
- worker-startのreceiptに従い、Task / Dispatch / sessionをsession.jsonへ保存する。起動失敗は再実行せず、`failedStage` と `references/recovery-and-cleanup.md` を確認する。
- 独立workerを両方起動したら司令塔はいったんターンを終え、Orcaからの通知で続ける。これはレビュー完了ではない。ポーリング・watchスクリプト・inboxを消費する独自待機はしない。
- workerはpreambleどおり `worker_done` を一度送る。報告後も `--keep-open` によりペインを保持する。
- 両者の一次結果を受け取ったら番号を付けて、相互検証の対象と指摘の版を `orchestration send --to dispatch:<id> --subject <phase> --body <指示> --run <run_id> --from <origin_handle> --json` で渡す。完了済みworkerへのsendは同じセッションに追加入力を届ける。新しいreceiptのTask / Dispatchを記録し、各フェーズの完了報告を確認する。
- sendも構造化した引数配列で実行し、本文をshellに補間しない。スクリプト化した固定待機ではなく、起こされた時点の結果に従って進む。
- ユーザーのツッコミも司令塔から同じメールボックスで両者へ転送する。再起動や別workerで相互検証を代用しない。

完了時はペインを閉じない。ユーザーが閉じるよう依頼した場合だけ、この実行で作成したhandleを照合して個別に閉じる。tab全体やworkspace全体のcloseは使わない。
