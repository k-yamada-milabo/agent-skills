# 状態管理と復旧

通常のセッション再開ではこの文書を読まない。`Current State`がない、状態が一致しない、マスタータスクが変わった、またはプロジェクト完了時だけ読む。

## pair-programming-log.mdの構造

`pair-programming-log.md`は個人の作業状態とレビュー証跡であり、通常はリポジトリへコミットしない。先頭に短い`Current State`を置き、その後に追記専用の`Event Log`を置く。

```markdown
# Current State

- Project:
- Master task:
- Detail plan:
- Current step:
- Status: instruction_active | reviewed | recovery_required
- Latest instruction:
- Latest review:
- HEAD:

## Active instruction

## Decisions needed for next work

## Worktree notes

## Unresolved matters

## Next candidate

## Document fingerprints

# Event Log
```

`Current State`は通常再開時の入力を小さくするためのローリングスナップショットである。100行程度を上限の目安にし、過去の説明、一般的な技術解説、完了済みの細かな操作を蓄積しない。履歴は`Event Log`へ残す。

文書fingerprintには少なくとも`overview.md`、`tasks.md`、`context.md`、現在の詳細プランの`git hash-object`を記録する。通常再開時は内容を読む前にhashを比較し、一致すれば現在ステップ以外を読み直さない。

## Baseline Event

新しいマスタータスクを開始するとき、または有効な状態を復元できないときに記録する。

- HEAD
- 開始時の作業ツリー状態
- 開始時から存在する追加・変更ファイルの`git hash-object`。削除は`deleted`
- 各変更が今回の作業範囲内か範囲外か
- 対象の`overview.md`、`tasks.md`、`context.md`、承認済み詳細プランを全文確認したこと

Baselineは差分の起点であり、その時点の変更がレビュー済みであることを意味しない。

## Instruction Event

- 安定したInstruction ID
- 詳細プランのパス、ステップ番号、必要ならサブID
- 目的
- 変更対象
- 参考実装
- 守るべき制約
- 完了条件
- 確認方法
- 明示的な対象外、作成途中として扱うファイル

同じ内容を`Current State`の`Active instruction`にも置く。チャットにはこの記録と同じ指示を出す。

## Decision Event

- 安定したDecision ID
- 対象のマスタータスク、ステップ、Instruction ID
- 決定内容
- 理由
- 影響範囲
- 採用しなかった案。後で再検討される可能性がある場合だけ記録する
- `context.md`または詳細プランへの反映先と反映状態

次のいずれかに当たる合意を記録する。

- 公開API、型、命名を変える
- DB、認可、RLS、監査、エラー契約を変える
- 承認済み計画または対象範囲を変える
- 後で同じ議論を繰り返す可能性が高い
- 次のセッションでレビュー範囲を誤認する可能性がある

後続へ影響しない一般的な説明、単発のコマンド、途中で直したtypoは記録しない。

## Review Event

- 対応するInstruction ID
- 詳細プランのパス、ステップ番号、サブID
- 実装箇所。ファイルパスと、可能なら関数、型、テストなどの安定した識別子
- 今回のイベントで追加、変更、削除、renameされたファイルだけ
- 各対象ファイルのレビュー後の`git hash-object`。削除は`deleted`
- 実行した検証と結果
- passedまたはchanges_requested
- 未確認事項

ファイル状態はBaselineへpassedのReview Eventを順番に適用して復元する。同じファイルが後続で変更された場合は最新hashで置き換える。`pair-programming-log.md`など進捗記録だけの変更は、成果物自体がレビュー対象でない限りhash対象にしない。

## Session Handoff

- 完了または継続中のInstruction ID
- レビュー結果
- 今回確定した判断
- 正式文書へ反映済み・未反映の区別
- 対象外または作成途中の変更
- 未解決事項
- 次の候補。具体的な次指示は新しいセッションで決める

同じ内容を`Current State`へ圧縮して反映する。通常再開ではこの`Current State`だけを読み、過去のEvent Logは読まない。

## 状態不一致からの復旧

1. `docs/projects/README.md`、適用される`AGENTS.md`、対象の`overview.md`、`tasks.md`、`context.md`、承認済み詳細プラン全体を読む。
2. 最新のBaseline以降のInstruction、Decision、passed Review、Session Handoffを読む。
3. `git rev-parse HEAD`、`git status --short`、記録対象の`git hash-object`を照合する。
4. 記録にない変更、hash不一致、想定外の削除・renameは未レビューとして扱う。hashから差分内容を復元できるとは考えない。
5. 正確な増分差分を復元できない場合は、影響するファイルと詳細ステップを保守的に再レビューする。
6. 復元した状態で新しいBaselineと`Current State`を記録する。

作業ツリーがdirtyでも開始を禁止しない。既存変更、今回の対象、対象外の作成途中変更を区別できない場合だけ、理由を確認するまでレビュー済みとして扱わない。

ユーザーが複数のレビュー済み指示をまとめてコミットした場合は、そのコミットを確認し、次のセッションで新しいHEADをBaselineとして記録する。エージェント自身はstageまたはcommitしない。

## 最終レビュー

`tasks.md`の全マスタータスクが完了した後、利用可能なサブエージェント機能で独立したレビュアーを一度だけ起動する。セッション履歴を渡さず、レビュアーへ`code-reviewer`スキルを明示的に使用させる。

次の情報だけを渡す。

- リポジトリルート
- `overview.md`、`tasks.md`、`context.md`、`pair-programming-log.md`、使用した詳細プランのパス
- 利用可能なら作業開始時のbase SHA
- 現在のHEAD、作業ツリー状態、対象ファイル
- 実行済みテストと結果
- 実装中に残した懸念や未確認事項

`Critical`または`Important`の指摘があれば、対応する指示へ戻り、ユーザーによる修正、完了報告、再レビューを行う。`Minor`は記録するが原則として完了を妨げない。ブロッキング指摘が解消され、必要な検証結果が揃った時点で完了を報告する。
