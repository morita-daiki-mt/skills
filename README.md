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
