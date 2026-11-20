# エージェント用の開発スキル

タスクの実装、独立レビュー、ブラウザ QA など、繰り返し使う開発手順をまとめたスキル集です。ブランチ、チェックコマンド、権限、環境は使用先のリポジトリの規約に従います。

汎用的にしたものなので、使う環境にあわせて調整すること


| スキル                                                                  | 使う場面                                     |
| -------------------------------------------------------------------- | ---------------------------------------- |
| [orca-delivery](.agents/skills/orca-delivery/SKILL.md)                       | Claude と Codex で仕様確認から実装・独立レビュー・QA まで進める |
| [orca-qa](.agents/skills/orca-qa/SKILL.md)                                   | Orca のブラウザを操作し、結果を証拠付きで報告する              |
| [cross-review](.agents/skills/cross-review/SKILL.md)                         | Claude と Codex に同じ差分を独立レビューさせ、指摘を相互検証する  |
| [implement-todo](.agents/skills/implement-todo/SKILL.md)                     | Markdown のタスク文書を起点に実装し、検証を経て進捗を更新する      |
| [review-feedback-learning](.agents/skills/review-feedback-learning/SKILL.md) | 人間のレビュー指摘から再発防止に使える観点を抽出する               |
| [rspec-quality](.agents/skills/rspec-quality/SKILL.md)                       | RSpec の検証内容と検出力を確認する                     |
| [quizzing](.agents/skills/quizzing/SKILL.md)                                 | 実装や設計を自分の言葉で説明できるか確かめる                   |

## Orca を使うスキル

Orca IDE を起動し、対象リポジトリまたはフォルダを Orca の workspace として開きます。Codex から Orca CLI が使える環境で、対象の checkout と workspace を照合して実行します。[Orca CLI のセットアップ](https://orca.dev/docs/cli/overview)

- `$orca-qa` は組み込みブラウザで操作します。テスト用の URL と操作・期待結果を渡すと、専用プロファイルでケースを走らせて証拠を残します。
- `$orca-delivery` はタスク文書を受け取り、Orca Orchestration で Claude と Codex の進行・実装・レビュー・QA を管理します。
- `$cross-review` は対象の差分を受け取り、Orca の依頼元タブに Codex と Claude のペインを開いて相互レビューします。

```text
$orca-qa http://127.0.0.1:3000/settings で表示名を保存し、リロード後の表示を確認して
```
