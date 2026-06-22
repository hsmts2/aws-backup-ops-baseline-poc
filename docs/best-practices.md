# ベストプラクティス

## 概要

AWS Backup の運用では、バックアップを取得するだけでなく、対象管理、復旧確認、削除耐性、通知、監査の観点を整理することが重要です。

このドキュメントでは、本リポジトリで採用した設計と、今後の運用で検討できるポイントを整理します。

<br>

## タグで対象を管理する

このテンプレートでは、`Backup=true` が付与されたリソースをバックアップ対象にします。

バックアップ対象外にしたいリソースにはタグを付与しません。

タグベースにすることで、CloudFormation テンプレートを変更せずにバックアップ対象を追加できます。

<br>

## EventBridge で失敗系イベントを通知する

AWS Backup は EventBridge にバックアップジョブやコピージョブ、復元ジョブなどの状態変化イベントを送信します。

AWS Backup の通知APIでもSNS通知は可能ですが、EventBridge は Backup Vault、Copy Job、Restore Job など、より広いイベントを扱えるため、このテンプレートでは EventBridge を基本にしています。

通常運用では成功通知をすべて送ると通知量が増えやすいため、まずは `FAILED`、`EXPIRED`、`ABORTED` を通知対象にします。

<br>

## Backup Vault は削除保護を意識する

このテンプレートでは、Backup Vault に `DeletionPolicy: Retain` を設定しています。

CloudFormation スタックを削除しても復旧ポイントを誤って削除しないためです。

スタック削除前には、復旧ポイントの保持期間、Vault Lock、手動削除可否を確認します。

<br>

## Vault Lock は段階的に利用する

Vault Lock は、復旧ポイントの削除耐性を高めるための機能です。

Compliance mode は猶予期間経過後にロック解除できなくなるため、最初から本番適用するのではなく、検証環境で保持期間と削除手順を確認します。

このテンプレートでは、誤ロックを避けるため `VaultLockMode` のデフォルトを `none` にしています。

<br>

## 復旧テストを定期的に行う

バックアップジョブが成功していても、復旧できるとは限りません。

定期的に復旧テストを行い、以下を確認します。

| 観点 | 確認内容 |
| --- | --- |
| 復旧ポイント | 想定した日時の復旧ポイントが存在するか |
| 復旧先 | 既存環境に影響しない復旧先を選べるか |
| 接続性 | 復旧後のEC2、EBS、RDSに接続できるか |
| アプリケーション | サービス起動やデータ参照ができるか |
| 後片付け | 復旧テスト用リソースを削除したか |

<br>

## 監査と拡張

より厳密な運用では、以下も検討します。

| 項目 | 内容 |
| --- | --- |
| AWS Backup Audit Manager | バックアップ計画、保持期間、Vault Lock などの準拠状況を確認 |
| Cross-Region copy | リージョン障害に備えて別リージョンへ復旧ポイントをコピー |
| Cross-account copy | アカウント侵害や誤操作に備えて別アカウントへ復旧ポイントをコピー |
| AWS Organizations backup policy | 複数アカウントに共通のバックアップポリシーを適用 |
| Restore testing | 復旧テストの定期実行と記録 |

このリポジトリでは、まず単一アカウント・単一リージョンの最小構成を作成し、通知と復旧確認まで説明できる構成にしています。

<br>

## 参考

- [Monitoring AWS Backup events using Amazon EventBridge](https://docs.aws.amazon.com/aws-backup/latest/devguide/eventbridge.html)
- [AWS Backup Vault Lock](https://docs.aws.amazon.com/aws-backup/latest/devguide/vault-lock.html)
- [AWS Backup Audit Manager](https://docs.aws.amazon.com/aws-backup/latest/devguide/aws-backup-audit-manager.html)
